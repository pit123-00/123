# Архитектура подключения соцсетей и автопостинга в `treg`

Данный отчет составлен на основе детального анализа исходного кода репозитория `treg` (`src/treg/oauth_providers.py`, `src/treg/application/connect.py`, `src/treg/infra/oauth_exchange.py`, `src/treg/infra/oauth_refresh.py`, `src/treg/infra/upstream/`, `src/treg/domain/connections/`).

В отчете описано, как устроен весь цикл подключения сторонних платформ через OAuth 2.0 / API-токены, безопасное хранение, обновление токенов, автоматическое проксирование и инъекция авторизационных данных, а также практическое руководство по повторению этой архитектуры в вашем проекте.

---

## 1. Концепция и общая модель `treg`

`treg` решает фундаментальную проблему интеграции с соцсетями для агентов и бэкенда:
1. **Разделение прав (Scopes / Capabilities)**: запрашиваются только минимально необходимые права (например, `read`, `post`, `manage`), чтобы пользователи не пугались огромных списков разрешений при авторизации.
2. **Безопасное хранение токенов**: access и refresh токены шифруются симметричным шифрованием (`Fernet`) в базе данных.
3. **Автономное обновление (Unattended Refresh)**: система сама отслеживает срок жизни токенов (`expires_at`), обновляет их до истечения (`_SKEW = 60s`) через refresh token или long-lived exchange.
4. **Прозрачная инъекция (Credential Injection)**: клиент/агент делает запрос к прокси-эндпоинту `treg`, а прокси подставляет актуальный Bearer token, query-параметры или HMAC-подпись (`appsecret_proof`) "на лету".
5. **Привязка к ресурсам (Resource Discovery & Scoping)**: для сервисов с множеством сущностей (Facebook Pages, YouTube Channels) пользователь выбирает целевой ресурс, и система сама извлекает специфичный токен страницы/канала.

---

## 2. Жизненный цикл подключения соцсети (OAuth 2.0 Flow)

### Шаг 1: Инициализация авторизации (`start_oauth_connection`)
Файл: `src/treg/application/connect.py`

1. Клиент выбирает провайдера (`provider_name`, например `facebook`, `instagram`, `tiktok`, `x`, `youtube`, `linkedin`) и уровень доступа (`capability`, например `post`).
2. Из реестра `oauth_providers.PROVIDERS` извлекаются настройки провайдера:
   - `auth_uri`, `token_uri`
   - `client_id`, `client_secret` (из настроек окружения приложения)
   - Список `scopes` для выбранного capability.
   - Специфика OAuth: PKCE (`code_verifier`), разделитель scopes (` ` или `,`), имя параметра `client_id` (например, `client_key` для TikTok), метод передачи секрета (`client_secret_post` или `client_secret_basic`).
3. Генерируется уникальный криптографический `state` (токен безопасности от CSRF).
4. Если провайдер требует PKCE (например, X / Twitter) или loopback PKCE (TikTok на localhost):
   - Генерируется `code_verifier = crypto.new_token()`.
   - Вычисляется `code_challenge = base64url(sha256(code_verifier))` или hex-sha256.
5. Запись сохраняется в таблицу `pending_oauth` в статусе `pending` с TTL (обычно 10–15 минут).
6. Формируется Consent URL:
   ```python
   # Пример для X/TikTok/Facebook
   params = {
       client_id_param: client_id,
       "redirect_uri": redirect_uri,  # строго свой callback /oauth/callback
       "response_type": "code",
       "scope": scope_separator.join(scopes),
       "state": state,
   }
   if code_challenge:
       params["code_challenge"] = code_challenge
       params["code_challenge_method"] = "S256"
   ```
7. Пользователь перенаправляется в браузер на страницу согласия соцсети.

---

### Шаг 2: Обработка Callback и обмен кода на токены (`complete_oauth_connection`)
Файлы: `src/treg/application/connect.py`, `src/treg/infra/oauth_exchange.py`

1. Соцсеть возвращает пользователя на `/oauth/callback?code=...&state=...`.
2. Проверяется валидность `state` в базе, отсутствие истечения срока (TTL) и однократность использования.
3. **Обмен кода на токен (`HTTPXOAuthExchangePort.exchange_code`)**:
   - Отправляется POST-запрос на `token_uri`:
     ```python
     data = {
         "grant_type": "authorization_code",
         "code": code,
         "redirect_uri": pending.redirect_uri,
         client_id_param: pending.client_id,
     }
     if pending.code_verifier:
         data["code_verifier"] = pending.code_verifier
     ```
   - Способ авторизации:
     - Если `client_secret_basic` (X, Pinterest): в HTTP Header `Authorization: Basic base64(client_id:client_secret)`.
     - Иначе в теле: `data["client_secret"] = client_secret`.
4. **Обмен на долгоживущий токен (Long-Lived Token Exchange)**:
   - **Meta / Facebook Graph API**:
     Короткоживущий токен пользователя (~1-2 часа) обменивается на 60-дневный:
     `GET /v25.0/oauth/access_token?grant_type=fb_exchange_token&client_id=...&client_secret=...&fb_exchange_token={short_token}`.
   - **Instagram Basic Display / Login**:
     `GET https://graph.instagram.com/access_token?grant_type=ig_exchange_token&client_secret=...&access_token={short_token}`.
5. **Сохранение в базу (`Secret`)**:
   - Токены упаковываются в JSON:
     ```json
     {
       "access_token": "...",
       "token": "...",
       "refresh_token": "...",
       "client_id": "...",
       "client_secret": "...",
       "token_uri": "...",
       "expires_at": 1780000000.0,
       "token_endpoint_auth_method": "client_secret_basic"
     }
     ```
   - JSON шифруется ключом приложения (`crypto.encrypt`) и сохраняется в таблице `secrets`.
6. **Определение личности (Identity Recording)**:
   - Выполняется запрос к профилю (например, `/v2/userinfo` для LinkedIn, `/2/users/me` для X, `/v2/user/info/` для TikTok), чтобы зафиксировать ID пользователя и имя/юзернейм (например, `urn:li:person:...` или `@username`).

---

### Шаг 3: Выбор ресурса (Resource Discovery & Scoping)
Файл: `src/treg/application/connect.py` (`list_connection_resources`, `select_connection_resource`)

Для платформ, где публикация идет не в личный профиль, а в конкретную страницу или канал (Facebook Pages, YouTube Channel):
1. **Discovery**:
   - Facebook: запрос `GET /me/accounts?fields=id,name` и `GET /me/businesses?fields=owned_pages{id,name},client_pages{id,name}`.
   - YouTube: запрос `GET /youtube/v3/channels?part=snippet&mine=true`.
2. **Selection**:
   - Пользователь выбирает страницу/канал.
   - **Для Facebook Page**: критически важный нюанс! Запросы к API Facebook Page отклоняются с User Access Token (ошибка code 190 / subcode 2069032). Требуется **Page Access Token**. При выборе страницы `treg` запрашивает `/me/accounts?fields=id,access_token`, извлекает токен именно этой страницы и сохраняет его как `page_access_token`. Все последующие вызовы публикации идут с этим Page Token.

---

## 3. Механизм фонового обновления токенов (Token Refresh)
Файлы: `src/treg/domain/connections/refresh.py`, `src/treg/infra/oauth_refresh.py`

1. **Проверка свежести**:
   Перед выполнением запроса проверяется `is_stale(blob, skew=60)`:
   `time.time() > expires_at - 60`.
2. **Процесс обновления (`HTTPXOAuthRefreshPort.exchange`)**:
   - Блокировка мьютекса на уровне ID соединения (`asyncio.Lock`), чтобы параллельные запросы не вызывали гонку обновления.
   - Запрос на обновление:
     - Для большинства (Google, X, TikTok):
       POST на `token_uri` с `grant_type=refresh_token`, `refresh_token=...`.
       Учитывается метод авторизации клиента (`client_secret_basic` vs body).
     - Для Instagram:
       GET на `https://graph.instagram.com/refresh_access_token?grant_type=ig_refresh_token&access_token={token}`.
   - Полученный новый `access_token` (и опционально новый rotated `refresh_token`) перезаписывается в шифрованный blob в БД.
   - Обновляется `expires_at = time.time() + expires_in`.

---

## 4. Инъекция авторизации в запросы к API (Credential Injection)
Файлы: `src/treg/infra/upstream/injectors.py`, `src/treg/infra/upstream/relay.py`

Когда агент или сервис делает вызов к API:
1. Расшифровывается blob секрета.
2. Проверяется и при необходимости обновляется токен.
3. Инжектор (`oauth_injector`) подставляет токен в заголовки:
   `headers["Authorization"] = f"Bearer {token}"`.
4. **Защита Meta (Facebook / Instagram)**:
   Если провайдер имеет флаг `appsecret_proof: True`, вычисляется HMAC-SHA256 подпись:
   `proof = hmac.new(app_secret.encode('utf-8'), token.encode('utf-8'), hashlib.sha256).hexdigest()`.
   Параметр `appsecret_proof` автоматически добавляется в query params: `?appsecret_proof={proof}`. Это обязательно для продакшн-приложений Meta с включенной проверкой безопасности.

---

## 5. Как реализовать аналогичную систему в своем проекте

### Рекомендуемая архитектура базы данных (SQLAlchemy / PostgreSQL)

```python
# models.py
from datetime import datetime
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey, Boolean
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class SocialAccount(Base):
    __tablename__ = "social_accounts"

    id = Column(Integer, primary_key=True)
    org_id = Column(Integer, index=True)
    platform = Column(String(50))           # 'facebook', 'instagram', 'tiktok', 'x', 'youtube', 'linkedin'
    account_identifier = Column(String(100))# page_id, ig_user_id, channel_id, user_urn, etc.
    account_name = Column(String(255))      # Название страницы / канала / юзернейм

    # Зашифрованные данные токенов (Fernet)
    encrypted_credentials = Column(Text)    # JSON {access_token, refresh_token, token_uri, client_id, etc.}

    # Для Facebook/Instagram специфичный токен страницы:
    page_access_token_encrypted = Column(Text, nullable=True)

    expires_at = Column(DateTime, nullable=True)
    scopes = Column(Text)                   # Сохраненные скоупы через пробел
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

### Модуль безопасного обновления токена (Refresh Manager)

```python
# oauth_manager.py
import asyncio
import time
import json
import hmac
import hashlib
import httpx
from cryptography.fernet import Fernet

fernet = Fernet(ENCRYPTION_KEY)
_locks = {}

async def get_valid_token(account: SocialAccount) -> str:
    lock = _locks.setdefault(account.id, asyncio.Lock())
    async with lock:
        blob = json.loads(fernet.decrypt(account.encrypted_credentials.encode()).decode())

        # Если это Facebook Page, приоритет у page_access_token
        if account.platform == "facebook" and account.page_access_token_encrypted:
            page_blob = json.loads(fernet.decrypt(account.page_access_token_encrypted.encode()).decode())
            return page_blob["access_token"]

        expires_at = blob.get("expires_at", 0)
        # Если токен истекает через 60 секунд или уже истек:
        if time.time() > expires_at - 60 and blob.get("refresh_token"):
            blob = await refresh_oauth_token(account.platform, blob)
            account.encrypted_credentials = fernet.encrypt(json.dumps(blob).encode()).decode()
            account.expires_at = datetime.fromtimestamp(blob["expires_at"])
            # сохранить account в DB...

        return blob["access_token"]

def compute_appsecret_proof(app_secret: str, access_token: str) -> str:
    return hmac.new(app_secret.encode("utf-8"), access_token.encode("utf-8"), hashlib.sha256).hexdigest()
```

---

В следующих файлах каталога `1212/` представлен детальный разбор каждой социальной сети в отдельности.
