# Отчет: Анализ YTScribe и youtube-transcript-api (youtube_analysis)
Источник данных:
*   YTScribe: [https://github.com/dparedesi/YTScribe](https://github.com/dparedesi/YTScribe)
*   youtube-transcript-api: [https://github.com/jdepoix/youtube-transcript-api](https://github.com/jdepoix/youtube-transcript-api)

## 1. Извлечение метаданных (YTScribe)
В `YTScribe` в модуле `extractor.py` (используется для извлечения видео с канала) изначально задумывался парсинг скрытых JSON-объектов `ytInitialPlayerResponse` и `ytInitialData`. Однако в текущей версии репозитория этот подход заменен на использование внешних библиотек **yt-dlp** (предпочтительно) или **pytube** (в качестве fallback).
Библиотека просто извлекает плоский список (`extract_flat=True`) через yt-dlp, обходя необходимость прямого парсинга HTML, что является более надежным решением, так как yt-dlp регулярно обновляется под изменения YouTube.

**Пример использования `yt-dlp` для извлечения (из `YTScribe`):**
```python
ydl_opts = {
    "quiet": True,
    "no_warnings": True,
    "extract_flat": True,
    "playlistend": max_videos,
}
with yt_dlp.YoutubeDL(ydl_opts) as ydl:
    info = ydl.extract_info(url, download=False)
    # Далее парсится info["entries"]
```

## 2. Извлечение субтитров без квот (youtube-transcript-api)
Получение субтитров происходит не через официальный Data API, а через скрытый внутренний API YouTube (InnerTube API), который используется самим веб-клиентом YouTube.

**Алгоритм действий (из `_transcripts.py`):**
1.  **Запрос к странице видео:** Делается обычный GET запрос к `https://www.youtube.com/watch?v={video_id}`.
2.  **Обход согласия (Consent):** Если YouTube перенаправляет на страницу `consent.youtube.com`, библиотека извлекает токен `v`, создает cookie `CONSENT=YES+<token>` и повторяет запрос.
3.  **Извлечение `INNERTUBE_API_KEY`:** В HTML странице с помощью регулярного выражения `r'"INNERTUBE_API_KEY":\s*"([a-zA-Z0-9_-]+)"'` извлекается ключ API.
4.  **Запрос к InnerTube API:** Делается POST запрос к `https://www.youtube.com/youtubei/v1/player?key={api_key}`. В тело запроса (`json`) передается:
    ```json
    {
        "context": {
            "client": {
                "clientName": "WEB",
                "clientVersion": "2.20240909.00.00"
            }
        },
        "videoId": "ID_ВИДЕО"
    }
    ```
5.  **Извлечение ссылок на субтитры:** Из ответа извлекается объект `captions.playerCaptionsTracklistRenderer.captionTracks`. Этот массив содержит объекты треков с ключами `languageCode`, `baseUrl`, `vssId` и `kind`. Если трек сгенерирован автоматически, у него присутствует `"kind": "asr"`.

## 3. Логика автоперевода субтитров
В InnerTube API субтитры возвращаются в формате XML. Библиотека позволяет легко переводить субтитры "на лету", используя встроенную функциональность YouTube.

**Реализация:**
К URL-адресу базовых субтитров (`baseUrl`), полученному из объекта `captionTracks`, просто добавляется параметр `&tlang={language_code}`, например, `&tlang=ru`.

```python
# Фрагмент из youtube_transcript_api/_transcripts.py
def translate(self, language_code: str) -> "Transcript":
    # ... проверки ...
    return Transcript(
        self._http_client,
        self.video_id,
        "{url}&tlang={language_code}".format(
            url=self._url, language_code=language_code
        ),
        # ...
    )
```
После выполнения GET-запроса по этому URL, YouTube отдаст автоматически переведенный XML файл с субтитрами.

## 4. Механизм Rate Limiting и защита от бана
Если делать слишком много запросов, YouTube начинает возвращать HTTP 429 или требовать прохождение ReCAPTCHA.

В `youtube-transcript-api` (в модуле `_api.py` и `_transcripts.py`) реализовано следующее:
1.  **Обнаружение блокировки:**
    *   HTTP статус 429 вызывает исключение `IpBlocked`.
    *   Наличие `class="g-recaptcha"` в HTML вызывает исключение `IpBlocked`.
    *   Статус `LOGIN_REQUIRED` с причиной `Sign in to confirm you’re not a bot` вызывает `RequestBlocked`.
2.  **Повторы (Retries) с Proxy:**
    Используется класс `ProxyConfig`. Если настроено использование прокси (через параметры в библиотеке `requests` — `proxies` в `Session`), класс `TranscriptListFetcher` делает попытку повторного выполнения запроса:
    ```python
    try:
        # fetch_captions...
    except RequestBlocked as exception:
        if try_number + 1 < retries:
            return self._fetch_captions_json(video_id, try_number=try_number + 1)
        raise exception
    ```
3.  **Использование сессий:** Объект `requests.Session()` переиспользуется для поддержания консистентного состояния. Опционально можно передавать флаг `Connection: close`, если нужно заставить прокси каждый раз открывать новое соединение. Дополнительно настраивается `urllib3.Retry` с параметром `status_forcelist=[429]`, который автоматически делает паузу и повторяет запрос на уровне самого HTTP-клиента.

## Итог для Cherry-picking
В своем проекте можно реализовать обертку, которая сначала делает GET запрос к видео, парсит регуляркой `INNERTUBE_API_KEY`, затем делает POST на `/youtubei/v1/player`, извлекает `baseUrl` из `captions`, парсит XML (`<text start="1.5" dur="3.0">Text</text>`) и, при необходимости перевода, подставляет `&tlang=`.
