# Технический анализ архитектуры python-substack и адаптация для PostgreSQL

Настоящий отчет содержит детальный анализ логики работы библиотеки `python-substack`, которая позволяет программно взаимодействовать с платформой Substack, извлекать данные постов, авторов и статистику подписчиков. Отчет структурирован из трех частей согласно требованиям, и адаптирован для интеграции в PostgreSQL-базированный проект.

## 1. Техническая реализация и архитектура

### HTTP-клиент и Управление сессиями
Библиотека построена вокруг класса `Api` (`substack/api.py`), который служит оберткой (wrapper) над REST-подобными эндпоинтами Substack, используя стандартную библиотеку `requests`.
- **Сессии:** Основным механизмом сохранения состояния является объект `requests.Session`. Он позволяет сохранять cookies, необходимые для аутентификации между запросами.
- **Аутентификация:** Поддерживается вход по email/password, но более важным механизмом является использование `cookies_string` или файла `cookies_path`. Авторизация (логин) эмулируется путем отправки POST-запроса на `/api/v1/login`, после чего сервер возвращает cookie `substack.sid` (и другие), обеспечивающие доступ к приватным данным (например, к черновикам, метрикам публикации).
- **Обработка ошибок:** Статический метод `_handle_response` проверяет статус коды HTTP (от 200 до 299). В случае ошибки выбрасывается `SubstackAPIException`. При успешном ответе данные автоматически декодируются из JSON: `return response.json()`. Если ответ не является валидным JSON (например, сервер вернул HTML при ошибке маршрутизации), выбрасывается `SubstackRequestException`.

**Реализация в PostgreSQL-проекте:**
В бэкенде (например, FastAPI/Node.js), который будет работать с PostgreSQL, необходимо:
- Хранить токены и cookies в базе данных, чтобы фоновые воркеры (напр., Celery, Airflow, в духе 'auto-news' или 'instagrapi') могли выполнять запросы без повторной авторизации.
- Реализовать пул сессий.

### Эндпоинты (REST API)
Библиотека обращается к внутреннему непубличному API Substack (обычно `/api/v1/...` или прямые URL):
- `/api/v1/reader/posts` - лента пользователя.
- `/api/v1/drafts`, `/api/v1/post_management/published` - управление публикациями и черновиками.
- `/api/v1/publication_launch_checklist` - отсюда нетривиально парсится `subscriberCount` (количество подписчиков).
- `/api/v1/category/public/{category_id}/{category_type}` - списки авторов и публикаций по категориям.

## 2. Функции и логическое функционирование

### Жизненный цикл извлечения данных
1. **Инициализация:** При создании объекта `Api` загружаются куки (из файла или строки) или происходит логин. Это позволяет обращаться к `self.base_url` (читалка) или `self.publication_url` (админка публикации).
2. **Запрос данных:** Например, вызов `get_published_posts(offset=0, limit=25)` формирует GET запрос к `/api/v1/post_management/published` с query-параметрами пагинации.
3. **Парсинг:** Substack возвращает JSON. Данные содержат метаданные поста (заголовок, дата, автор, статистика лайков/комментариев) и, в случае запроса конкретного поста (`get_draft`), его содержимое.
4. **Объекты-обертки:** В файле `substack/post.py` реализована абстракция `Post`, позволяющая конструировать посты программно, добавлять секции и публиковать их.
5. **Извлечение статистики:** Для получения количества подписчиков логика обращается к эндпоинту `publication_launch_checklist`, возвращаемому JSON'у и извлекает ключ `["subscriberCount"]`.

### Интеграция в конвейер данных (Data Pipeline)
В архитектуре, похожей на 'auto-news', логическое функционирование должно быть построено через DAG'и (Directed Acyclic Graphs):
- **Скрейпер-воркер:** Запускается по расписанию, инициализирует сессию, берет `offset` из PostgreSQL.
- Получает пачку постов (JSON).
- Разбивает JSON на реляционные данные: профиль автора (`users`), публикацию (`publications`) и сам пост (`posts`).
- Для векторного поиска (например, с pgvector) текстовое содержимое поста очищается от HTML-тегов, прогоняется через LLM для получения эмбеддингов и сохраняется в БД.

## 3. Обмен информацией, память и потоки данных (Схема данных)

Для хранения данных, извлеченных с Substack в базе данных PostgreSQL (с учетом гибридной модели реляционных данных + JSONB для гибкости), предлагается следующая схема.

### Схема данных для PostgreSQL

```sql
-- Таблица аккаунтов (управление сессиями 'instagrapi' style)
CREATE TABLE substack_accounts (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    cookies_json JSONB NOT NULL, -- Хранение substack.sid и других куки
    is_active BOOLEAN DEFAULT TRUE,
    last_login_at TIMESTAMP WITH TIME ZONE
);

-- Таблица публикаций (Publications / Newsletters)
CREATE TABLE publications (
    id VARCHAR(100) PRIMARY KEY, -- Внутренний ID Substack
    name VARCHAR(255) NOT NULL,
    base_url TEXT UNIQUE NOT NULL,
    author_id VARCHAR(100),
    subscriber_count INT DEFAULT 0,
    category_id VARCHAR(100),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Таблица авторов (Authors / Users)
CREATE TABLE authors (
    id VARCHAR(100) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    bio TEXT,
    profile_image_url TEXT,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Таблица постов (Posts / Drafts)
CREATE TABLE substack_posts (
    id VARCHAR(100) PRIMARY KEY,
    publication_id VARCHAR(100) REFERENCES publications(id) ON DELETE CASCADE,
    author_id VARCHAR(100) REFERENCES authors(id) ON DELETE SET NULL,
    title TEXT NOT NULL,
    subtitle TEXT,
    post_date TIMESTAMP WITH TIME ZONE NOT NULL,
    is_published BOOLEAN DEFAULT TRUE,

    -- Метрики извлекаемые из JSON (stats)
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,

    -- Контент и сырые данные
    content_html TEXT,
    raw_json JSONB, -- Полный ответ API для гибкости (в стиле 'auto-news')

    -- Для интеграции RAG/pgvector
    embedding vector(1536),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_posts_pub_date ON substack_posts(publication_id, post_date DESC);
CREATE INDEX idx_posts_embedding ON substack_posts USING ivfflat (embedding vector_cosine_ops);
```

### Потоки данных (Data Flows)
1. **Авторизация:** Воркер запрашивает `cookies_json` из `substack_accounts` и инициализирует сессию библиотеки `python-substack`.
2. **Сбор данных:** Воркер опрашивает `get_published_posts()`. Возвращаемый список обрабатывается:
    - Обновляются счетчики в таблице `publications` (например, `subscriberCount`).
    - Идентификаторы авторов проверяются в таблице `authors` (Upsert).
    - Новые посты сохраняются в `substack_posts`. Для сложных структур (вложения, метаданные) используется столбец `raw_json` (`JSONB`).
3. **Обогащение:** В асинхронном режиме другой воркер берет посты без `embedding`, очищает `content_html`, генерирует векторы и сохраняет их обратно в PostgreSQL.
4. **Потребление (Поиск):** Клиентские агенты (в стиле 'aif-handoff') могут делать гибридные SQL/Cypher запросы (на основе графовой архитектуры), объединяя полнотекстовый поиск по `title`, графовые связи (Автор -> Публикация -> Пост) и векторное сходство по `embedding`.
