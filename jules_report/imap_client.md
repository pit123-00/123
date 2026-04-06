# IMAP Client & MIME Parser (repo: LetterFeed)

В репозитории `LetterFeed` (`backend/app/services/email_processor.py`) реализован надежный механизм работы с протоколом IMAP и парсинга формата MIME. Этот код можно использовать как готовый `Fetcher` для получения Email-рассылок со 100% совместимостью в Ingestion Layer системы PBD.

## IMAP Client

Логика подключения и выборки сообщений использует встроенную библиотеку `imaplib`.

### 1. Подключение к серверу
Функция `_connect_to_imap` устанавливает SSL-соединение, авторизуется и выбирает нужную папку (например, `INBOX`).

```python
import imaplib

def _connect_to_imap(settings: Settings, search_folder: str) -> imaplib.IMAP4_SSL | None:
    try:
        mail = imaplib.IMAP4_SSL(settings.imap_server)
        mail.login(settings.imap_username, settings.imap_password)
        status, messages = mail.select(search_folder)
        if status != "OK":
            mail.logout()
            return None
        return mail
    except Exception as e:
        return None
```

### 2. Поиск непрочитанных писем
Используется команда `search` с флагом `(UNSEEN)`.

```python
def _fetch_unread_email_ids(mail: imaplib.IMAP4_SSL) -> list[str]:
    status, messages = mail.search(None, "(UNSEEN)")
    if status != "OK":
        return []
    return messages[0].split()
```

### 3. Загрузка сообщения
Загружается только сырое тело письма: `mail.fetch(num, "(BODY.PEEK[])")`. Флаг `PEEK` важен, так как он не помечает письмо как прочитанное автоматически. Извлечение объекта `Message` происходит через `email.message_from_bytes(data[0][1])`.

## MIME Parser

Электронные письма могут содержать текст, HTML, вложения и быть вложенными друг в друга (multipart). Функция `_get_email_body` рекурсивно обходит (walk) структуру MIME-сообщения и извлекает нужный контент, отдавая приоритет HTML.

```python
from email.message import Message

def _get_email_body(msg: Message) -> str:
    html_body = ""
    text_body = ""

    # Обход всех частей письма
    for part in msg.walk():
        ctype = part.get_content_type()
        cdispo = str(part.get("Content-Disposition"))

        # Пропускаем вложения
        if "attachment" in cdispo:
            continue

        if ctype == "text/html":
            try:
                payload = part.get_payload(decode=True)
                charset = part.get_content_charset() or "utf-8"
                html_body = payload.decode(charset, "ignore")
            except Exception:
                pass
        elif ctype == "text/plain":
            try:
                payload = part.get_payload(decode=True)
                charset = part.get_content_charset() or "utf-8"
                text_body = payload.decode(charset, "ignore")
            except Exception:
                pass

    # Приоритет HTML, фолбэк на обычный текст
    return html_body or text_body
```

## Как применить в PBD (Ingestion Layer)

1. **Создание EmailFetcher**: Обернуть логику `_connect_to_imap`, `_fetch_unread_email_ids` и загрузки писем в асинхронный класс `EmailFetcher` или микросервисную задачу Celery/Hatchet.
2. **Парсинг в JSON Feed**: После извлечения данных (тема, отправитель, дата, тело письма `_get_email_body`), эти данные маппятся на Pydantic модель `Item` (из `jsonfeed-util`), чтобы нормализовать формат:
   - `id`: сгенерированный UUIDv7
   - `title`: `msg["Subject"]`
   - `authors`: `msg["From"]`
   - `date_published`: `msg["Date"]`
   - `content_html`: `_get_email_body(msg)`
3. После формирования объекта `Item`, он передается в следующий этап пайплайна — `NormalizerService`, где будет обработан с помощью `readability` и `nh3`.
