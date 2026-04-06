# Отделение парсинга от сохранения (LetterFeed для IngestionService)

В репозитории LetterFeed логика парсинга и сохранения сильно связана в функции `_process_single_email` (файл `backend/app/services/email_processor.py`).
Там происходит сразу и скачивание (`mail.fetch`), и извлечение данных (`email.message_from_bytes`, `_get_email_body`, `_extract_and_clean_html`), и сохранение в базу (SQLAlchemy `create_entry`).

Чтобы правильно отделить этап парсинга от сохранения и внедрить его как независимый Fetcher в слой `Ingestion Layer (Fetchers & Emitters)` проекта PBD, нужно применить паттерн **Adapter/Repository**.

## Рекомендуемый подход для PBD:

1. **Создать абстрактный интерфейс Fetcher-а (IMAPFetcher):**
   Его задача — только подключаться, забирать новые письма (`_connect_to_imap`, `_fetch_unread_email_ids`), извлекать сырой HTML (`_get_email_body`), и возвращать данные в памяти (например, как список Pydantic-моделей или словарей), **не зная ничего о базе данных**.

2. **Извлечение и нормализация (Парсинг):**
   Логику из `_extract_and_clean_html` (использование `readability` (через `nh3` и `BeautifulSoup`) для очистки HTML) вынести в отдельный `NormalizerService` (L2_normalized слой в вашей архитектуре). Fetcher отдает сырой HTML (или очищенный, если вы решите совместить), но не сохраняет его.

3. **Слой Emitters (Сохранение в БД):**
   Полученный от Fetcher список структур конвертируется в строгий `JSON Feed 1.1`.
   После этого отдельный модуль (Emitter) берет этот JSON, генерирует детерминированный UUIDv7 хеш (url+date+title) для дедупликации, и делает `SQLModel` upsert в таблицу `L1_raw`.

### Пример архитектурного разделения (Псевдокод):

```python
# 1. Fetcher (IMAP -> Raw Data)
class IMAPFetcher:
    def fetch_new_emails(self, cursor_state) -> list[RawEmailData]:
        mail_client = self._connect()
        # Поиск только новых (unseen или по дате)
        unread_ids = self._get_unread_ids(mail_client)
        emails = []
        for eid in unread_ids:
            raw_msg = self._fetch_message(eid)
            # Только извлечение полей, без БД
            emails.append(RawEmailData(
                subject=raw_msg["Subject"],
                body=self._get_email_body(raw_msg),
                message_id=raw_msg["Message-ID"],
                date=raw_msg["Date"]
            ))
        return emails

# 2. Pipeline (Оркестрация)
def process_imap_source(source_config):
    fetcher = IMAPFetcher(source_config)
    raw_emails = fetcher.fetch_new_emails()

    # Преобразование в единый стандарт (JSON Feed 1.1)
    universal_atoms = [convert_to_json_feed(email) for email in raw_emails]

    # 3. Сохранение (Emitter)
    for atom in universal_atoms:
        atom.uuid = generate_uuidv7(atom.url, atom.date, atom.title)
        db_repository.upsert_raw_atom(atom)  # SQLModel -> PostgreSQL (L1_raw)
```

Такой подход гарантирует, что если база отвалится, Fetcher не упадет на середине, а если добавится новый источник (Telegram), логика сохранения (Emitter) останется неизменной.