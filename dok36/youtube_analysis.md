# Анализ архитектуры `youtube-transcript-api` и `YTScribe` и схема данных для PostgreSQL

## 1. Техническая реализация и архитектура

Обе библиотеки предназначены для извлечения метаданных и субтитров (транскриптов) с YouTube без необходимости использования официального YouTube Data API с квотами.

- **youtube-transcript-api**: Это узконаправленная, низкоуровневая библиотека для получения транскриптов. Она использует веб-скрейпинг статического HTML видео-страницы (`watch?v=...`) и непубличные эндпоинты (InnerTube API).
  - Для работы с сетью используется `requests.Session`, что обеспечивает поддержку прокси-серверов (`ProxyConfig`) и кастомных заголовков.
  - Реализованы механизмы обхода и определения причин недоступности видео (возрастные ограничения `AGE_RESTRICTED`, капчи `BOT_DETECTED`, недоступность видео `VIDEO_UNAVAILABLE`).
  - Данные субтитров скачиваются в формате XML, который затем преобразуется во внутренние структуры Python.

- **YTScribe**: Выступает в роли высокоуровневой обертки-оркестратора над `youtube-transcript-api` и использует `yt-dlp` (или `pytube` в качестве фолбэка) для получения метаданных канала и видео.
  - Проект реализует логику пакетной обработки (batch processing) с использованием локальных CSV-файлов (`videos.csv`) в качестве базы данных для сохранения состояния задач (state memory).
  - Управление скачиванием снабжено механизмом rate limiting (искусственные задержки `delay`), чтобы снизить риск блокировки IP-адресов серверами YouTube.
  - Результаты сохраняются в файловую систему в формате Markdown с блоком YAML frontmatter, содержащим метаданные.

**Архитектурный подход для адаптации к PostgreSQL**:
Локальные файлы, CSV-таблицы и in-memory очереди должны быть заменены на транзакционные таблицы PostgreSQL. Состояния скачивания (`pending`, `success`, `error`) управляются прямо в БД. Механизм блокировок `SELECT ... FOR UPDATE SKIP LOCKED` позволит запускать асинхронные фоновые процессы (background workers), избегая конфликтов (race conditions) при параллельной обработке видео.

## 2. Функции и логические операции

- **Извлечение метаданных (YTScribe: `extractor.py`, `metadata.py`)**:
  - `ChannelExtractor` обращается к `yt-dlp` в режиме плоского извлечения (`extract_flat=True`) для быстрого получения списка `video_id` с канала без загрузки самих видео.
  - `extract_metadata_from_html` осуществляет парсинг регулярными выражениями объектов `ytInitialPlayerResponse` и `ytInitialData` из HTML-кода для получения: `title`, `author`, `duration`, `views`, `published_date` и `description`.

- **Загрузка и парсинг транскриптов (youtube-transcript-api: `_transcripts.py`)**:
  - `_fetch_video_html`: Получает HTML-код страницы. При необходимости автоматически генерирует cookie для обхода согласия на файлы cookie.
  - `_extract_innertube_api_key`: Ищет скрытый API ключ `INNERTUBE_API_KEY` в HTML.
  - `_fetch_innertube_data`: Делает POST-запрос к API и извлекает JSON с конфигурацией субтитров.
  - `TranscriptList.build`: Создает списки доступных языков, разделяя их на автоматически сгенерированные (`asr`) и созданные вручную. Поддерживается "на лету" перевод субтитров на другие языки (параметр `&tlang=`).
  - `_TranscriptParser.parse`: Переводит полученные XML-данные субтитров в список объектов-сниппетов `FetchedTranscriptSnippet` (текст, время начала, длительность).

- **Пакетная обработка (YTScribe: `batch.py`)**:
  - `download_from_csv`: Читает строки из CSV-файла. Для каждой записи проверяет статус `is_already_downloaded()`. Если статус не заполнен, запускает загрузку с задержкой, обновляет поля в памяти и сразу же перезаписывает изменения в CSV через `update_csv_status`. В случае получения ошибки `IPBlockedError` обработка немедленно останавливается для предотвращения бана.

## 3. Информационный обмен, память и потоки данных

**Потоки данных (Data Flows) в процессе работы:**
1. **Входные данные**: URL канала или список URL видео.
2. **Экстракция ID**: Нормализация URL и извлечение уникального 11-символьного ID (`video_id`), например: `dQw4w9WgXcQ`.
3. **Парсинг метаданных**: Отправка HTTP-запроса на YouTube $\rightarrow$ получение HTML $\rightarrow$ парсинг JSON объектов $\rightarrow$ создание датакласса `VideoMetadata`.
4. **Запрос списка субтитров**: Запрос манифеста субтитров через InnerTube $\rightarrow$ выбор предпочтительного языка (обычно приоритет у английского, созданного вручную) $\rightarrow$ создание датакласса `TranscriptMetadata`.
5. **Загрузка текста**: HTTP-запрос файла субтитров $\rightarrow$ парсинг XML $\rightarrow$ извлечение таймкодов и текста в `FetchedTranscriptSnippet` $\rightarrow$ конкатенация текста для полнотекстовой версии.
6. **Сохранение (Память)**: В исходном коде данные сериализуются в Markdown. При переносе на PostgreSQL данные будут конвертироваться в реляционную модель с использованием JSONB для гибких метаданных.

## Схема данных для PostgreSQL (PostgreSQL Schema Mapping)

Схема создана на базе Python-моделей из исходного кода (`VideoMetadata`, `TranscriptMetadata`, `FetchedTranscriptSnippet`). В ней учтены паттерны работы фоновых задач, а также возможность хранения фрагментов транскрипта как массива JSON-объектов.

```sql
-- Таблица каналов (опционально, для организации видео)
CREATE TABLE youtube_channels (
    channel_id VARCHAR(50) PRIMARY KEY,
    url TEXT UNIQUE NOT NULL,
    title VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_scraped_at TIMESTAMP WITH TIME ZONE
);

-- Таблица видео и метаданных (Основано на модели VideoMetadata)
CREATE TABLE youtube_videos (
    video_id VARCHAR(20) PRIMARY KEY, -- 11-символьный ID YouTube
    channel_id VARCHAR(50) REFERENCES youtube_channels(channel_id),
    url TEXT NOT NULL,
    title TEXT,
    author VARCHAR(255),
    description TEXT,
    duration_seconds INTEGER,
    duration_minutes NUMERIC(10, 2),
    view_count BIGINT,
    published_date DATE,
    keywords TEXT, -- Можно также использовать TEXT[] для массивов

    -- Оркестрация: Управление статусом загрузки (Основано на DownloadStatus)
    transcript_status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'success', 'skipped', 'error', 'ip_blocked'
    transcript_error_message TEXT,
    last_download_attempt TIMESTAMP WITH TIME ZONE,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Таблица транскриптов (Основано на TranscriptMetadata и FetchedTranscript)
CREATE TABLE youtube_transcripts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id VARCHAR(20) REFERENCES youtube_videos(video_id) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL, -- например 'en', 'ru'
    is_generated BOOLEAN, -- Сгенерированы автоматически
    is_translatable BOOLEAN, -- Доступен ли перевод

    -- Полный конкатенированный текст (для простого отображения)
    full_text TEXT,

    -- Хранение сырых сегментов (Основано на FetchedTranscriptSnippet)
    -- Структура объектов внутри массива: {"text": "Hello", "start": 1.5, "duration": 3.0}
    snippets JSONB,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- Ограничение уникальности на язык для конкретного видео
    UNIQUE (video_id, language_code)
);

-- Индекс для оптимизации работы фоновых воркеров, выбирающих задачи из очереди
CREATE INDEX idx_youtube_videos_status ON youtube_videos(transcript_status)
WHERE transcript_status = 'pending';

-- Индекс для полнотекстового поиска или GIN-индекс по массиву фрагментов JSONB
CREATE INDEX idx_transcripts_snippets ON youtube_transcripts USING GIN (snippets);
```
