# Подробный технический анализ Podsync: конвертация видео в подкасты

Проект Podsync преобразует видео и аудио с платформ (YouTube, Vimeo и др.) в стандартные RSS-ленты для подкастов. Этот документ содержит подробный анализ компонентов `pkg/builder/`, `pkg/ytdl/`, слоя базы данных и генерации RSS, а также предлагает адаптацию архитектуры и схемы данных для PostgreSQL и Python.

---

## Часть 1. Техническая реализация и архитектура

### 1.1 `pkg/builder/` (Классы-сборщики)
В пакете `pkg/builder/` реализован паттерн "Строитель" (Builder) для различных провайдеров контента. Главный интерфейс `Builder` имеет один основной метод `Build(ctx, cfg)`, который принимает конфигурацию фида и возвращает модель `model.Feed`.
- Архитектура основана на абстракции: каждый провайдер (YouTube, Vimeo, SoundCloud) реализует свой Builder. Для YouTube это `YouTubeBuilder`, который использует официальный клиент `google.golang.org/api/youtube/v3`.
- Builder отвечает за:
  1. **Извлечение метаданных канала/плейлиста:** получение информации о названии, описании, авторе и обложке фида.
  2. **Обход пагинации (queryItems):** получение списка эпизодов с учетом сортировки и лимита (PageSize).
  3. **Маппинг данных:** преобразование проприетарного формата провайдера в унифицированную структуру `model.Episode`. Для YouTube используются запросы `Videos.List` для получения длительности и расширенного описания.
  4. **Оценка размера:** поскольку точный размер файла до скачивания неизвестен, Builder рассчитывает примерный размер (getSize) на основе длительности и выбранного качества/формата битрейта.

### 1.2 `pkg/ytdl/` (Обертка над youtube-dl)
Пакет `pkg/ytdl/` инкапсулирует вызовы бинарного файла `youtube-dl` (или его форков вроде yt-dlp) через `os/exec`. Основной структурой является `YoutubeDl`.
- **Инициализация:** проверяется наличие бинарника `youtube-dl` и зависимостей (`ffmpeg` / `avconv`).
- **Самообновление (SelfUpdate):** реализован фоновый горутиной автоматический апдейт бинарника `youtube-dl` каждые 24 часа. Доступ к выполнению команд защищен `sync.Mutex` (`updateLock`), чтобы предотвратить запуск загрузок во время обновления.
- **Методы загрузки:**
  - `PlaylistMetadata`: извлекает метаданные плейлиста через `youtube-dl -J --playlist-items 0`.
  - `Download`: создает временную директорию (`os.MkdirTemp`), формирует аргументы (формат, качество, извлечение аудио через ffmpeg) и скачивает эпизод. Возвращает `io.ReadCloser` на скачанный файл и автоматически очищает временную директорию при закрытии или ошибке. Формирование аргументов (`buildArgs`) строго зависит от настроек фида (Video/Audio, High/Low quality).

### 1.3 База данных (BadgerDB) и Генерация RSS (pkg/feed)
- **BadgerDB (`pkg/db/`):** используется встроенная Key-Value база BadgerDB. Данные сериализуются в JSON.
  - Структура ключей иерархична: фиды хранятся по ключам `feed/{feedID}`, а эпизоды — `episode/{feedID}/{episodeID}`. Это позволяет эффективно извлекать все эпизоды конкретного фида через префикс-итератор (`badger.DefaultIteratorOptions.Prefix`).
  - При обновлении фида (`AddFeed`) новые эпизоды добавляются к существующим, старые не перезаписываются, что важно для сохранения состояния уже скачанных файлов.
- **XML RSS Генерация (`pkg/feed/xml.go`):** XML формируется "на лету" из данных BadgerDB с использованием пакета `github.com/eduncan911/podcast`.
  - Эпизоды фильтруются: в RSS попадают только те, которые имеют статус `EpisodeDownloaded`.
  - Ссылки генерируются как комбинация `hostname/feedID/episodeName`, где `episodeName` строится по шаблону (например, `id.ext`).

---

## Часть 2. Функции и логическая работа

### 2.1 Логика добавления и обновления видео
1. **Парсинг конфигурации:** При старте система читает `config.toml`, где определены URL-адреса, ключи API, форматы и расписание обновлений.
2. **Получение новых эпизодов (Builder):** По расписанию фоновый воркер обращается к `Builder` для каждого фида. Builder делает API-запрос к провайдеру (например, YouTube) и возвращает объект `model.Feed` с массивом новых/последних `model.Episode`. Эти эпизоды изначально получают статус `EpisodeNew`.
3. **Запись в БД:** Полученный `Feed` и его `Episodes` сохраняются в BadgerDB. Если эпизод уже существует, он игнорируется (чтобы не затереть статус скачанного).
4. **Загрузка (Downloader):** Отдельный процесс/горутина сканирует БД на наличие эпизодов со статусом `EpisodeNew` или `EpisodeError`.
5. **Вызов ytdl:** Для новых эпизодов вызывается `ytdl.Download`. `youtube-dl` скачивает файл и, при необходимости, `ffmpeg` извлекает аудио.
6. **Сохранение медиа:** После успешной загрузки файл перемещается в целевое хранилище (локальный диск или S3).
7. **Обновление статуса:** В БД статус эпизода меняется на `EpisodeDownloaded`.
8. **Обслуживание (Web-сервер):** При GET-запросе по URL фида (например, `/youtube-channel.xml`), веб-сервер извлекает из BadgerDB `Feed` и все эпизоды со статусом `EpisodeDownloaded`, сортирует их по дате публикации (по убыванию) и генерирует XML, подставляя локальные URL/S3 ссылки в теги `<enclosure>`.

---

## Часть 3. Информационный обмен, память и потоки данных

### 3.1 Модель данных Go / BadgerDB

| Структура | Описание |
| :--- | :--- |
| `model.Feed` | Описывает канал подкаста. Поля: ID, Provider, LinkType (channel, user, playlist), Format, Quality, CoverArt, Title, Description. Сериализуется в JSON по ключу `feed/{id}`. |
| `model.Episode` | Описывает один выпуск. Поля: ID, Title, Description, Thumbnail, Duration, VideoURL, PubDate, Size, Status. Сериализуется в JSON по ключу `episode/{feed_id}/{episode_id}`. |

**Жизненный цикл `Status`:**
- `EpisodeNew` -> API вернул новое видео.
- `EpisodeDownloaded` -> Файл скачан, доступен в RSS.
- `EpisodeError` -> Ошибка скачивания (ytdl HTTP 429 и т.д.).
- `EpisodeCleaned` -> Файл удален с диска (для ротации).

### 3.2 Предлагаемая схема данных для PostgreSQL (и Python/SQLAlchemy)

Для реализации на Python (с использованием SQLAlchemy/AsyncPG и PostgreSQL) предлагается нормализованная реляционная схема данных. В качестве ORM можно использовать SQLAlchemy.

#### 1. Таблица `feeds`
```sql
CREATE TABLE feeds (
    id VARCHAR PRIMARY KEY,                  -- Уникальный идентификатор (хеш URL или ID провайдера)
    provider VARCHAR(50) NOT NULL,           -- 'youtube', 'vimeo', 'soundcloud'
    link_type VARCHAR(50) NOT NULL,          -- 'channel', 'playlist', 'user'
    item_id VARCHAR NOT NULL,                -- Оригинальный ID провайдера (UCxxxxxx)
    title VARCHAR,
    description TEXT,
    author VARCHAR,
    cover_art VARCHAR,                       -- URL обложки
    format VARCHAR(20) NOT NULL,             -- 'audio', 'video'
    quality VARCHAR(20) NOT NULL,            -- 'high', 'low'
    page_size INTEGER DEFAULT 50,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_access TIMESTAMP WITH TIME ZONE
);
```

#### 2. Таблица `episodes`
```sql
CREATE TYPE episode_status AS ENUM ('new', 'downloaded', 'error', 'cleaned');

CREATE TABLE episodes (
    id VARCHAR PRIMARY KEY,                  -- Оригинальный ID видео (напр. vdQk4x...)
    feed_id VARCHAR NOT NULL REFERENCES feeds(id) ON DELETE CASCADE,
    title VARCHAR NOT NULL,
    description TEXT,
    thumbnail VARCHAR,
    duration BIGINT DEFAULT 0,               -- Длительность в секундах
    video_url VARCHAR NOT NULL,              -- Оригинальный URL на YouTube/Vimeo
    pub_date TIMESTAMP WITH TIME ZONE,
    size BIGINT DEFAULT 0,                   -- Размер в байтах
    status episode_status DEFAULT 'new',
    file_path VARCHAR,                       -- Локальный путь или S3 ключ скачанного файла
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    CONSTRAINT unique_feed_episode UNIQUE(feed_id, id)
);

CREATE INDEX idx_episodes_feed_status ON episodes(feed_id, status);
CREATE INDEX idx_episodes_pub_date ON episodes(pub_date DESC);
```

#### 3. Архитектура для Python (Celery / Asyncio)
1. **Builders (Python):** Замените `youtube-dl` на современный `yt-dlp` (библиотека Python). Используйте `google-api-python-client` для взаимодействия с YouTube Data API, асинхронный `httpx` для сетевых запросов.
2. **Оркестрация:** Используйте Celery или Asyncio/APScheduler для запуска воркеров, которые будут периодически парсить новые видео через API и записывать их в таблицу `episodes` со статусом `new`.
3. **Очередь скачивания:** Воркер Celery берет записи со статусом `new`, вызывает `yt-dlp` (`ydl.download`), конвертирует через `ffmpeg-python`, загружает на S3 (через `boto3`) и меняет статус в PostgreSQL на `downloaded`.
4. **Web-сервер:** FastAPI или Aiohttp маршрут `/feed/{feed_id}.xml`. Делает `SELECT * FROM feeds` и `SELECT * FROM episodes WHERE feed_id = X AND status = 'downloaded' ORDER BY pub_date DESC`. Формирует XML RSS с использованием библиотеки (например, `feedgen`) "на лету" и отдает клиенту.
