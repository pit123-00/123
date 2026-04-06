# Отчет по архитектуре и логике tiktok-rss-flat

Данный отчет содержит анализ репозитория [tiktok-rss-flat](https://github.com/conoro/tiktok-rss-flat) и описывает его техническую архитектуру, логические функции и потоки данных, а также предлагает схему для реализации аналогичного функционала в PostgreSQL.

## 1. Техническая реализация и архитектура

Проект построен как легковесный парсер, который выполняется в виде регулярной задачи (cron) в GitHub Actions и использует GitHub Pages для хостинга сгенерированных статических RSS/XML файлов.

### Основные компоненты:
*   **Среда выполнения**: GitHub Actions (Runner `ubuntu-latest`).
*   **Язык и библиотеки**: Python 3.12 (скрипт `postprocessing.py`).
    *   `TikTokApi`: Неофициальная библиотека (обертка/парсинг) для получения данных из TikTok.
    *   `playwright`: Фреймворк для автоматизации браузера (используется `chromium` в headless/non-headless режимах) для обхода защиты от ботов, выполнения JavaScript, и рендеринга страницы для скриншотов обложек видео.
    *   `feedgen`: Библиотека для формирования RSS/Atom фидов (генерация `XML`).
*   **Хранилище (Flat files / Flat Data)**:
    *   `subscriptions.csv`: Список отслеживаемых TikTok-пользователей.
    *   В репозитории создаются папки `rss/` для `.xml` файлов и `thumbnails/` для изображений-обложек.
*   **Оркестрация**: Workflow GitHub Actions (`.github/workflows/flat.yml`) срабатывает каждые 4 часа. Он настраивает Python-окружение, запускает виртуальный дисплей `Xvfb` (для playwright), выполняет скрипт, а затем делает коммит и пушит изменения (сгенерированные XML и картинки) обратно в ветку `main`.

## 2. Функции и логическая работа

Основная логика содержится в файле `postprocessing.py`. Выполнение идет по следующим шагам:

1.  **Инициализация конфигурации**: Скрипт читает базовые URL для GitHub Pages из `config.py` (чтобы указывать правильные ссылки на RSS и картинки) и извлекает токен `MS_TOKEN` из переменных окружения. Токен сессии необходим для авторизации или имитации сессии обычного браузера при запросах к TikTok API (для обхода капчи/ограничений).
2.  **Обход списка пользователей**: Скрипт считывает `subscriptions.csv`, получая `username` (например, `iamtabithabrown`).
3.  **Генерация заголовка RSS**: Используя `feedgen`, инициализируется RSS-лента с указанием автора, заголовка, логотипа и ссылки на сам фид.
4.  **Запрос к TikTok**:
    *   Асинхронно создается сессия в `TikTokApi` (`api.create_sessions`) с подстановкой `ms_token`.
    *   Вызывается `api.user(user).videos(count=10)` — получение последних 10 видео пользователя.
5.  **Парсинг видео и генерация записей в XML**:
    *   Для каждого видео создается `entry` (запись) в RSS.
    *   *ID и ссылка*: формируется URL вида `https://tiktok.com/@{user}/video/{video.id}`.
    *   *Дата*: берется timestamp создания видео (`createTime`) и конвертируется в формат времени (UTC).
    *   *Описание (title/content)*: Извлекается текст поста (`desc`).
6.  **Работа с медиа (обложки видео)**:
    *   Скрипт извлекает URL обложки (`video.as_dict['video']['cover']`).
    *   Проверяет, существует ли локальный файл скриншота в `/thumbnails/`.
    *   Если файла нет, запускается **Playwright** (`runscreenshot()`). Он открывает в браузере URL обложки и делает ее скриншот (JPEG, quality 20), сохраняя на диск. Это делается, чтобы изображения отдавались с GitHub Pages и не было проблем с CORS или битыми ссылками из-за истечения срока действия токенов в CDN TikTok (типичная проблема со ссылками на ресурсы TikTok).
    *   В тело XML добавляется тег `<img>`, который ссылается на сохраненную обложку.
7.  **Сохранение**: Файл XML сохраняется в директорию `rss/`. Все изменения пушатся обратно в GitHub.

## 3. Обмен информацией, память и потоки данных

*   **Входные данные**:
    *   Статические: `subscriptions.csv` (список юзернеймов).
    *   Секреты: `MS_TOKEN` (передается через GitHub Secrets в env `MS_TOKEN`).
*   **Внешний вызов (Source)**: Запросы `HTTP GET` (через Playwright и HTTP-клиенты `TikTokApi`) на сервера TikTok (например, `v16-web.tiktok.com`, `tiktokcdn.com`). Ответ приходит в виде JSON-объектов (структуру которых можно увидеть в `tiktok_example_data.json`).
*   **Преобразование (Mapping)**:
    *   `video.id` -> `<id>` в RSS.
    *   `createTime` -> `<published>` и `<updated>`.
    *   `desc` -> `<title>`.
    *   `cover` (URL) -> Playwright скачивает -> Локальный путь `/thumbnails/{user}/screenshot_{id}.jpg` -> Публичный URL `ghRawURL + screenshotsubpath` -> `<content> <img src=...> </content>`.
*   **Выходные данные (Memory / Sink)**:
    *   Диск (репозиторий): Формируются/обновляются файлы в `/rss/{user}.xml` и изображения в `/thumbnails/{user}/*.jpg`.
    *   Статический сервер: GitHub Pages отдает сохраненные `.xml` файлы любым RSS-клиентам (например, Feedly).

---

## 4. Схема данных (PostgreSQL Proposal)

Для переноса этой логики в ваш проект с использованием PostgreSQL (согласно вашим предыдущим проектам с Airflow/Celery, подходом 'ProductHunt.com-Scrapers' с JSONB и 'ON CONFLICT' upsert стратегией), предлагается следующая реляционная структура. Она поддерживает мульти-арендность, хранение сырых метаданных (JSONB) для гибкости и связь объектов в стиле подписок.

```sql
-- Таблица отслеживаемых пользователей/подписок (замена subscriptions.csv)
CREATE TABLE tiktok_users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    tiktok_user_id VARCHAR(255), -- Внутренний ID TikTok (например, 6799344441436275718)
    nickname VARCHAR(255),
    avatar_url TEXT,
    last_scraped_at TIMESTAMP WITH TIME ZONE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Таблица для хранения спарсенных видео
CREATE TABLE tiktok_videos (
    id SERIAL PRIMARY KEY,
    video_id VARCHAR(255) UNIQUE NOT NULL, -- TikTok Video ID (например, 7019325673739177221)
    user_id INTEGER REFERENCES tiktok_users(id) ON DELETE CASCADE,

    title TEXT,                            -- Текст/описание (desc)
    tiktok_created_at TIMESTAMP WITH TIME ZONE, -- Дата создания видео в TikTok (createTime)

    -- Медиа ссылки (публичные URL вашего сервиса)
    cover_image_url TEXT,                  -- Ваш локальный/S3 URL загруженной обложки (замена Playwright скриншотам)
    video_url TEXT,                        -- Если планируете качать само видео (вдохновение 'podsync')

    -- Хранение сырой информации на случай изменений API или необходимости дополнительных данных
    raw_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Индексы для быстрой выборки и генерации RSS
CREATE INDEX idx_tiktok_videos_user_id ON tiktok_videos(user_id);
CREATE INDEX idx_tiktok_videos_tiktok_created_at ON tiktok_videos(tiktok_created_at DESC);
CREATE INDEX idx_tiktok_videos_metadata ON tiktok_videos USING GIN (raw_metadata);

-- Пример UPSERT (ON CONFLICT) для вставки спарсенных данных:
/*
INSERT INTO tiktok_videos (video_id, user_id, title, tiktok_created_at, raw_metadata)
VALUES (
    '7019325673739177221',
    (SELECT id FROM tiktok_users WHERE username = 'iamtabithabrown'),
    'Happy Friday!! Let it go...',
    to_timestamp(1634314116),
    '{"id": "7019325673739177221", "stats": {"playCount": 22700}}'::jsonb
)
ON CONFLICT (video_id) DO UPDATE SET
    title = EXCLUDED.title,
    raw_metadata = EXCLUDED.raw_metadata,
    updated_at = NOW();
*/
```

### Интеграция с вашей архитектурой
*   **Сбор данных**: Асинхронные воркеры (например, Celery или Airflow) считывают `tiktok_users`, берут прокси, `msToken` и парсят TikTok.
*   **Обработка медиа**: Скачивают обложки (и, возможно, видео), сохраняют в облачное хранилище (например, S3), и записывают публичные HTTPS URLs с правильными заголовками CORS в колонки `cover_image_url`. Использование локальных путей не рекомендуется для возможности отображения во frontend-приложениях.
*   **Генерация RSS/XML (Backend)**: Ваш API (например, FastAPI или Django) динамически генерирует XML на лету (или кэширует с помощью Redis), извлекая данные из таблицы `tiktok_videos` с сортировкой по `tiktok_created_at DESC`. Это эффективнее, чем статические "flat files", при масштабировании.