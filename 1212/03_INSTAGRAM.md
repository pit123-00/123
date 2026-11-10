# Отчет по подключению и автопостингу: Instagram

Файлы в кодовой базе `treg`:
- Настройки провайдера: `src/treg/oauth_providers.py` (`INSTAGRAM`)
- Каталог эндпоинтов: `src/treg/catalog/instagram.yaml`, `src/treg/catalog/instagram.extended.yaml`
- Описание правил авторизации: `src/treg/domain/connections/authorization.py`
- Обмен токенов: `src/treg/infra/oauth_exchange.py` (`_extend_instagram_token`)

---

## 1. Как устроена авторизация и подключение Instagram в `treg`

Instagram поддерживает **две принципиально разные модели авторизации**, и в коде `treg` они обе реализованы через объект `AuthorizationMethod`:

### Метод 1: Прямой Instagram Login (`instagram-login`)
Подключение профессионального аккаунта Instagram (Creator или Business) напрямую, без обязательной привязки к странице Facebook.
- **Auth URI**: `https://www.instagram.com/oauth/authorize`
- **Token URI**: `https://api.instagram.com/oauth/access_token`
- **Base URL**: `https://graph.instagram.com/v25.0`
- **Параметры авторизации**: `{"enable_fb_login": "false"}`
- **Разделитель скоупов**: запятая `","`.
- **Scopes**:
  - `read`: `instagram_business_basic`, `instagram_business_manage_insights`
  - `post`: `instagram_business_basic`, `instagram_business_manage_insights`, **`instagram_business_content_publish`**
  - `manage`: добавляет `instagram_business_manage_comments`, `instagram_business_manage_messages`.
- **Обмен на долгоживущий токен (`_extend_instagram_token`)**:
  1. Первичный POST на `api.instagram.com/oauth/access_token` возвращает короткоживущий токен.
  2. Запрос на продление:
     `GET https://graph.instagram.com/access_token?grant_type=ig_exchange_token&client_secret=...&access_token={token}`
     Возвращает 60-дневный токен (`expires_in: 5184000`).
  3. Продление токена в фоне (`refresh_token`):
     `GET https://graph.instagram.com/refresh_access_token?grant_type=ig_refresh_token&access_token={token}`.
     Токен можно продлевать бесконечно, пока он валиден (раз в ~30-50 дней).

### Метод 2: Через связанную страницу Facebook (`facebook-page`)
Исторический метод Instagram Graph API. Требует, чтобы Instagram Professional аккаунт был привязан к Facebook Page.
- **Auth URI**: `https://www.facebook.com/v25.0/dialog/oauth`
- **Base URL**: `https://graph.facebook.com/v25.0`
- **Scopes**: `instagram_basic`, `instagram_content_publish`, `instagram_manage_comments`, `pages_show_list`, `pages_read_engagement`, `business_management`.
- **Поиск ID аккаунта Instagram (`discover_path`)**:
  Запрос к Graph API: `GET /me/accounts?fields=instagram_business_account{id,username}`
  Из ответа извлекается `instagram_business_account.id` (это так называемый `ig_user_id`).
- Преимущества этого метода: доступ к поиску по хэштегам, отслеживанию упоминаний аккаунта (`mentions`), каталогам товаров.

---

## 2. Механизм постинга в Instagram (Content Publishing API)

В кодовой базе `treg` детально расписана специфика публикаций в Instagram:
> **Важное правило Instagram Graph API**:
> В Instagram **нет единого вызова "опубликовать пост"**. Любая публикация — это минимум двухэтапный процесс (а для каруселей — четырехэтапный) через концепцию **Медиа-контейнеров (Media Containers)**.
> Кроме того, API **никогда не принимает бинарные байты файлов**. Instagram принимает только **прямой публичный URL (`image_url` / `video_url`)**, по которому серверы Meta сами скачивают файл. URL должен быть публичным (не localhost, без базовой авторизации) и жить достаточно долго.

### Лимиты публикаций
- Meta ограничивает аккаунт: **100 публикаций за скользящий 24-часовой период** (карусель считается за 1 публикацию).
- Остаток лимита проверяется эндпоинтом:
  `GET /{ig_user_id}/content_publishing_limit`.
- Контейнеры хранятся 24 часа, после чего сгорают, если не были опубликованы.

---

## 3. Пошаговые сценарии публикаций

### 3.1. Публикация одного изображения
1. **Создание контейнера**:
   - `POST /{ig_user_id}/media`
   - Параметры в теле:
     ```json
     {
       "image_url": "https://mysite.com/images/post1.jpg",
       "caption": "Текст нашего поста с хэштегами #news #tech"
     }
     ```
   - Ответ: `{ "id": "179234567890" }` (это `creation_id` / container ID).
2. **Проверка статуса контейнера (опционально для фото, обязательно для видео)**:
   - `GET /{creation_id}?fields=status_code`
   - Дождаться статуса `FINISHED`.
3. **Финальная публикация**:
   - `POST /{ig_user_id}/media_publish`
   - Параметры:
     ```json
     {
       "creation_id": "179234567890"
     }
     ```
   - Ответ: `{ "id": "1809988776655" }` — ID опубликованного поста. Публичный URL: `https://www.instagram.com/p/{shortcode}/`.

### 3.2. Публикация видео / Reels
1. **Создание контейнера**:
   - `POST /{ig_user_id}/media`
   - Параметры:
     ```json
     {
       "media_type": "REELS",
       "video_url": "https://mysite.com/videos/reel.mp4",
       "caption": "Наш новый Reel!",
       "share_to_feed": true,
       "thumb_offset": 1000
     }
     ```
2. **Ожидание транскодирования видео**:
   - Периодический опрос `GET /{creation_id}?fields=status_code` с интервалом в 5–10 секунд.
   - Статус переходит: `IN_PROGRESS` -> `FINISHED` (или `ERROR`).
3. **Публикация**:
   - `POST /{ig_user_id}/media_publish` с `creation_id`.

### 3.3. Публикация Карусели (Альбома от 2 до 10 слайдов)
1. Для каждого фото или видео создается дочерний контейнер:
   - `POST /{ig_user_id}/media` с параметрами:
     `{"image_url": "...", "is_carousel_item": true}`
   - Получаем список ID: `[id_item1, id_item2, id_item3]`.
2. Создается родительский контейнер карусели:
   - `POST /{ig_user_id}/media` с параметрами:
     ```json
     {
       "media_type": "CAROUSEL",
       "children": ["id_item1", "id_item2", "id_item3"],
       "caption": "Общая подпись ко всей карусели"
     }
     ```
   - Получаем `carousel_container_id`.
3. Публикуем карусель:
   - `POST /{ig_user_id}/media_publish` с `creation_id: carousel_container_id`.

### 3.4. Публикация Stories (Историй)
- `POST /{ig_user_id}/media` с `media_type: "STORIES"`, `image_url` или `video_url`.
- Затем `media_publish`.

---

## 4. Как реализовать в своем проекте (Python / FastAPI)

```python
import asyncio
import httpx

class InstagramPublisher:
    def __init__(self, access_token: str, ig_user_id: str):
        self.token = access_token
        self.ig_user_id = ig_user_id
        self.base_url = "https://graph.instagram.com/v25.0"

    async def publish_photo(self, image_url: str, caption: str) -> str:
        async with httpx.AsyncClient() as client:
            # Шаг 1: Создание контейнера
            r_container = await client.post(
                f"{self.base_url}/{self.ig_user_id}/media",
                headers={"Authorization": f"Bearer {self.token}"},
                json={"image_url": image_url, "caption": caption}
            )
            r_container.raise_for_status()
            creation_id = r_container.json()["id"]

            # Шаг 2: Проверка готовности
            for _ in range(10):
                status_res = await client.get(
                    f"{self.base_url}/{creation_id}",
                    headers={"Authorization": f"Bearer {self.token}"},
                    params={"fields": "status_code"}
                )
                status = status_res.json().get("status_code")
                if status == "FINISHED":
                    break
                elif status == "ERROR":
                    raise RuntimeError("Instagram media container processing failed")
                await asyncio.sleep(2)

            # Шаг 3: Публикация
            r_pub = await client.post(
                f"{self.base_url}/{self.ig_user_id}/media_publish",
                headers={"Authorization": f"Bearer {self.token}"},
                json={"creation_id": creation_id}
            )
            r_pub.raise_for_status()
            return r_pub.json()["id"]
```
