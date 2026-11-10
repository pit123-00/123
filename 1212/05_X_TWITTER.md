# Отчет по подключению и автопостингу: X (Twitter)

Файлы в кодовой базе `treg`:
- Настройки провайдера: `src/treg/oauth_providers.py` (`X`)
- Каталог эндпоинтов: `src/treg/catalog/x.yaml`, `src/treg/catalog/x.extended.yaml`
- Обработка PKCE и Basic Auth: `src/treg/infra/oauth_exchange.py`, `src/treg/infra/oauth_refresh.py`

---

## 1. Как устроена авторизация X в коде `treg`

### Точки входа и параметры OAuth 2.0
- **Auth URI**: `https://x.com/i/oauth2/authorize`
- **Token URI**: `https://api.x.com/2/oauth2/token`
- **Base URL**: `https://api.x.com`
- **Протокол**: OAuth 2.0 с обязательным **PKCE (Proof Key for Code Exchange)**.

### Критические требования X API к авторизации:
1. **Обязательный PKCE (`pkce = True`)**:
   X категорически отклоняет обмен кода авторизации на токен, если запрос отправлен без `code_verifier`.
   В `treg` генерируется криптостойкий токен: `code_verifier = crypto.new_token()`. При формировании ссылки согласия рассчитывается challenge: `code_challenge = base64url(sha256(code_verifier))`.
2. **Аутентификация клиента strictly Basic Auth**:
   `token_endpoint_auth_method = "client_secret_basic"`.
   X возвращает HTTP 401, если передать `client_secret` в теле POST-запроса! Секрет и ID клиента должны передаваться исключительно в HTTP-заголовке:
   `Authorization: Basic base64(client_id:client_secret)`.
   Это же правило строго соблюдается и при обновлении токена через refresh token.
3. **Автономное продление токенов (`offline.access`)**:
   Обычный access token X живет ровно **2 часа**. Без скоупа `offline.access` платформа не выдает `refresh_token`, и подключение потребует ручного переподключения каждые 2 часа.
4. **Скоупы (Scopes)**:
   - `read`: `tweet.read`, `users.read`, `offline.access`
   - `write` (для автопостинга): `tweet.read`, `tweet.write`, `users.read`, `offline.access`.

---

## 2. Модель ценообразования X API (Pay-per-use Billing)

В кодовой базе `treg` зафиксированы важные факты об изменениях в тарифах X:
- X полностью отказался от традиционных подписок (Free / Basic / Pro) в пользу предоплатной системы по факту использования (Prepaid Pay-per-use), которая **списывается со счета владельца приложения (App Owner)**:
  - Публикация текстового твита: **$0.015** за вызов.
  - Публикация твита со ссылкой (URL в тексте): **$0.200** за вызов!
  - Чтение одного ресурса поста (GET): **$0.005**.
  - Чтение одного ресурса пользователя: **$0.010**.

---

## 3. Возможности постинга: Что и как можно публиковать

Основной эндпоинт публикации: `POST /2/tweets` (требует scope `tweet.write`).

### 3.1. Создание обычного поста (Твита)
- **Эндпоинт**: `POST /2/tweets`
- **Тело запроса**:
  ```json
  {
    "text": "Релиз новой версии системы! https://example.com/update"
  }
  ```
- **Ограничения длины текста**:
  - Стандартный аккаунт: **280 символов**.
  - Аккаунт с подпиской X Premium: **до 25 000 символов**.
  - Любая ссылка (URL) считается за фиксированные **23 символа** (сокращатель `t.co`).
- **Ответ**:
  `HTTP 201 Created` с телом:
  ```json
  {
    "data": {
      "id": "1898765432109876543",
      "text": "Релиз новой версии системы! https://example.com/update"
    }
  }
  ```
  *Примечание: X API не возвращает готовый публичный URL поста, он формируется вручную: `https://x.com/{username}/status/{data.id}`.*

### 3.2. Создание тредов (цепочек твитов) и ответов
Для создания треда каждый последующий пост ссылается на ID предыдущего через объект `reply`:
- **Тело второго твита**:
  ```json
  {
    "text": "Продолжение треда: часть 2...",
    "reply": {
      "in_reply_to_tweet_id": "1898765432109876543"
    }
  }
  ```

### 3.3. Цитирование твита (Quote Tweet)
Вместо объекта `reply` передается поле `quote_tweet_id`:
```json
{
  "text": "Интересный взгляд на вещи:",
  "quote_tweet_id": "1898765432109876543"
}
```

### 3.4. Опросы (Polls)
В тело можно добавить объект `poll` (нельзя совмещать с медиа):
```json
{
  "text": "Какой стек вы используете?",
  "poll": {
    "options": ["FastAPI", "Django", "Node.js", "Go"],
    "duration_minutes": 1440
  }
}
```

### 3.5. Ограничение круга отвечающих (`reply_settings`)
- Допустимые значения: `following`, `mentionedUsers`, `subscribers`, `verified`.
- *Ловушка из кода*: Если хотите разрешить отвечать всем, поле `reply_settings` нужно **опустить вообще** (отправка литерала `"everyone"` вызовет ошибку валидации).

---

## 4. Нюанс с медиафайлами (Картинки и видео в X)

В комментариях к коду `x.yaml` подробно разобран статус загрузки медиа:
> **Прикрепление медиа (фото/видео) невозможно без отдельного скоупа `media.write`!**
> Чтобы прикрепить картинку к твиту через параметр `media: {"media_ids": ["..."]}`, медиа-ID сначала нужно получить через `POST /2/media/upload` (или legacy v1.1 upload). Для этого приложению требуется разрешение `media.write`. Если у приложения запрошены только стандартные скоупы `tweet.read`, `tweet.write`, `users.read`, вызов загрузки медиа вернет 403 Forbidden.

---

## 5. Как реализовать в своем проекте (Python)

### 1. Генерация PKCE и ссылки авторизации
```python
import secrets
import hashlib
import base64
from urllib.parse import urlencode

def generate_pkce_pair():
    code_verifier = secrets.token_urlsafe(64)
    digest = hashlib.sha256(code_verifier.encode('ascii')).digest()
    code_challenge = base64.urlsafe_b64encode(digest).decode('ascii').rstrip('=')
    return code_verifier, code_challenge

def get_x_auth_url(client_id: str, redirect_uri: str, state: str, code_challenge: str) -> str:
    params = {
        "response_type": "code",
        "client_id": client_id,
        "redirect_uri": redirect_uri,
        "scope": "tweet.read tweet.write users.read offline.access",
        "state": state,
        "code_challenge": code_challenge,
        "code_challenge_method": "S256"
    }
    return f"https://x.com/i/oauth2/authorize?{urlencode(params)}"
```

### 2. Обмен кода на токен с Basic Auth
```python
import httpx

async def exchange_x_code(code: str, code_verifier: str, client_id: str, client_secret: str, redirect_uri: str):
    async with httpx.AsyncClient() as client:
        data = {
            "code": code,
            "grant_type": "authorization_code",
            "redirect_uri": redirect_uri,
            "code_verifier": code_verifier,
        }
        # Передача Client ID и Secret строго через HTTP Basic Auth!
        resp = await client.post(
            "https://api.x.com/2/oauth2/token",
            data=data,
            auth=(client_id, client_secret)
        )
        resp.raise_for_status()
        return resp.json()  # access_token, refresh_token, expires_in
```

### 3. Публикация твита
```python
async def publish_tweet(access_token: str, text: str, in_reply_to_id: str = None) -> str:
    payload = {"text": text}
    if in_reply_to_id:
        payload["reply"] = {"in_reply_to_tweet_id": in_reply_to_id}

    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "https://api.x.com/2/tweets",
            headers={
                "Authorization": f"Bearer {access_token}",
                "Content-Type": "application/json"
            },
            json=payload
        )
        resp.raise_for_status()
        tweet_data = resp.json()["data"]
        return tweet_data["id"]
```
