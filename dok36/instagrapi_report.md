# Анализ репозитория: instagrapi

## 1. Техническая реализация и архитектура
Библиотека `instagrapi` — это полнофункциональный клиент для приватного (мобильного) и публичного (веб) API Instagram, реализованный на Python. Архитектура построена на использовании объектно-ориентированного подхода с применением миксинов (mixins). Основной класс `Client` (определен в `__init__.py`) собирается из множества узкоспециализированных миксинов (например, `AuthMixin`, `PhotoMixin`, `MediaMixin`, `CommentMixin`, `DirectMixin`). Это позволяет структурировать огромный объем функционала API.

**Основные компоненты:**
*   **Сетевой слой:** Используется библиотека `requests` для обработки HTTP/HTTPS запросов. Приватный и публичный API разделены (в миксинах `private.py` и `public.py`). Сохраняются сессии (cookies) для аутентифицированных запросов.
*   **Генерация параметров устройства:** Instagram требует, чтобы запросы выглядели так, будто они отправлены с реального мобильного устройства. Конфигурация включает версию Android, разрешение экрана, производителя, модель процессора и версию приложения (см. `config.py`, `USER_AGENT_BASE`, `DEVICE_SETTINGS`).
*   **Криптография (шифрование пароля):** При логине пароль не отправляется в открытом виде. Он шифруется алгоритмом, требуемым серверами Instagram, с помощью RSA и AES-GCM шифрования, используя публичный ключ, предварительно полученный от сервера (`password.py`). Формируется специфическая строка с меткой `#PWD_INSTAGRAM:4:<timestamp>:<base64_payload>`.
*   **Подписи запросов:** Для приватного API (POST-запросы) формируется "подписанное тело" запроса (`signed_body`). В `instagrapi` `generate_signature` добавляет префикс `SIGNATURE.` к urlencoded данным, симулируя подпись приложения (в новых версиях API этот механизм упрощен или симулируется, так как ключи `ig_sig_key_version` больше не требуются/устарели для многих эндпоинтов в данной реализации).

## 2. Функции и логическая работа (Как работают пункты и Схема данных)

**Позволяет делать всё, что делает официальное приложение:**
1.  **Формирование HTTP-запросов и сессии (Cookies):**
    *   **User-Agent:** Динамически формируется на базе параметров устройства (Android version, DPI, Resolution, Manufacturer, Model, App Version). `User-Agent` добавляется во все заголовки.
    *   **Сессии и Cookie:** `requests.Session()` управляет куками. Ключевые куки, такие как `sessionid`, `csrftoken` и `ds_user_id` сохраняются после логина. `AuthMixin` умеет выгружать их в словарь или файл настроек и загружать обратно для восстановления сессии без повторного логина.
    *   **UUID и токены:** Генерируются уникальные идентификаторы устройства (UUID, android_device_id) для каждого логина, чтобы сымитировать уникальную установку приложения.
2.  **Публикация фото/видео (`PhotoMixin`, `VideoMixin`):**
    *   Сначала медиафайл загружается на сервер через отдельный процесс (rupload), который возвращает `upload_id`. В этот момент отправляются бинарные данные, часто разбитые на чанки.
    *   После загрузки отправляется POST запрос "configure" (например, `media/configure/`) с указанием полученного `upload_id`, подписи `caption` (текст поста), `usertags` (отметки пользователей) и `location` (геолокация).
3.  **Лайки (`MediaMixin`):**
    *   Отправляется POST запрос на `media/{media_id}/like/` (или `/unlike/`). В теле запроса передаются данные контекста (из какой ленты был лайк `feed_position`, `inventory_source`).
4.  **Комментарии (`CommentMixin`):**
    *   Отправляется POST запрос на `media/{media_id}/comment/`. В теле запроса (отформатированном в `signed_body`) передается `comment_text` и уникальный `idempotence_token` (UUID) для предотвращения дублирования комментариев.
5.  **Отправка сообщений (`DirectMixin`):**
    *   POST запрос на эндпоинт отправки директа (например, `/direct_v2/threads/broadcast/text/`). Указываются `thread_ids` или `user_ids`, текст сообщения `text` и `client_context` (уникальный ID, генерируемый клиентом).
6.  **Сбор данных о профилях (`UserMixin`):**
    *   Сбор выполняется как через приватный мобильный API (`users/{user_id}/info/`), так и через публичный GraphQL API (`GraphQL` запрос для `web_profile_info`), собирая информацию о пользователе: фолловеры, подписки, био, аватарки. Результаты маппятся во внутренние Pydantic-подобные структуры (модели из `types.py` такие как `UserShort`, `User`).

## 3. Информационный обмен, память и потоки данных: Моделирование в PostgreSQL

Чтобы реализовать подобный функционал (сбор данных, управление сессиями, логгирование действий) в вашем проекте на PostgreSQL, потребуется следующая архитектура базы данных.

### Схема данных (PostgreSQL)

**1. Управление аккаунтами (Ботами) и Сессиями:**
Вам нужно хранить состояние каждого аккаунта, которым управляет система, включая конфигурацию устройства (для стабильности User-Agent) и куки.

```sql
CREATE TABLE ig_accounts (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password_encrypted TEXT, -- Зашифрованный пароль
    proxy_url TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Сохранение состояния устройства для каждого аккаунта,
-- чтобы User-Agent был консистентным
CREATE TABLE ig_device_settings (
    account_id INTEGER PRIMARY KEY REFERENCES ig_accounts(id) ON DELETE CASCADE,
    app_version VARCHAR(50),
    android_version VARCHAR(10),
    android_release VARCHAR(10),
    dpi VARCHAR(20),
    resolution VARCHAR(20),
    manufacturer VARCHAR(50),
    model VARCHAR(50),
    device VARCHAR(50),
    cpu VARCHAR(20),
    uuid UUID,                 -- Генерируется один раз
    android_device_id VARCHAR(50)
);

-- Хранение сессий, чтобы не логиниться каждый раз
CREATE TABLE ig_sessions (
    account_id INTEGER PRIMARY KEY REFERENCES ig_accounts(id) ON DELETE CASCADE,
    sessionid VARCHAR(255),
    csrftoken VARCHAR(255),
    ds_user_id VARCHAR(100),
    mid VARCHAR(255),          -- Machine ID (из кук)
    cookies_json JSONB,        -- Дамп всех кук (dict_from_cookiejar)
    last_login_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

**2. Сбор данных о профилях (Профили и Медиа):**
Результаты скрапинга через приватный/публичный API.

```sql
-- Таблица пользователей (собранные данные)
CREATE TABLE target_users (
    pk BIGINT PRIMARY KEY,     -- Уникальный ID Instagram (user_id)
    username VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(255),
    is_private BOOLEAN,
    profile_pic_url TEXT,
    biography TEXT,
    follower_count INTEGER,
    following_count INTEGER,
    last_scraped_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Граф связей: Кто на кого подписан (для GraphRAG)
CREATE TABLE user_relations (
    follower_pk BIGINT REFERENCES target_users(pk),
    followed_pk BIGINT REFERENCES target_users(pk),
    relation_type VARCHAR(50), -- 'following', 'follower'
    scraped_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_pk, followed_pk)
);

-- Собранные посты (Медиа)
CREATE TABLE target_media (
    media_pk VARCHAR(255) PRIMARY KEY, -- id из Instagram (например, 234123123_45345)
    user_pk BIGINT REFERENCES target_users(pk),
    media_type INTEGER,                -- 1: фото, 2: видео, 8: карусель
    code VARCHAR(100),                 -- Короткий код (shortcode) url поста
    caption TEXT,
    like_count INTEGER,
    comment_count INTEGER,
    taken_at TIMESTAMP WITH TIME ZONE,
    raw_data JSONB                     -- Полный дамп JSON для будущей аналитики или эмбеддингов
);
```

**3. Логирование действий (Лайки, Комментарии, Директ):**
Очередь задач или лог выполненных действий.

```sql
CREATE TABLE action_logs (
    id SERIAL PRIMARY KEY,
    account_id INTEGER REFERENCES ig_accounts(id),
    action_type VARCHAR(50),      -- 'LIKE', 'COMMENT', 'DIRECT_MESSAGE', 'UPLOAD_PHOTO'
    target_id VARCHAR(255),       -- media_pk или user_pk
    payload JSONB,                -- {"text": "привет"} для коммента/директа
    status VARCHAR(50),           -- 'PENDING', 'SUCCESS', 'FAILED'
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    executed_at TIMESTAMP WITH TIME ZONE
);
```

### Принцип работы в связке с Postgres:
1. Вы запускаете фоновый воркер (на Python). Он читает активные аккаунты из `ig_accounts`.
2. Загружает параметры устройства из `ig_device_settings` и сессию из `ig_sessions` (JSONB) и передает их в `Client` (`instagrapi`).
3. Для задач сбора: воркер вызывает метод (например, `user_info`), получает Pydantic объект, преобразует его и делает `INSERT/UPDATE` в `target_users` и `target_media`. Поля вроде списка URL картинок можно сохранить в `JSONB` колонке `raw_data`.
4. Для задач постинга/лайков: система читает задачи из `action_logs` где status = `PENDING`, вызывает соответствующий метод (`media_like`, `photo_upload`), и по результатам ответа (`result["status"] == "ok"`) обновляет лог на `SUCCESS` или сохраняет ошибку.
