# Отчет по архитектуре и данным библиотек для скрапинга Twitter (X)

Этот отчет подготовлен на основе прямого анализа исходного кода шести указанных репозиториев (`twscrape`, `twitter-advanced-search`, `TweeterPy`, `twitter-api-client`, `XRSS`, `twitter-rss`). Отчет структурирован для последующей реализации аналогичного функционала в вашем проекте на PostgreSQL.

## 1. Техническая реализация и архитектура

Большинство проанализированных инструментов взаимодействуют с недокументированным внутренним GraphQL API платформы X (Twitter).

- **twscrape (Python):** Асинхронная библиотека (использует `httpx`). Использует локальную базу данных SQLite (`aiosqlite`) для хранения пула аккаунтов, управления блокировками и ротацией сессий/прокси. Это мощный инструмент для массового сбора данных, так как он автоматически обходит лимиты, переключаясь между учетными записями.
- **twitter-api-client (Python):** Синхронная/асинхронная оболочка над GraphQL API. Не использует базу данных, сохраняет результаты прямо в файлы `.json` или возвращает в виде словарей. Содержит скрипты для автоматического обновления конечных точек (endpoints) GraphQL, парсинга аудио-пространств (Live Spaces) и перехвата веб-сокетов.
- **TweeterPy (Python):** Похож на `twitter-api-client`. Использует `requests` для синхронных вызовов к GraphQL. API endpoints и features switches прописаны в файле констант. Сохраняет сессии (cookies, токены) в локальные файлы конфигурации.
- **twitter-rss (TypeScript):** Утилита на Node.js (с использованием `twitter-openapi-typescript` и `fast-xml-parser`). Запускается в Docker вместе с виртуальным дисплеем (Xvfb) и Puppeteer для эмуляции браузера и обхода защиты (TLS fingerprinting через CycleTLS). Не использует БД, сохраняет результат в XML файлы на диске.
- **XRSS (Python):** Web-приложение на FastAPI. Использует `twikit` (еще одна неофициальная библиотека) для взаимодействия с Twitter и `Redis` для кэширования ответов. Формирует RSS-ленту с помощью `feedgen`.
- **twitter-advanced-search:** Этот репозиторий является справочником (документацией в `README.md`) по операторам расширенного поиска Twitter (например, `from:`, `filter:media`, `since_time:`), а не программным кодом. Эти операторы передаются параметром `rawQuery` в GraphQL API во всех вышеупомянутых библиотеках.

## 2. Функции и логическая работа

Логика всех библиотек сводится к трем основным процессам:

1.  **Аутентификация и сессии:** Поскольку API X требует авторизации, библиотеки эмулируют веб-клиент. Они используют `ct0` (CSRF токен) и `auth_token` (токен авторизации из Cookies). Если токенов нет, скрипты (`XRSS`, `twitter-rss`, `twscrape`) выполняют полноценный логин по юзернейму, паролю и, при необходимости, TOTP-коду или подтверждению по email.
2.  **Обход пагинации (Cursor Pagination):** GraphQL API не использует страницы по номерам. В конце каждого ответа (timeline) возвращается объект курсора (`cursorType: Bottom`). Библиотеки (`TweeterPy`, `twitter-api-client`, `twscrape`) в цикле отправляют этот курсор в следующем запросе для получения следующей порции твитов или пользователей.
3.  **Кэширование и анти-бан:**
    *   **twscrape** использует SQLite для ведения статистики ошибок по каждому аккаунту и может заморозить аккаунт (`active=FALSE`) при получении ошибки rate limit.
    *   **XRSS** кэширует профили пользователей и ленту твитов в Redis с установленным временем жизни (TTL), а также имеет фоновую задачу для автоматического обновления кэша за секунды до его истечения, чтобы пользователю не приходилось ждать.
    *   Генерация RSS происходит путем трансформации собранного JSON: извлечение медиа, формирование HTML-описания (защита от XSS экранированием в `twitter-rss`) и маппинг в XML теги.

## 3. Обмен информацией, память и потоки данных (схемы для PostgreSQL)

Ответы от GraphQL API приходят в виде глубоко вложенных JSON словарей. `twscrape` — единственная из библиотек, которая строго типизирует ответ с помощью Python `dataclasses` (в `models.py`), парся сырой JSON в структурированные объекты. `TweeterPy` и `twitter-api-client` просто возвращают или сохраняют сырой JSON.

Для реализации этого в **PostgreSQL**, учитывая архитектурный вдохновляющий пример с гибридным использованием графов и RDBMS, рекомендуется следующая схема.

### Управление аккаунтами (по образу `twscrape` `db.py`)

```sql
CREATE TABLE twitter_accounts (
    username VARCHAR(255) PRIMARY KEY,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    user_agent TEXT NOT NULL,
    active BOOLEAN DEFAULT TRUE,
    cookies JSONB DEFAULT '{}'::jsonb, -- хранение auth_token и ct0
    headers JSONB DEFAULT '{}'::jsonb,
    stats JSONB DEFAULT '{}'::jsonb,   -- количество запросов, ошибки
    proxy VARCHAR(255),
    last_used TIMESTAMP WITH TIME ZONE
);
```

### Схема данных: Пользователи (Users)

Информация извлекается из `user_results.result`.

```sql
CREATE TABLE twitter_users (
    id BIGINT PRIMARY KEY, -- id_str (напр. 44196397)
    username VARCHAR(255) UNIQUE NOT NULL, -- screen_name (без @)
    display_name VARCHAR(255),
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE,
    followers_count INT,
    friends_count INT,
    statuses_count INT,
    profile_image_url TEXT,
    is_verified BOOLEAN DEFAULT FALSE,
    is_blue BOOLEAN DEFAULT FALSE,
    raw_data JSONB -- Полный оригинальный JSON пользователя для гибкости
);
```

### Схема данных: Твиты (Tweets)

Извлекается из `tweet_results.result`. Твиты могут быть постами, реплаями, ретвитами или квотами.

```sql
CREATE TABLE tweets (
    id BIGINT PRIMARY KEY,
    user_id BIGINT REFERENCES twitter_users(id),
    conversation_id BIGINT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    raw_content TEXT, -- Полный текст твита (в GraphQL это full_text)
    lang VARCHAR(10),

    -- Статистика (метрики)
    reply_count INT DEFAULT 0,
    retweet_count INT DEFAULT 0,
    like_count INT DEFAULT 0,
    quote_count INT DEFAULT 0,
    view_count INT DEFAULT 0,

    -- Ссылки на другие твиты (для построения графа разговоров / ретвитов)
    in_reply_to_tweet_id BIGINT,
    in_reply_to_user_id BIGINT,
    retweeted_tweet_id BIGINT,
    quoted_tweet_id BIGINT,

    -- Медиа, хэштеги и линки (лучше хранить в JSONB для удобного парсинга)
    entities JSONB,
    -- Содержит массивы: media (фото/видео URL), hashtags, urls, user_mentions

    raw_data JSONB -- Сохраняем оригинальный JSON для обратной совместимости
);

-- Индексы для ускорения работы
CREATE INDEX idx_tweets_user_id ON tweets(user_id);
CREATE INDEX idx_tweets_conversation ON tweets(conversation_id);
CREATE INDEX idx_tweets_entities_gin ON tweets USING GIN (entities);
```

### Поток данных (Data Flow) в вашем приложении:
1.  **Оркестрация (Background Tasks):** Фоновый worker берет `active` аккаунт из `twitter_accounts`.
2.  **Запрос:** Делается HTTPS запрос к GraphQL API X (с передачей cookies, headers, прокси). Для поиска используются параметры из `twitter-advanced-search`.
3.  **Ингрест:** Полученный JSON разбирается.
    *   Если найдены новые пользователи, они вставляются в `twitter_users` через `ON CONFLICT (id) DO UPDATE`.
    *   Твиты вставляются в `tweets` аналогичным образом (Upsert). Из массива `entities.media` извлекаются URL (помните правило из памяти: *public HTTPS URLs for all media files*).
4.  **Агрегация (XRSS подход):** При запросе ленты RSS вашим фронтендом, запрос идет к PostgreSQL (или Redis кэшу). Из `tweets` выбираются записи по `user_id`, сортируются по `created_at DESC`, текст и медиа из `entities` форматируются в HTML для генерации XML RSS.
