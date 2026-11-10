# Отчет по подключению и автопостингу: YouTube

Файлы в кодовой базе `treg`:
- Настройки провайдера: `src/treg/oauth_providers.py` (`YOUTUBE`)
- Каталог эндпоинтов: `src/treg/catalog/youtube.yaml`, `src/treg/catalog/youtube.extended.yaml`
- Обработка квот и ресурсов: `src/treg/application/connect.py`

---

## 1. Как устроена авторизация YouTube в коде `treg`

### Точки входа и параметры
- **Auth URI**: `https://accounts.google.com/o/oauth2/v2/auth`
- **Token URI**: `https://oauth2.googleapis.com/token`
- **Base URL**: `https://youtube.googleapis.com`
- **Upload Base URL**: `https://youtube.googleapis.com` (загрузка видео идет на префикс `/upload/youtube/v3/videos`)
- **Протокол**: Google OAuth 2.0 (Authorization Code Grant с `access_type=offline` и `prompt=consent` для получения `refresh_token`).

### Градации доступа (Capabilities & Scopes)
В YouTube разница между уровнями доступа критична:
1. **`read`**:
   - `https://www.googleapis.com/auth/youtube.readonly` — чтение данных канала, плейлистов, статистики, просмотров.
2. **`post`** (для публикации видео):
   - Все из `read` + **`https://www.googleapis.com/auth/youtube.upload`**.
   - *Важный нюанс из кода*: скоуп `youtube.upload` позволяет **только загрузить** видео. Если загрузить видео с этим скоупом, приложение **не сможет отредактировать метаданные или удалить** это видео позже.
3. **`manage`** (полное управление):
   - Все из `post` + `https://www.googleapis.com/auth/youtube` + `https://www.googleapis.com/auth/youtube.force-ssl`. Дает полные права на редактирование, модерацию комментариев, стримы и удаление видео.

---

## 2. Экономика квот Google Data API (Quota Units)

В коде `youtube.yaml` задокументирована жесткая система квот YouTube:
> Google Cloud выдает бесплатно **10 000 quota units в день на проект**:
> - Вызов `search.list` стоит **100 units** (100 поисковых запросов полностью сжигают дневную квоту!).
> - Вызов `playlistItems.list` стоит всего **1 unit** (чтение плейлиста "Uploads" канала в 100 раз дешевле поиска).
> - **Вызов `videos.insert` (загрузка видео) стоит 1 600 units!**
> Это означает, что один проект Google Cloud может загрузить максимум **6 видео в день** на стандартной квоте на все подключенные каналы вместе взятые!

### Ограничение неаудированных проектов Google (Audit Gate):
> Пока проект Google Cloud не прошел официальный аудит соответствия YouTube API Compliance Audit, вызов `videos.insert` завершается успешно (HTTP 200), **но загруженное видео принудительно блокируется в статусе `private` (приватное)**, независимо от того, какое значение передано в поле `privacyStatus`.

---

## 3. Механизм загрузки видео: Resumable Upload

Загрузка видео на YouTube происходит в два этапа по протоколу возобновляемой загрузки (Resumable Upload):

### Этап 1: Инициализация сессии загрузки
- **HTTP Метод и URL**:
  `POST https://youtube.googleapis.com/upload/youtube/v3/videos?uploadType=resumable&part=snippet,status,id`
- **Заголовки**:
  - `Authorization: Bearer {access_token}`
  - `Content-Type: application/json; charset=UTF-8`
  - `X-Upload-Content-Type: video/mp4`
  - `X-Upload-Content-Length: {file_size_in_bytes}`
- **Тело запроса (Метаданные видео)**:
  ```json
  {
    "snippet": {
      "title": "Обзор архитектуры проекта",
      "description": "Подробное техническое описание системы автопостинга.",
      "tags": ["разработка", "python", "api"],
      "categoryId": "28"
    },
    "status": {
      "privacyStatus": "public",
      "selfDeclaredMadeForKids": false
    }
  }
  ```
  *Критично: Поле `selfDeclaredMadeForKids` (сделано ли видео для детей) юридически обязательно (требование COPPA) — без него YouTube вернет ошибку.*

- **Ответ от YouTube**:
  Возвращается `HTTP 200 OK` с пустым телом и заголовком **`Location`**, содержащим уникальный URL сессии загрузки:
  `Location: https://youtube.googleapis.com/upload/youtube/v3/videos?upload_id=sA45gh...`

### Этап 2: Передача бинарных байтов видео (`PUT`)
- **HTTP Метод и URL**: `PUT {Location_URL}`
- **Заголовки**:
  - `Content-Length: {file_size}`
  - `Content-Type: video/mp4`
- В теле запроса передаются чистые байты файла. (Для очень больших файлов можно передавать чанками через заголовок `Content-Range: bytes 0-10485759/52428800`).
- **Финальный ответ**:
  JSON-объект созданного видео, содержащий `id`:
  `{"id": "dQw4w9WgXcQ", "snippet": {...}, "status": {...}}`.
  Публичный URL: `https://www.youtube.com/watch?v={id}` или `https://youtu.be/{id}` (для Shorts: `https://www.youtube.com/shorts/{id}`).

---

## 4. Установка кастомного превью (Thumbnail)

- **Эндпоинт**: `POST https://youtube.googleapis.com/upload/youtube/v3/thumbnails/set?videoId={videoId}&uploadType=media`
- **Заголовки**:
  - `Authorization: Bearer {access_token}`
  - `Content-Type: image/jpeg` (или `image/png`)
- **Тело**: сырые бинарные байты картинки (максимум 2 МБ, рекомендуемое разрешение 1280x720).
- *Ловушка из кода*: Для установки кастомного превью канал автора **должен быть верифицирован по номеру телефона**. Неверифицированный канал получает ошибку `HTTP 403 Forbidden` даже при наличии всех правильных скоупов.

---

## 5. Как реализовать в своем проекте (Python)

```python
import os
import httpx

async def upload_youtube_video(
    access_token: str,
    video_path: str,
    title: str,
    description: str,
    privacy: str = "public"
) -> str:
    file_size = os.path.getsize(video_path)

    metadata = {
        "snippet": {
            "title": title[:100],
            "description": description[:5000],
            "categoryId": "22"
        },
        "status": {
            "privacyStatus": privacy,
            "selfDeclaredMadeForKids": False
        }
    }

    async with httpx.AsyncClient(timeout=300.0) as client:
        # 1. Инициализация сессии
        init_headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json; charset=UTF-8",
            "X-Upload-Content-Type": "video/mp4",
            "X-Upload-Content-Length": str(file_size)
        }
        init_resp = await client.post(
            "https://youtube.googleapis.com/upload/youtube/v3/videos?uploadType=resumable&part=snippet,status,id",
            headers=init_headers,
            json=metadata
        )
        init_resp.raise_for_status()
        session_url = init_resp.headers["Location"]

        # 2. Загрузка байтов файла
        with open(video_path, "rb") as f:
            video_bytes = f.read()

        upload_headers = {
            "Content-Type": "video/mp4",
            "Content-Length": str(file_size)
        }
        upload_resp = await client.put(session_url, headers=upload_headers, content=video_bytes)
        upload_resp.raise_for_status()

        video_id = upload_resp.json()["id"]
        return video_id
```
