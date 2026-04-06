# Инъекция кастомных полей (расширений) в jsonfeed

Задача: Позволяет ли библиотека `jsonfeed` инжектить скрытые кастомные поля (например `_sync_cursor` из логики 15-routing) в финальный JSON до сохранения?

## Анализ библиотеки jsonfeed

Посмотрим на метод `_to_ordered_dict` класса `Item` из библиотеки `jsonfeed`:

```python
    def _to_ordered_dict(self) -> dict:
        ordered = {"id": self.id}
        if self.url:
            ordered["url"] = self.url
        # ... жестко закодированный список полей (title, content_html, etc)
        if self.attachments:
            ordered["attachments"] = [a._to_ordered_dict() for a in self.attachments]
        return ordered
```

Спецификация JSON Feed поддерживает расширения (custom extensions). Обычно они начинаются с символа подчеркивания, например `_sync_cursor`.

Однако библиотека **`lukasschwab/jsonfeed` вообще не поддерживает кастомные поля**.
1. При парсинге (`Item.parse()`) она игнорирует любые поля, которых нет в ее жестко закодированном списке.
2. При сериализации (`to_json() -> _to_ordered_dict()`) она выводит только те поля, которые определены в конструкторе класса `Item`. Вы не можете добавить `item._sync_cursor = "123"` и ожидать, что оно появится в JSON-выводе.

## Решение для PBD (Инъекция кастомных метаданных)

Использовать `lukasschwab/jsonfeed` как единственную модель данных в PBD нельзя, так как ваша архитектура (`Awareness System`) требует хранения обширных метаданных (скоринг F-G-R, теги K1/K2, курсоры).

**Вариант 1: Отказаться от `jsonfeed` в пользу Pydantic (Рекомендуемый)**

Поскольку вы используете FastAPI и PostgreSQL (`JSONB`), гораздо правильнее написать свою собственную Pydantic модель, которая строго соответствует стандарту JSON Feed 1.1, но позволяет добавлять кастомные поля (Extra.allow).

```python
from pydantic import BaseModel, Field

class JSONFeedItem(BaseModel):
    id: str
    url: str | None = None
    title: str | None = None
    content_html: str | None = None
    summary: str | None = None
    date_published: str | None = None

    # Кастомные поля PBD (соответствуют стандарту JSON Feed, начинаясь с `_`)
    _sync_cursor: str | None = Field(default=None, alias="_sync_cursor")
    _pbd_fgr_score: dict | None = Field(default=None, alias="_pbd_fgr_score")

    class Config:
        # Позволяет парсить и отдавать поля с подчеркиванием
        populate_by_name = True
        # Если вы хотите разрешить любые другие кастомные поля
        extra = "allow"
```

Эта модель отлично ляжет в столбец PostgreSQL `JSONB` через `SQLModel`, и вы сможете использовать индексы (GIN) для поиска по этим кастомным полям.

**Вариант 2: monkey-patching или обертка (Не рекомендуется)**

Если вы обязаны использовать `jsonfeed`, вам придется оборачивать его сериализацию:

```python
feed_item = jsonfeed.Item(id="123", title="Test")
item_dict = feed_item._to_ordered_dict()
# Инжектим кастомное поле руками
item_dict["_sync_cursor"] = "cursor_123"
final_json = json.dumps(item_dict)
```
Это костыль, который усложнит типизацию в вашем проекте. Подход с `Pydantic` на 100% соответствует вашему стеку (FastAPI + SQLModel).