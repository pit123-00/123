# Дедупликация на лету

В репозитории `LetterFeed` механизм дедупликации электронных писем реализован "на лету" перед тяжеловесными операциями парсинга и обработки контента.

## Оригинальная логика в LetterFeed

В файле `backend/app/services/email_processor.py` в функции `_process_single_email` проверка на дубликаты выполняется сразу после извлечения минимальных метаданных (Отправитель и Message-ID):

```python
def _process_single_email(
    num: str,
    mail: imaplib.IMAP4_SSL,
    db: Session,
    sender_map: dict[str, Newsletter],
    settings: Settings,
) -> None:
    # 1. Извлекается только заголовок (PEEK)
    status, data = mail.fetch(num, "(BODY.PEEK[])")
    msg = email.message_from_bytes(data[0][1])

    # 2. Получение уникального идентификатора
    message_id = msg.get("Message-ID")

    if not message_id:
        logger.warning("Email has no Message-ID, skipping.")
        return

    # 3. Дедупликация: проверка наличия записи в БД ДО парсинга контента
    if get_entry_by_message_id(db, message_id):
        logger.info(f"Email with Message-ID {message_id} already processed, skipping.")
        return

    # ... дальнейший тяжелый парсинг HTML (readability/nh3), создание Entry и т.д.
```

Идея очень проста и эффективна: нет смысла запускать парсеры, чистильщики или обращаться к LLM, если сообщение уже было обработано.

## Интеграция в PBD (Универсальный Атом & Fingerprint)

Для `Personal Browser Dashboard`, архитектура которого предполагает поглощение из множества источников (RSS, Email, Telegram, и т.д.), полагаться только на `Message-ID` (который есть только в Email) невозможно.

Поэтому, адаптируя эту идею, мы используем **Fingerprint (UUIDv7)**:

1. **Единый алгоритм хеширования**:
   Независимо от того, пришел ли пост из Telegram или RSS, Ingestion Layer (сборщик) должен сгенерировать детерминированный хэш на основе стабильных метаданных.
   Например: `hash(url + date + title)`.

2. **UUIDv7 Fingerprint**:
   Полученный хэш конвертируется в UUIDv7. Преимущество UUIDv7 заключается в том, что он лексикографически сортируется по времени, что улучшает производительность индексов в PostgreSQL.

3. **Проверка на лету (Ранний возврат)**:
   При получении новой порции данных (например, распарсенного `JSON Feed`), сборщик вычисляет UUIDv7 Fingerprint для каждого `Item`.
   Система выполняет запрос в БД (или проверяет Redis-кэш): `SELECT EXISTS(SELECT 1 FROM atoms WHERE id = $1)`.

4. **Экономия ресурсов**:
   Только если Атома с таким UUIDv7 еще нет в БД, система:
   - Сохраняет `L1_raw` запись.
   - Запускает `NormalizerService` (очистка HTML).
   - Ставит отложенную задачу (Celery/Hatchet) на запуск `ReasoningService` (LLM) и `EmbedderService`.

Таким образом, мы берем идею проверки "до парсинга" из LetterFeed, но делаем ключ дедупликации универсальным для всех типов контента.
