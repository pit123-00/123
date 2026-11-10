# Отчет по подключению и автопостингу: Facebook Pages

Файлы в кодовой базе `treg`:
- Настройки провайдера: `src/treg/oauth_providers.py` (`FACEBOOK`)
- Каталог эндпоинтов: `src/treg/catalog/facebook.yaml`, `src/treg/catalog/facebook.extended.yaml`
- Логика авторизации и ресурсов: `src/treg/application/connect.py`

---

## 1. Как устроена авторизация и подключение Facebook в коде `treg`

### Точки входа и протокол
- **Auth URI**: `https://www.facebook.com/v25.0/dialog/oauth`
- **Token URI**: `https://graph.facebook.com/v25.0/oauth/access_token`
- **Base URL**: `https://graph.facebook.com/v25.0`
- **Протокол**: OAuth 2.0 Authorization Code Grant с автоматическим обменом на долгоживущий токен (Long-Lived Token Exchange).
- **Разделитель Scopes**: пробел `" "`.

### Разделение возможностей (Capabilities & Scopes)
В коде `treg` определены следующие уровни разрешений:
1. **`read`**:
   - `pages_show_list` — базовый доступ к списку страниц пользователя.
   - `pages_read_engagement` — чтение вовлеченности аудитории и контента страницы.
   - `read_insights` — статистика и аналитика страницы.
   - `business_management` — критически важный скоуп! Необходим, если страницы привязаны к Business Manager (Meta Business Portfolio). Без него вызов `/me/businesses` возвращает "Missing Permission", и страницы агентств/бизнесов оказываются невидимыми.
2. **`post`** (для автопостинга):
   - Все права из `read` + **`pages_manage_posts`**. Позволяет публиковать, планировать и удалять посты на странице.
3. **`manage`** (полное управление):
   - Все права из `post` + `pages_manage_engagement` (модерация комментариев), `pages_read_user_content`, `pages_manage_metadata` (настройки страницы и вебхуки), `pages_messaging` (Messenger), `publish_video` (публикация нативных видео/Reels), `leads_retrieval`, `pages_manage_ads`, `catalog_management`.

---

## 2. Ключевая архитектурная особенность: Page Access Token vs User Token

**Главная ловушка Meta Graph API**:
Запросы на публикацию в ленту страницы (`/{page_id}/feed`, `/{page_id}/photos`, etc.) **категорически отклоняют токен пользователя (User Access Token)** с ошибкой `code 190 / subcode 2069032` ("This method must be called with a Page Access Token").

### Как эта проблема решена в коде `treg`:
1. После того как пользователь прошел OAuth-авторизацию, система получает **User Access Token**.
2. Затем короткоживущий User Token обменивается на 60-дневный:
   ```http
   GET https://graph.facebook.com/v25.0/oauth/access_token
     ?grant_type=fb_exchange_token
     &client_id={meta_client_id}
     &client_secret={meta_client_secret}
     &fb_exchange_token={short_lived_user_token}
   ```
3. Далее вызывается обнаружение ресурсов (Resource Discovery):
   - Прямые страницы: `GET /me/accounts?fields=id,name,access_token`
   - Страницы в Business Portfolio: `GET /me/businesses?fields=owned_pages{id,name,access_token},client_pages{id,name,access_token}`
4. При выборе конкретной страницы (Page Selection) система извлекает из ответа `access_token` этой страницы (Page Access Token).
5. Полученный Page Token сохраняется в зашифрованном виде. **Все последующие запросы на публикацию и чтение контента этой страницы выполняются ИМЕННО с Page Access Token!**
6. Для безопасности ко всем запросам прикрепляется подпись `appsecret_proof`:
   `appsecret_proof = HMAC-SHA256(meta_client_secret, page_access_token)`.

---

## 3. Возможности постинга: Что и как можно публиковать

Все действия публикации происходят от имени Страницы (Page), а не от личного профиля (Meta запретила публиковать в личные профили через API много лет назад).

### 3.1. Публикация текстового поста и постов со ссылкой
- **Эндпоинт**: `POST /{page_id}/feed`
- **Заголовки**: `Authorization: Bearer {page_access_token}`
- **Параметры**:
  - `message` (string): текст публикации.
  - `link` (string): ссылка. Meta автоматически сгенерирует OpenGraph-превью (карточку) статьи.
  - `published` (boolean): `true` для мгновенной публикации. `false` для неопубликованного поста (черновика / запланированного поста).
  - `scheduled_publish_time` (integer, Unix timestamp): время отложенной публикации (от 10 минут до 6 месяцев в будущем). Требует `published: false`.
  - `attached_media` (array of objects): прикрепление ранее загруженных неопубликованных фото: `[{"media_fbid": "12345"}, {"media_fbid": "67890"}]`.

### 3.2. Публикация фотографий (Одиночные фото и карусели)
- **Эндпоинт**: `POST /{page_id}/photos`
- **Параметры**:
  - `url` (string): публично доступный URL изображения, либо multipart-загрузка файла.
  - `caption` (string): подпись к фото.
  - `published` (boolean):
    - `true`: фото публикуется сразу в ленту как отдельный пост.
    - `false`: фото загружается как неопубликованное (staged), возвращается `{ "id": "<photo_id>" }`.
- **Схема создания поста с альбомом (каруселью) из нескольких фото**:
  1. Загрузить первое фото: `POST /{page_id}/photos` с `published=false`. Получить `id_1`.
  2. Загрузить второе фото: `POST /{page_id}/photos` с `published=false`. Получить `id_2`.
  3. Опубликовать пост: `POST /{page_id}/feed` с:
     ```json
     {
       "message": "Наш новый альбом",
       "attached_media": [
         {"media_fbid": "id_1"},
         {"media_fbid": "id_2"}
       ]
     }
     ```

### 3.3. Публикация видео и Reels
- **Эндпоинт**: `POST /{page_id}/videos`
- **Параметры**:
  - `file_url` (string): прямой URL видеофайла, который Meta скачает на свои серверы.
  - `title` (string): заголовок видео.
  - `description` (string): описание / текст публикации.
  - Для Reels: специальные флаги публикации в формате Reels.

### 3.4. Управление и модерация (из `facebook.extended.yaml`)
- `POST /{post_id}` — редактирование текста уже опубликованного поста (`message`).
- `DELETE /{post_id}` — удаление поста со страницы.
- `POST /{comment_id}/comments` — ответ на комментарий от имени страницы.
- `POST /{comment_id}?is_hidden=true` — скрытие нежелательного комментария.
- `DELETE /{comment_id}` — удаление комментария.

---

## 4. Как повторить подключение Facebook в своем проекте (Step-by-step)

1. **Регистрация приложения в Meta for Developers**:
   - Создать приложение типа "Business".
   - Подключить продукт "Facebook Login for Business".
   - В настройках указать валидный `Redirect URI` (например, `https://your-domain.com/auth/facebook/callback`).
   - Получить `App ID` и `App Secret`.
   - В App Review запросить разрешения: `pages_show_list`, `pages_read_engagement`, `pages_manage_posts`, `business_management`. (В режиме разработки доступно администраторам и тестерам приложения без прохождения ревью).

2. **Формирование ссылки на авторизацию**:
   ```python
   def get_facebook_auth_url(client_id: str, redirect_uri: str, state: str) -> str:
       scopes = [
           "pages_show_list",
           "pages_read_engagement",
           "read_insights",
           "business_management",
           "pages_manage_posts"
       ]
       return (
           f"https://www.facebook.com/v25.0/dialog/oauth?"
           f"client_id={client_id}&"
           f"redirect_uri={redirect_uri}&"
           f"state={state}&"
           f"scope={','.join(scopes)}"
       )
   ```

3. **Обмен кода на User Token и Long-Lived Token**:
   ```python
   async def handle_facebook_callback(code: str, client_id: str, client_secret: str, redirect_uri: str):
       async with httpx.AsyncClient() as client:
           # 1. Получаем короткоживущий токен
           r = await client.get(
               "https://graph.facebook.com/v25.0/oauth/access_token",
               params={
                   "client_id": client_id,
                   "client_secret": client_secret,
                   "redirect_uri": redirect_uri,
                   "code": code
               }
           )
           short_token = r.json()["access_token"]

           # 2. Обмениваем на 60-дневный долгоживущий токен
           r_long = await client.get(
               "https://graph.facebook.com/v25.0/oauth/access_token",
               params={
                   "grant_type": "fb_exchange_token",
                   "client_id": client_id,
                   "client_secret": client_secret,
                   "fb_exchange_token": short_token
               }
           )
           long_user_token = r_long.json()["access_token"]
           return long_user_token
   ```

4. **Получение списка страниц и их Page Access Tokens**:
   ```python
   async def get_facebook_pages(long_user_token: str, app_secret: str):
       proof = hmac.new(app_secret.encode(), long_user_token.encode(), hashlib.sha256).hexdigest()
       async with httpx.AsyncClient() as client:
           resp = await client.get(
               "https://graph.facebook.com/v25.0/me/accounts",
               params={"fields": "id,name,access_token", "appsecret_proof": proof},
               headers={"Authorization": f"Bearer {long_user_token}"}
           )
           pages = resp.json().get("data", [])
           # Сохраните в базу page["id"], page["name"], и page["access_token"]
           return pages
   ```

5. **Функция публикации поста**:
   ```python
   async def publish_facebook_post(page_id: str, page_token: str, app_secret: str, text: str, link: str = None):
       proof = hmac.new(app_secret.encode(), page_token.encode(), hashlib.sha256).hexdigest()
       payload = {"message": text, "published": True}
       if link:
           payload["link"] = link

       async with httpx.AsyncClient() as client:
           resp = await client.post(
               f"https://graph.facebook.com/v25.0/{page_id}/feed",
               params={"appsecret_proof": proof},
               headers={"Authorization": f"Bearer {page_token}"},
               json=payload
           )
           resp.raise_for_status()
           return resp.json()  # {"id": "123456789_987654321"}
   ```
