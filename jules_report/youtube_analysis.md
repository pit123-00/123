# Анализ извлечения данных YouTube

Ссылки на репозитории:
* YTScribe: https://github.com/dparedesi/YTScribe
* youtube-transcript-api: https://github.com/jdepoix/youtube-transcript-api

## 1. Техническая реализация и архитектура
Библиотека YTScribe использует подход с fallback-механизмами для извлечения метаданных канала (в `extractor.py`). В качестве основного инструмента используется `yt-dlp` (с отключением загрузки видео), а если он недоступен, применяется `pytube`.
Получение субтитров делегируется базовой библиотеке `youtube-transcript-api`. Она реализует прямой парсинг скрытых конфигурационных объектов (таких как `ytInitialPlayerResponse` и `ytInitialData`) непосредственно из исходного HTML страницы видео. Это позволяет обойти квоты официального YouTube Data API, так как запросы эмулируют обычного пользователя, загружающего страницу.

Архитектура работы с сетью в `youtube-transcript-api` (файл `_api.py`) построена на основе объекта `requests.Session` для переиспользования соединений и потенциального использования cookie, с поддержкой адаптеров `urllib3.Retry` для автоматических повторных запросов (Rate Limiting) при получении HTTP статусов 429 (Too Many Requests).

## 2. Функции и логика работы
**Парсинг скрытых JSON-объектов (`ytInitialPlayerResponse`, `ytInitialData`)**:
При запросе HTML-кода страницы видео (`_fetch_html`), YouTube возвращает скрипты, в которых встроены объекты `ytInitialPlayerResponse` (описывает параметры видео, доступные форматы и статусы воспроизведения) и `ytInitialData` (описывает UI и данные плейлиста/рекомендаций).
В `youtube-transcript-api` (в частности в `TranscriptListFetcher`) для получения ключа API InnerTube используется регулярное выражение (`"INNERTUBE_API_KEY":\s*"([a-zA-Z0-9_-]+)"`), после чего к этому внутреннему API отправляется POST запрос.
Ранее использовался прямой парсинг HTML-кода для нахождения `ytInitialPlayerResponse` (как видно из статических ассетов).

**Обращение к внутреннему InnerTube API для выгрузки XML субтитров и логика автоперевода**:
Полученный ключ используется для POST-запроса на `INNERTUBE_API_URL` (обычно `/youtubei/v1/player`). Ответ API содержит структуру `captions -> playerCaptionsTracklistRenderer -> captionTracks`, где перечислены доступные субтитры.
Для получения перевода на другой язык (логика автоперевода) в конец URL добавляется параметр `&tlang={language_code}`. Библиотека `youtube-transcript-api` позволяет найти сгенерированные либо загруженные пользователем субтитры по языковому коду (в классе `TranscriptList`), а функция `translate(language_code)` формирует новый URL для загрузки XML и применяет парсер `_TranscriptParser` для извлечения текста и временных меток.

**Механизм Rate Limiting (искусственные задержки)**:
Для предотвращения бана IP адреса (ошибка `IpBlocked` или `RequestBlocked` с рекапчей), `youtube-transcript-api` использует:
1. Систему проксирования (`ProxyConfig`), которая позволяет задать разные прокси.
2. Обработку статуса 429 в `requests.adapters.HTTPAdapter(max_retries=retry_config)`.
3. Рекурсивный повтор с ограничением `retries_when_blocked` в функции `_fetch_captions_json`.

## 3. Информационный обмен, память и потоки данных
**Входные данные**: URL канала или ID видео. При массовой загрузке, список ID или URL.
**Поток обработки в YTScribe**:
1. Передача URL канала в `ChannelExtractor.extract_videos`.
2. Извлечение списка видео (`VideoMetadata`) через `yt-dlp` (или `pytube`).
3. Для каждого видео: инициализация `youtube-transcript-api.YouTubeTranscriptApi`.

**Поток обработки в youtube-transcript-api**:
1. Формируется HTTP GET запрос страницы видео.
2. Сервер возвращает HTML, содержащий `INNERTUBE_API_KEY` и/или рекапчу.
3. Формируется POST запрос к InnerTube API (`/youtubei/v1/player` или аналогичный) с JSON-телом, содержащим `context` и `videoId`.
4. Сервер возвращает JSON с блоком `captions`.
5. Парсинг `captionTracks` и получение базового URL `baseUrl` для XML файла субтитров. Если нужен перевод, добавляется `&tlang=`.
6. Выполнение GET запроса по полученному URL для загрузки XML.
7. Парсинг XML (`ElementTree.fromstring`), очистка HTML тегов (`unescape`) и формирование массива словарей/объектов (`FetchedTranscriptSnippet`) с `text`, `start`, `duration`.
**Выходные данные**: Список сегментов текста (субтитров) с временными метками для сохранения, например, в JSONL, CSV или для передачи агентам (LLM). Память здесь используется кратковременная - скачанный XML парсится в оперативной памяти и сразу трансформируется в результат, после чего сборщик мусора очищает память.
