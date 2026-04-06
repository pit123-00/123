# Логика FeedState (курсоры скачанных данных) и перенос в `sync_state`

В проекте `feeds.fun` управление состоянием фидов реализовано через Enum-классы `FeedState` и `FeedError` (в файле `ffun/feeds/entities.py`).

## Как реализовано в feeds.fun:
- `FeedState`: описывает общие состояния (`not_loaded`, `loaded`, `damaged`, `orphaned`).
- `FeedError`: детализированный `IntEnum` ошибок (например, `network_connection_timeout`, `parsing_format_error`, `protocol_no_entries_in_feed`).
- В сущности `Feed` (Pydantic-подобная `BaseEntity`) хранятся поля:
  - `state` (FeedState)
  - `last_error` (FeedError)
  - `load_attempted_at` (datetime)
  - `loaded_at` (datetime)

Система использует эти поля как метаданные для шедулера: если `state == damaged`, система может увеличить интервал опроса; если `state == not_loaded`, приоритизировать скачивание. Поля дат используются для вычисления того, когда нужно опрашивать фид снова.

В LetterFeed логика "курсора" для IMAP неявная: она опирается на флаг `(UNSEEN)` на почтовом сервере (состояние хранится на стороне IMAP сервера, а не в локальной БД). После скачивания письмо помечается как прочитанное (`mail.store(num, "+FLAGS", "\\Seen")`) или перемещается в другую папку.

## Перенос паттерна на дизайн таблицы `sync_state` в PBD:

Для PBD (где важна строгая идемпотентность и поддержка разных источников) паттерн `FeedState` из feeds.fun можно и нужно расширить до полноценной таблицы `sync_state`.

Вместо того чтобы полагаться на флаги внешних систем (как в IMAP), вы должны хранить "курсор считывания" в своей БД.

### Рекомендуемая структура таблицы `sync_state` (SQLModel):

```python
class SyncState(SQLModel, table=True):
    id: uuid.UUID = Field(default_factory=uuid7, primary_key=True)
    source_id: str = Field(index=True) # ID источника (URL RSS, папка IMAP, канал Telegram)

    # Enum аналогичный FeedState из feeds.fun
    status: str = Field(default="idle") # idle, syncing, error, suspended

    # Детализированные ошибки (как FeedError)
    last_error_code: int | None = None
    last_error_msg: str | None = None

    # КУРСОР: Главное отличие. Полиморфное поле для разных типов источников.
    # Для RSS: ETag или Last-Modified из HTTP заголовков.
    # Для Telegram/Twitter: ID последнего скачанного сообщения.
    # Для IMAP: UID последнего обработанного письма (если сервер поддерживает UID) или дата.
    cursor_value: str | None = None

    last_sync_attempt: datetime.datetime | None = None
    last_sync_success: datetime.datetime | None = None
```

### Как это работает в Ingestion Layer:
1. При запуске задачи Celery для `source_id = 'my-rss'`, воркер читает `sync_state`, получает `cursor_value` (например, HTTP ETag "W/12345").
2. Fetcher делает запрос с учетом курсора (`If-None-Match: "W/12345"`).
3. Если данных нет (304 Not Modified), обновляется только `last_sync_attempt`.
4. Если есть новые данные:
   - Данные сохраняются (L1_raw).
   - `SyncState` обновляется новым курсором (например, последним Message-ID или новым ETag) и `status` ставится в `idle`.
5. Если ошибка (таймаут): `status` -> `error`, `last_error_code` заполняется (по аналогии с `FeedError.network_connection_timeout` из feeds.fun).

**Вывод:** Паттерн `FeedState` отлично перекладывается на `sync_state`. Главное улучшение для PBD — добавить поле `cursor_value` для обеспечения идемпотентности, чтобы не зависеть от состояния во внешних системах (например, если письмо в IMAP кто-то случайно пометит как непрочитанное, PBD не скачает его дважды, так как `cursor_value` уже запомнил его UID).