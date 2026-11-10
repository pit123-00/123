# Отчет по подключению и автопостингу: LinkedIn

Файлы в кодовой базе `treg`:
- Настройки провайдера: `src/treg/oauth_providers.py` (`LINKEDIN`)
- Каталог эндпоинтов: `src/treg/catalog/linkedin.yaml`
- Проверка токена и экспирации: `src/treg/domain/connections/refresh.py`

---

## 1. Как устроена авторизация LinkedIn в коде `treg`

### Точки входа и параметры
- **Auth URI**: `https://www.linkedin.com/oauth/v2/authorization`
- **Token URI**: `https://www.linkedin.com/oauth/v2/accessToken`
- **Base URL**: `https://api.linkedin.com`
- **Протокол**: OAuth 2.0 Authorization Code Flow.

### Особенности скоупов LinkedIn в коде:
LinkedIn предоставляет очень узкий набор разрешений для стандартных self-serve приложений:
- **Скоуп `write`**: `["openid", "profile", "email", "w_member_social"]`
  - `openid`, `profile`, `email` — получение базовых данных о пользователе.
  - `w_member_social` — публикация постов в личную ленту участника от его имени.

### Что намеренно исключено (Deliberately Absent в `linkedin.yaml`):
1. **Чтение постов или аналитики пользователя (`r_member_social`)**:
   LinkedIn выдает доступ на чтение постов участника **только партнерам по специальной заявке** (Partnership Programme). Обычные приложения не имеют права читать ленту.
2. **Публикация от имени компании (Company Pages)**:
   Требует скоупов `w_organization_social` / `rw_organization_admin` в рамках **Community Management API** для верифицированных компаний.
3. **Ads / Campaign Manager**:
   Требует утверждения в программе **Marketing Developer Platform**.

---

## 2. Критические нюансы токенов и версионирования API

### 1. Срок жизни токена и отсутствие автоматического Refresh
В файле `refresh.py` зафиксирована важная деталь:
> На стандартном бесплатном уровне LinkedIn выдает токен, который живет **60 дней**, но **не предоставляет `refresh_token`** для его фонового обновления.
> Токен работает 60 дней и затем тихо умирает. `treg` отслеживает поле `expires_at` и за 7 дней до истечения выставляет флаг `needs_reconnect: true`, предупреждая пользователя о необходимости повторного входа.

### 2. Обязательные заголовки для всех вызовов к `/rest/*`:
Все эндпоинты LinkedIn REST API требуют два специфических заголовка:
- `LinkedIn-Version: 202601` (формат даты YYYYMM — LinkedIn объявляет версии устаревшими каждый квартал!).
- `X-Restli-Protocol-Version: 2.0.0`.
Единственный эндпоинт, который не требует версионирования — это `GET /v2/userinfo` (именно поэтому он используется для проверки здоровья токена).

### 3. Определение профиля (Identity)
При подключении `treg` делает запрос к `GET /v2/userinfo` и извлекает ID пользователя `sub`. Идентификатор автора в API всегда имеет вид URN:
`urn:li:person:{sub}`.

---

## 3. Возможности постинга: Что и как можно публиковать

Публикация осуществляется через эндпоинт **`POST /rest/posts`**.

### 3.1. Текстовый пост и пост со ссылкой
- **Эндпоинт**: `POST https://api.linkedin.com/rest/posts`
- **Заголовки**:
  - `Authorization: Bearer {token}`
  - `LinkedIn-Version: 202601`
  - `X-Restli-Protocol-Version: 2.0.0`
  - `Content-Type: application/json`
- **Тело запроса**:
  ```json
  {
    "author": "urn:li:person:ABC123xyz",
    "commentary": "Делимся техническим отчетом о проектировании API.\n\nЧитать подробнее: https://example.com/post",
    "visibility": "PUBLIC",
    "distribution": {
      "feedDistribution": "MAIN_FEED",
      "targetEntities": [],
      "thirdPartyDistributionChannels": []
    },
    "lifecycleState": "PUBLISHED",
    "isReshareDisabledByAuthor": false
  }
  ```
- *Специфика форматирования текста (`commentary`)*:
  Спецсимволы `( ) [ ] { } < > @ | ~` в тексте публикации **должны быть экранированы обратным слэшем `\`**, иначе LinkedIn вернет ошибку синтаксического анализа!
- *Специфика ответа*:
  При успехе LinkedIn возвращает `HTTP 201 Created` с **пустым телом**! ID созданного поста передается в заголовке ответа:
  `x-restli-id: urn:li:share:1234567890`.

---

## 4. Трехшаговый "танец" с медиафайлами (Images & Video)

LinkedIn **никогда не принимает байты медиафайлов напрямую в эндпоинт `/rest/posts`**. Загрузка медиа устроена в 3 шага:

```
[1. initializeUpload] ──> Получаем URN и CDN uploadUrl
[2. Binary PUT]       ──> Отправляем байты файла на CDN uploadUrl
[3. posts.create]     ──> Публикуем пост, прикрепляя полученный URN
```

### 4.1. Публикация одного или нескольких изображений
1. **Инициализация**:
   - `POST /rest/images?action=initializeUpload`
   - Тело: `{"initializeUploadRequest": {"owner": "urn:li:person:ABC123xyz"}}`
   - Ответ:
     ```json
     {
       "value": {
         "uploadUrl": "https://media.licdn.com/mediaUpload/...",
         "image": "urn:li:image:D4E10AQ..."
       }
     }
     ```
2. **Загрузка байтов на CDN**:
   - `PUT {uploadUrl}` (единым запросом, бинарное тело). Нельзя дробить на чанки.
3. **Публикация поста**:
   - В поле `content` передается URN:
     ```json
     {
       "author": "urn:li:person:ABC123xyz",
       "commentary": "Пост с изображением",
       "visibility": "PUBLIC",
       "content": {
         "media": {
           "id": "urn:li:image:D4E10AQ...",
           "title": "Заголовок картинки"
         }
       },
       "distribution": {"feedDistribution": "MAIN_FEED"},
       "lifecycleState": "PUBLISHED"
     }
     ```
   - Для нескольких фото используется массив `content.multiImage.images: [{"id": "..."}, ...]`.

### 4.2. Публикация Видео
1. **Инициализация**:
   - `POST /rest/videos?action=initializeUpload`
   - В теле передается точный размер файла в байтах: `{"initializeUploadRequest": {"owner": "urn:li:...", "fileSizeBytes": 10485760}}`.
   - Ответ содержит массив инструкций: `uploadInstructions: [{"uploadUrl": "...", "firstByte": 0, "lastByte": 2097151}, ...]`.
2. **Загрузка частей видео**:
   - Каждая часть (по 2 МБ) отправляется отдельным PUT-запросом на свой `uploadUrl`.
   - **Обязательно сохранять заголовок `ETag` из каждого ответа!**
3. **Финализация видео**:
   - `POST /rest/videos?action=finalizeUpload`
   - Передается список ETag в строгом порядке:
     `{"finalizeUploadRequest": {"video": "urn:li:video:...", "uploadToken": "", "uploadedPartIds": ["etag1", "etag2"]}}`.
4. **Ожидание транскодирования**:
   - Опрос `GET /rest/videos/{url_encoded_video_urn}` до статуса `AVAILABLE`. Только после этого можно вызывать `/rest/posts`.

---

## 5. Как реализовать в своем проекте (Python)

```python
import httpx

class LinkedInPublisher:
    def __init__(self, access_token: str, person_urn: str):
        self.token = access_token
        self.person_urn = person_urn
        self.headers = {
            "Authorization": f"Bearer {access_token}",
            "LinkedIn-Version": "202601",
            "X-Restli-Protocol-Version": "2.0.0",
            "Content-Type": "application/json"
        }

    async def publish_text_post(self, text: str) -> str:
        # Экранирование спецсимволов для LinkedIn
        for ch in r"()[]{}<>@|~":
            text = text.replace(ch, f"\\{ch}")

        payload = {
            "author": self.person_urn,
            "commentary": text,
            "visibility": "PUBLIC",
            "distribution": {
                "feedDistribution": "MAIN_FEED",
                "targetEntities": [],
                "thirdPartyDistributionChannels": []
            },
            "lifecycleState": "PUBLISHED"
        }

        async with httpx.AsyncClient() as client:
            resp = await client.post("https://api.linkedin.com/rest/posts", headers=self.headers, json=payload)
            resp.raise_for_status()
            # ID поста возвращается в заголовке x-restli-id
            post_urn = resp.headers.get("x-restli-id")
            return post_urn
```
