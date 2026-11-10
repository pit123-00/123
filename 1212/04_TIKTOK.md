# Отчет по подключению и автопостингу: TikTok

Файлы в кодовой базе `treg`:
- Настройки провайдера: `src/treg/oauth_providers.py` (`TIKTOK`)
- Каталог эндпоинтов публикации: `src/treg/catalog/tiktok.yaml`
- Обработка OAuth: `src/treg/application/connect.py`, `src/treg/infra/oauth_refresh.py`

---

## 1. Как устроена авторизация TikTok в коде `treg`

### Точки входа и особенности протокола
- **Auth URI**: `https://www.tiktok.com/v2/auth/authorize/`
- **Token URI**: `https://open.tiktokapis.com/v2/oauth/token/`
- **Base URL**: `https://open.tiktokapis.com`
- **Протокол**: OAuth 2.0 (Login Kit for Web v2).

### Критические нюансы TikTok OAuth в коде:
1. **Параметр `client_key` вместо `client_id`**:
   `client_id_param = "client_key"`. TikTok полностью игнорирует стандартный параметр `client_id` и выдает ошибку, если передавать его стандартно.
2. **Разделитель Scopes**:
   `scope_separator = ","` (запятая, а не стандартный для OAuth2 пробел!).
   При этом в коде `treg` предусмотрена нормализация: в базу данных скоупы сохраняются через пробел, чтобы внутренние валидаторы работали универсально.
3. **PKCE для localhost / loopback**:
   `pkce_loopback_hex = True`. При разработке на localhost TikTok требует `code_challenge` в виде HEX-строки SHA-256 хэша (а не стандартного Base64URL).
4. **Обновление токена**:
   При обновлении токена через refresh token в теле запроса обязательно должен присутствовать `client_key` (а не `client_id`).

### Разрешения (Scopes) и Продукты в TikTok Portal
В TikTok разрешения выдаются пакетами (Products) в портале разработчика:
- `user.info.basic`, `user.info.profile`, `video.list`, `user.info.stats` (продукт **Login Kit**).
- `video.upload` (продукт **Content Posting API**).
- `video.publish` (активируется тумблером **Direct Post** внутри Content Posting API).

В `treg` заведены 3 уровня доступа (Capabilities):
- **`read`**: чтение профиля, списка видео и статистики.
- **`draft`** (`video.upload`): отправка видео во "Входящие" автора (Inbox).
- **`post`** (`video.upload` + `video.publish`): публикация видео напрямую в профиль.

---

## 2. Разделение режимов публикации: Прямой постинг vs Inbox (Черновик)

В коде `treg` объясняется принципиальная разница между `draft` и `post`:
1. **Режим Inbox (`video.upload`)**:
   Видео загружается на серверы TikTok и отправляется в мобильное приложение автора во вкладку **Inbox** (Входящие). Автор получает push-уведомление, открывает приложение, может наложить трендовую музыку, фильтры, стикеры и нажать "Опубликовать".
   *Внимание: Если автор не опубликовал видео из Inbox в течение 24 часов, TikTok удаляет файл!*
2. **Режим Direct Post (`video.publish`)**:
   Видео публикуется сразу в открытый профиль автора без ручного подтверждения в приложении.

---

## 3. Ловушка неаудированных приложений (Unaudited-App Trap)

В комментариях к коду `tiktok.yaml` задокументировано важнейшее ограничение TikTok API:
> **Внимание!**
> Пока ваше приложение не прошло официальный аудит использования Direct Post со стороны команды модерации TikTok, попытка опубликовать видео в открытый доступ вернет ошибку:
> `unaudited_client_can_only_post_to_private_accounts`.
> В режиме разработки/тестирования TikTok **требует, чтобы `privacy_level` был строго `SELF_ONLY`**, и **сам аккаунт автора в момент публикации был переключен в приватный режим** в настройках профиля!
> Кроме того, неаудированное приложение ограничено максимум **5 публикующими пользователями в течение 24 часов**.
> Режим Inbox (`inbox/video/init/`) этим ограничением не затронут.

Перед любой публикацией код рекомендует запросить разрешенные опции для данного автора:
`POST /v2/post/publish/creator_info/query/`.
Этот запрос возвращает список допустимых уровней приватности (`privacy_level_options`), максимальную длину видео и лимиты.

---

## 4. Пошаговый процесс публикации (Content Posting Flow)

Публикация в TikTok состоит из 3 обязательных шагов:

### Шаг 1: Инициализация публикации (`init`)
Выбирается один из эндпоинтов:
- **Видео напрямую в профиль**:
  `POST /v2/post/publish/video/init/`
- **Видео во входящие (Draft)**:
  `POST /v2/post/publish/inbox/video/init/`
- **Фотопост (карусель фотографий)**:
  `POST /v2/post/publish/content/init/`

Пример тела запроса для видео (`POST /v2/post/publish/video/init/`):
```json
{
  "post_info": {
    "title": "Инженерный разбор архитектуры #tech #coding",
    "privacy_level": "PUBLIC_TO_EVERYONE",
    "disable_comment": false,
    "disable_duet": false,
    "disable_stitch": false,
    "video_cover_timestamp_ms": 1000,
    "is_aigc": false,
    "brand_content_toggle": false,
    "brand_organic_toggle": false
  },
  "source_info": {
    "source": "FILE_UPLOAD",
    "video_size": 10485760,
    "chunk_size": 10485760,
    "total_chunk_count": 1
  }
}
```
*Примечание: Поле `is_aigc` (AI Generated Content) маркирует контент, созданный ИИ — TikTok юридически требует указывать его честно.*

**Ответ от TikTok**:
```json
{
  "data": {
    "publish_id": "v_pub_file~v2.123456789",
    "upload_url": "https://open-upload.tiktokapis.com/video/..."
  }
}
```

### Шаг 2: Загрузка бинарных байтов видео (`PUT`)
Байты видеофайла отправляются прямым HTTP PUT-запросом на полученный `upload_url`:
- Заголовок `Content-Type: video/mp4`
- Заголовок `Content-Range: bytes 0-10485759/10485760`
- В теле — бинарные байты файла.
- Для больших файлов можно разбивать на чанки (минимум 5 МБ, кроме последнего).

### Шаг 3: Опрос статуса публикации (Polling)
- **Эндпоинт**: `POST /v2/post/publish/status/fetch/`
- **Тело запроса**: `{"publish_id": "v_pub_file~v2.123456789"}`
- **Возможные статусы**:
  - `PROCESSING_UPLOAD` — файл загружается
  - `PROCESSING_DOWNLOAD` — сервер обрабатывает видео
  - `SEND_TO_USER_INBOX` — успешно доставлено во входящие (для черновиков)
  - `PUBLISH_COMPLETE` — успешно опубликовано в профиле
  - `FAILED` — публикация отклонена (недопустимый кодек, нарушение авторских прав и т.д.)
- Код рекомендует опрашивать статус каждые 15–20 секунд. Полный цикл может занимать от 30 секунд до нескольких минут.
- При успехе в поле `data.publicaly_available_post_id` возвращается массив с ID созданного видео. Публичный URL: `https://www.tiktok.com/@{username}/video/{post_id}`.

---

## 5. Как реализовать в своем проекте (Python)

```python
import httpx
import os
import asyncio

async def publish_tiktok_video(access_token: str, video_path: str, caption: str):
    file_size = os.path.getsize(video_path)
    base_url = "https://open.tiktokapis.com"

    async with httpx.AsyncClient(timeout=60.0) as client:
        headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json; charset=UTF-8"
        }

        # 1. Init
        init_payload = {
            "post_info": {
                "title": caption,
                "privacy_level": "PUBLIC_TO_EVERYONE",
                "disable_comment": False,
                "is_aigc": False
            },
            "source_info": {
                "source": "FILE_UPLOAD",
                "video_size": file_size,
                "chunk_size": file_size,
                "total_chunk_count": 1
            }
        }
        r_init = await client.post(f"{base_url}/v2/post/publish/video/init/", headers=headers, json=init_payload)
        r_init.raise_for_status()
        init_data = r_init.json()["data"]
        publish_id = init_data["publish_id"]
        upload_url = init_data["upload_url"]

        # 2. Upload bytes
        with open(video_path, "rb") as f:
            video_bytes = f.read()

        put_headers = {
            "Content-Type": "video/mp4",
            "Content-Range": f"bytes 0-{file_size - 1}/{file_size}"
        }
        r_upload = await client.put(upload_url, headers=put_headers, content=video_bytes)
        r_upload.raise_for_status()

        # 3. Poll status
        for _ in range(30):
            await asyncio.sleep(15)
            r_status = await client.post(
                f"{base_url}/v2/post/publish/status/fetch/",
                headers=headers,
                json={"publish_id": publish_id}
            )
            status_data = r_status.json().get("data", {})
            status = status_data.get("status")
            if status == "PUBLISH_COMPLETE":
                post_id = status_data.get("publicaly_available_post_id", [None])[0]
                return post_id
            elif status == "FAILED":
                raise RuntimeError(f"TikTok publish failed: {status_data.get('fail_reason')}")

        raise TimeoutError("TikTok video processing timed out")
```
