# Технический анализ архитектуры Spodcast и адаптация для PostgreSQL

Настоящий отчет содержит детальный анализ логики работы библиотеки `Spodcast`, которая позволяет скачивать эксклюзивные подкасты Spotify и генерировать RSS-ленты. Отчет структурирован из трех частей согласно требованиям, и адаптирован для интеграции в PostgreSQL-базированный проект (в духе многомодельной графовой базы и фоновых задач на Airflow/Celery).

## 1. Техническая реализация и архитектура

### Интеграция с Spotify API (librespot)
В отличие от библиотек, использующих стандартный HTTP-скрейпинг, `Spodcast` в значительной степени полагается на библиотеку `librespot` (написанную на Python), которая является портом оригинальной Rust-библиотеки с открытым исходным кодом.
- **Аутентификация:** Аутентификация (`spodcast.py`, метод `login`) работает не только по логину/паролю, но и поддерживает сессионные ключи/токены (credentials JSON файлы), что позволяет имитировать полноценный Spotify-клиент (Session). Это аналогично подходу, используемому в 'instagrapi' (сохранение cookies/сессий).
- **Манифесты и Метаданные:** Извлечение информации (`podcast.py`, `get_show_info`, `get_episode_info`) выполняется через внутренние API Spotify (Mercury/Hermes API), обращаясь к объектам `ShowId` и `EpisodeId`, и декодируя Protobuf-сообщения, возвращающие `info.show.name`, `info.duration` и другие данные о подкастах.

### Загрузка и транскодинг (FFmpeg)
В функции `download_episode` (`podcast.py`) реализовано два пути получения аудио:
1. **Открытые потоки (download_url):** Если подкаст имеет `external_url` (загружен извне Spotify), он просто скачивается по HTTP (`requests.get(url, stream=True)`).
2. **Зашифрованные потоки Spotify:** Если подкаст — эксклюзив Spotify, используется механизм `Spodcast.get_content_stream(episode_stream_id)`. `librespot` скачивает зашифрованные куски (чанки) в формате OGG Vorbis, дешифрует их "на лету", и затем (при наличии настроек) `ffmpeg` объединяет/транскодирует их в единый MP3/M4A файл.

### Генерация RSS-лент
Для каждого скачанного эпизода создается `.info` JSON-файл с метаданными. Файл `feedgenerator.py` собирает эти данные и формирует стандартный `feed.xml` для подкаст-клиентов.

## 2. Функции и логическое функционирование

### Жизненный цикл обработки запроса
1. **Парсинг URL:** Пользователь передает ссылку на подкаст. Регулярные выражения извлекают `show_id` или `episode_id` в формате Base62.
2. **Декодирование ID:** Base62 конвертируется в HEX (`EpisodeId.from_base62(episode_id_str).hex_id()`).
3. **Получение эпизодов:** Если передан `show_id`, вызывается `get_episodes`, который запрашивает метаданные шоу и возвращает список ID эпизодов, сортируя их по дате публикации (`info.publish_time`).
4. **Скачивание и обработка:** Для каждого эпизода:
   - Проверяется, существует ли уже файл (по `FILE_EXISTS`).
   - Если нет, открывается поток (stream) с серверов Spotify CDN.
   - Файл сохраняется локально (`show_directory`).
5. **Сохранение метаданных:** Создается словарь `episode_info` (mimetype, duration, date, title, description, filename) и сохраняется на диск. Позже этот файл используется для генерации XML-ленты.

### Интеграция в PostgreSQL проект (Data Pipeline)
В разрабатываемой системе архитектура Spodcast может быть адаптирована следующим образом:
- **Управление сессиями:** Сессии (credentials JSON, генерируемые librespot) хранятся в БД, а фоновый планировщик ('Airflow DAG' или 'Celery worker') берет активную сессию для опроса обновлений подкастов.
- **Хранилище медиа:** Скачанные и транскодированные MP3-файлы не просто складываются в файловую систему, а загружаются в S3/Cloud. URL файла сохраняется в PostgreSQL в таблице `media`. Бэкенд на Hono может раздавать их как HTTPS-прокси с CORS-заголовками, аналогично требованиям 'nitter'.
- **Динамический RSS:** Вместо статической генерации `feed.xml` на диске, бэкенд на лету генерирует RSS-ответ из PostgreSQL при обращении по роуту `/rss/:show_id`.

## 3. Обмен информацией, память и потоки данных (Схема данных)

Для переноса модели `Spodcast` в PostgreSQL предлагается реляционная схема с поддержкой JSONB и векторов.

### Схема данных для PostgreSQL

```sql
-- Таблица аккаунтов Spotify (Управление сессиями)
CREATE TABLE spotify_accounts (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    credentials_json JSONB NOT NULL, -- Токены и кэш librespot
    is_active BOOLEAN DEFAULT TRUE,
    last_used_at TIMESTAMP WITH TIME ZONE
);

-- Таблица подкастов (Шоу)
CREATE TABLE podcasts (
    id VARCHAR(50) PRIMARY KEY, -- Spotify Show ID (HEX или Base62)
    name VARCHAR(255) NOT NULL,
    description TEXT,
    cover_image_url TEXT,
    spotify_uri VARCHAR(100) UNIQUE,
    publisher VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Таблица эпизодов
CREATE TABLE podcast_episodes (
    id VARCHAR(50) PRIMARY KEY, -- Spotify Episode ID
    podcast_id VARCHAR(50) REFERENCES podcasts(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    release_date TIMESTAMP WITH TIME ZONE NOT NULL,
    duration_ms INT NOT NULL,
    spotify_uri VARCHAR(100) UNIQUE,
    external_url TEXT, -- Ссылка на внешний CDN, если есть

    -- Медиаданные
    local_file_path TEXT, -- S3 URL или относительный путь после FFmpeg
    mimetype VARCHAR(50),
    file_size_bytes BIGINT,

    -- Динамические данные (protobuf/json)
    raw_metadata JSONB,

    -- Для интеграции RAG/pgvector (если мы делаем транскрибацию Whisper)
    transcript_text TEXT,
    embedding vector(1536),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_episodes_podcast_date ON podcast_episodes(podcast_id, release_date DESC);
CREATE INDEX idx_episodes_embedding ON podcast_episodes USING ivfflat (embedding vector_cosine_ops);
```

### Потоки данных (Data Flows)
1. **Оркестрация:** Фоновая задача ('huginn' style) инициирует проверку новых эпизодов для всех `podcasts` в БД, используя авторизацию из `spotify_accounts`.
2. **Получение данных:** `get_episodes(show_id)` извлекает новые `EpisodeId`, которых нет в `podcast_episodes`.
3. **Загрузка и транскодинг (Асинхронно):** Воркер ставит в очередь загрузку. `librespot` скачивает чанки. Поток перенаправляется в FFmpeg (через Python subprocess/pipes). Готовый MP3 отправляется в S3 хранилище.
4. **Обновление БД:** После успешной загрузки создается запись в `podcast_episodes` с `local_file_path` и `duration_ms`.
5. **AI-Пайплайн (Дополнительно):** Отдельный 'subagent' (в стиле 'opencode/agents') забирает аудиофайл, прогоняет через Whisper API (speech-to-text), сохраняет текст в `transcript_text`, генерирует эмбеддинги для `embedding` колонки. Это позволяет реализовать гибридный векторный поиск (по тексту) внутри аудиоподкастов.
6. **Выдача:** Приложение выдает динамический RSS по запросу, вытаскивая данные из PostgreSQL, тем самым реализуя полностью 'подкастовый' RAG (Graph RAG).
