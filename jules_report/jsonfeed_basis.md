# Базис jsonfeed-util (repo: jsonfeed)

В репозитории `jsonfeed` в файле `jsonfeed/jsonfeed/__init__.py` реализована библиотека для работы со стандартом JSON Feed 1.1. Эта микро-библиотека идеальна как строгий стандарт валидации для слоя Emitters в системе Personal Browser Dashboard (PBD). В PBD любой источник должен конвертироваться в `JSON Feed 1.1`.

## Структура классов (базис для Pydantic-контрактов)

Оригинальная реализация использует обычные классы Python с методами `parse(dict)` и `_to_ordered_dict()`. Для PBD эти классы легко и нужно адаптировать в строгие `Pydantic` модели (`BaseModel`).

### 1. Feed (Основной объект)

Хранит метаданные ленты и список элементов.
- Обязательные поля: `title`.
- Дополнительные: `home_page_url`, `feed_url`, `description`, `user_comment`, `next_url`, `icon`, `favicon`, `expired`, `language`, `author`, `authors`, `hubs`, `items`.

*Адаптация в Pydantic:*
```python
from pydantic import BaseModel, HttpUrl, Field
from typing import Optional, List

class Feed(BaseModel):
    version: str = "https://jsonfeed.org/version/1.1"
    title: str
    home_page_url: Optional[HttpUrl] = None
    feed_url: Optional[HttpUrl] = None
    description: Optional[str] = None
    user_comment: Optional[str] = None
    next_url: Optional[HttpUrl] = None
    icon: Optional[HttpUrl] = None
    favicon: Optional[HttpUrl] = None
    expired: bool = False
    language: Optional[str] = None
    authors: Optional[List['Author']] = Field(default_factory=list)
    hubs: Optional[List['Hub']] = Field(default_factory=list)
    items: List['Item'] = Field(default_factory=list)
```

### 2. Item (Элемент/Пост/Атом)

Конкретная запись в ленте. Это базис для `Универсального Атома` (Universal Atom) в PBD.
- Обязательные поля: `id`.
- Дополнительные: `url`, `external_url`, `title`, `content_html`, `content_text`, `summary`, `image`, `banner_image`, `date_published`, `date_modified`, `authors`, `tags`, `attachments`.

*Адаптация в Pydantic:*
```python
from datetime import datetime

class Item(BaseModel):
    id: str
    url: Optional[HttpUrl] = None
    external_url: Optional[HttpUrl] = None
    title: Optional[str] = None
    content_html: Optional[str] = None
    content_text: Optional[str] = None
    summary: Optional[str] = None
    image: Optional[HttpUrl] = None
    banner_image: Optional[HttpUrl] = None
    date_published: Optional[datetime] = None
    date_modified: Optional[datetime] = None
    authors: Optional[List['Author']] = Field(default_factory=list)
    tags: Optional[List[str]] = Field(default_factory=list)
    attachments: Optional[List['Attachment']] = Field(default_factory=list)
```

### 3. Author и Attachment (Вспомогательные сущности)

*Адаптация в Pydantic:*
```python
class Author(BaseModel):
    name: Optional[str] = None
    url: Optional[HttpUrl] = None
    avatar: Optional[HttpUrl] = None

class Attachment(BaseModel):
    url: HttpUrl
    mime_type: str
    title: Optional[str] = None
    size_in_bytes: Optional[int] = None
    duration_in_seconds: Optional[int] = None

class Hub(BaseModel):
    type: str
    url: HttpUrl
```

## Как применить в PBD (Ingestion Layer)

1. Создать пакет `pbd.contracts.jsonfeed` и поместить туда вышеуказанные Pydantic-модели.
2. В слое **Fetchers & Emitters** (HTTP, IMAP, Telegram сборщики) обязать каждый адаптер возвращать объект модели `Feed` или `Item`.
3. Благодаря Pydantic, мы автоматически получим строгую валидацию типов (URL-адреса, даты RFC 3339) и защиту от некорректных данных на самом входе в систему (L1_raw).
4. Дедупликация (UUIDv7) будет применяться к полю `id` у `Item`.
