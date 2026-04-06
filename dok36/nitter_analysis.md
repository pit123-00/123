# Технический анализ архитектуры Nitter и адаптация для PostgreSQL

Настоящий отчет содержит детальный анализ реализации хранения данных, генерации представлений и проксирования медиа в проекте Nitter, а также рекомендации по переносу этой логики в базу данных PostgreSQL. Отчет структурирован согласно требуемому формату из трех частей.

## 1. Техническая реализация и архитектура

### Архитектура кэширования (Redis -> PostgreSQL)
В Nitter основным механизмом хранения извлеченных из Twitter данных является Redis (`redis_cache.nim`). Он используется не только как временный кэш, но и как основное хранилище для переиспользования данных, чтобы не превышать лимиты API Twitter.
- **Сериализация:** Данные (объекты `User`, `Tweet`, `List`) сериализуются в бинарный формат библиотекой `flatty`, затем сжимаются с помощью `supersnappy` и сохраняются в Redis с заданным временем жизни (TTL) — обычно `baseCacheTime` (1 час).
- **Ключи:** Используются префиксы: `p:username` для профилей, `t:id` для твитов, `uid:name` для маппинга имени пользователя в ID, `rss:query` для кэширования RSS-лент.

**Реализация в PostgreSQL:**
Для переноса этой архитектуры в PostgreSQL можно использовать комбинацию реляционного хранения для постоянных данных и механизма временного хранения:
- Использование **UNLOGGED таблиц** для временного кэша RSS или твитов, которые можно потерять при перезагрузке сервера, но к которым нужен быстрый доступ.
- **JSONB** для хранения сложных структур (например, вариантов видео, опций опросов, массива медиа-файлов), что позволит избежать чрезмерной нормализации и ускорить чтение.
- Поле `expires_at (TIMESTAMP)` для реализации механизма TTL. Планировщик (например, `pg_cron`) может периодически удалять устаревшие записи, эмулируя поведение `EXPIRE` из Redis.

### Генерация представлений (Nim-макросы HTML)
Nitter генерирует страницы на лету на сервере (Server-Side Rendering) без использования клиентского JavaScript. Для этого применяется библиотека Karax (`views/tweet.nim`, `views/timeline.nim`).
- Макросы (такие как `buildHtml(tdiv)`) превращают Nim-код в легковесный HTML.
- Маршрутизатор (роутер) определяет, нужен ли HTML или RSS XML (например, при запросе `/username/rss`), и вызывает соответствующий генератор (`routes/rss.nim`).
- **Перезапись URL медиа:** Все ссылки на внешние ресурсы (картинки, видео с `pbs.twimg.com` или `video.twimg.com`) подменяются на локальные пути. Функция `getPicUrl` (`utils.nim`) преобразует ссылку в `/pic/enc/...` (Base64 + HMAC подпись) или `/pic/...` (URL-кодирование).

### Проксирование медиа
Роутинг `routes/media.nim` перехватывает запросы к `/pic/...` и `/video/...`.
- Nitter делает асинхронный HTTP GET-запрос к оригинальному серверу Twitter, получая контент.
- Затем он передает заголовки (Content-Type, Content-Length) и поток данных (stream) клиенту.
- Реализована проверка кэша на клиенте (`If-None-Match` и `ETag`, в качестве которого выступает хэш от URL), чтобы отдавать `HTTP 304 Not Modified` и экономить трафик.
- Это скрывает IP-адрес конечного пользователя от Twitter и обходит блокировки CORS/трекинга.
- Также важно соблюдать требование предоставления публичных HTTPS URL с правильными заголовками CORS для всех медиафайлов.

## 2. Функции и логическое функционирование

### Жизненный цикл обработки запроса
1. **Запрос:** Пользователь или RSS-ридер делает запрос к профилю (например, `/elonmusk`).
2. **Поиск в кэше:** Nitter проверяет наличие пользователя в Redis (`getCachedUser`). Если данные свежие, они десериализуются и используются мгновенно.
3. **Парсинг (при промахе кэша):** Если данных нет, выполняется GraphQL/API запрос к Twitter. Полученный JSON парсится (`parser.nim`), формируются структуры `User` и `Tweets`.
4. **Сохранение в кэш:** Сформированные объекты сжимаются и сохраняются в Redis (функции `cache(data: User)`, `cache(data: Tweet)`), обновляется кэш RSS.
5. **Рендеринг:** В генератор представлений передаются собранные данные. При формировании блоков (например, в функции `renderPhotoAttachment` в `views/tweet.nim`) оригинальные `photo.url` оборачиваются в `getOrigPicUrl()`.
6. **Выдача клиенту:** Готовый HTML/XML возвращается клиенту.

### Логика работы проксирования медиа в Postgres-проекте
В разрабатываемом проекте на PostgreSQL и бэкенде (например, Node.js, Python/FastAPI) логику проксирования можно реализовать следующим образом:
- Создать эндпоинт `/api/media/proxy?url=...&signature=...`.
- Бэкенд валидирует `signature` (HMAC), чтобы предотвратить использование сервера как открытого прокси (Open Proxy).
- Если запрос валиден, бэкенд скачивает медиа или использует потоковую передачу (streaming pipe).
- Для часто запрашиваемых медиафайлов можно сохранять их метаданные в таблицу PostgreSQL `media_cache (url_hash, content_type, local_path, created_at)` и загружать сами файлы в S3-совместимое хранилище или файловую систему, отдавая ссылки через CDN, что снизит нагрузку на бэкенд.

## 3. Обмен информацией, память и потоки данных (Схема данных)

Для переноса модели Nitter в PostgreSQL предлагается следующая реляционная схема данных, основанная на структурах `types.nim`.

### Основные сущности (Схема данных для PostgreSQL)

```sql
-- Таблица пользователей (User)
CREATE TABLE users (
    id VARCHAR(50) PRIMARY KEY, -- 'id_str' из Twitter
    username VARCHAR(100) UNIQUE NOT NULL,
    fullname VARCHAR(255),
    location TEXT,
    bio TEXT,
    user_pic TEXT, -- Оригинальный URL
    banner TEXT,
    following_count INT DEFAULT 0,
    followers_count INT DEFAULT 0,
    tweets_count INT DEFAULT 0,
    likes_count INT DEFAULT 0,
    media_count INT DEFAULT 0,
    is_protected BOOLEAN DEFAULT FALSE,
    verified_type VARCHAR(20), -- 'None', 'Blue', 'Business', 'Government'
    join_date TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() -- для логики кэша
);

-- Таблица твитов (Tweet)
CREATE TABLE tweets (
    id BIGINT PRIMARY KEY,
    thread_id BIGINT,
    reply_id BIGINT,
    user_id VARCHAR(50) REFERENCES users(id) ON DELETE CASCADE,
    text TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    is_pinned BOOLEAN DEFAULT FALSE,
    has_thread BOOLEAN DEFAULT FALSE,
    stats_replies INT DEFAULT 0,
    stats_retweets INT DEFAULT 0,
    stats_likes INT DEFAULT 0,
    stats_views INT DEFAULT 0,
    retweet_id BIGINT REFERENCES tweets(id) ON DELETE SET NULL,
    quote_id BIGINT REFERENCES tweets(id) ON DELETE SET NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Таблица медиа (MediaEntities)
CREATE TABLE media (
    id SERIAL PRIMARY KEY,
    tweet_id BIGINT REFERENCES tweets(id) ON DELETE CASCADE,
    kind VARCHAR(20) NOT NULL, -- 'photoMedia', 'videoMedia', 'gifMedia'
    url TEXT NOT NULL, -- Оригинальный URL
    thumb_url TEXT,
    alt_text TEXT,
    video_duration_ms INT,
    video_variants JSONB, -- Варианты разрешений и битрейта (Resolution, Bitrate, ContentType)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Таблица кэшированных RSS-лент (замена Redis rss:*)
CREATE UNLOGGED TABLE rss_cache (
    query_key VARCHAR(255) PRIMARY KEY,
    feed_xml TEXT,
    min_cursor VARCHAR(255),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL
);
CREATE INDEX idx_rss_expires ON rss_cache(expires_at);
```

### Потоки данных (Data Flows)
1. **Ingestion (Ввод):** Подсистема парсинга извлекает JSON, разбивает его на `users`, `tweets` и `media`, и выполняет `INSERT ... ON CONFLICT DO UPDATE` (Upsert) в PostgreSQL. Обновляется поле `updated_at`.
2. **Retrieval (Извлечение):** При запросе профиля данные выбираются из таблиц с использованием индексов (например, индекс по `user_id` и `created_at` `DESC` в таблице `tweets`). Если `updated_at` старше `baseCacheTime`, инициируется фоновый процесс обновления данных (через брокер сообщений или Celery/Airflow).
3. **Presentation (Выдача):** На уровне SQL-запроса или ORM выбираются твиты и JOIN'ятся с таблицей `media`. При сериализации в JSON для фронтенда или при рендеринге HTML, функции форматирования "на лету" подменяют оригинальные `url` медиа-ресурсов на прокси-ссылки (например, `/api/proxy/media?hmac=...&url=...`). Фронтенд (Canvas/OpenCut) сможет загружать эти файлы, так как бэкенд отдаст публичные HTTPS-ссылки с корректными заголовками CORS.
