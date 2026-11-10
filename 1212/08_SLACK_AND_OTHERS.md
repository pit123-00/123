# Отчет по другим социальным сервисам и рекламным сетям в `treg`

В кодовой базе `treg`, помимо основных социальных сетей с OAuth-авторизацией (Facebook, Instagram, TikTok, X, YouTube, LinkedIn), реализованы интеграции со смежными социальными и рекламными платформами. Ниже представлен анализ их подключения и возможностей.

---

## 1. Slack (Командные каналы и сообщества)

Файлы в кодовой базе `treg`: `src/treg/oauth_providers.py` (`SLACK`).

### Модель подключения: "Bring-Your-Own-Bot"
В отличие от соцсетей общего доступа, подключение к Slack не использует централизованное OAuth-приложение `treg`.
В коде задокументирована причина:
> Slack-бот принадлежит конкретному воркспейсу (Workspace-scoped). Общее приложение третьей стороны создавало бы риск безопасности, читая внутренние сообщения компании.
> Поэтому `treg` использует модель **"Bring-Your-Own-Bot"**: пользователь в один клик создает бота в своем воркспейсе через предзаполненный JSON-манифест и вставляет `Bot User OAuth Token` (`xoxb-...`).

### Предзаполненный манифест приложения Slack:
В URL создания (`setup_url`) передается конфигурация с готовыми правами:
- Публикация сообщений: `chat:write`, `chat:write.public` (позволяет боту писать в любые публичные каналы без предварительного приглашения).
- Чтение каналов и истории: `channels:read`, `groups:read`, `im:read`, `mpim:read`, `channels:history`, `groups:history`.
- Реакции и файлы: `reactions:read`, `reactions:write`, `files:read`, `app_mentions:read`.

### Проверка валидности (Health Probe)
- Эндпоинт: `POST https://slack.com/api/auth.test` с заголовком `Authorization: Bearer xoxb-...`.
- *Ловушка Slack API*: для невалидного токена Slack возвращает `HTTP 200 OK` с телом `{"ok": false, "error": "invalid_auth"}`. Поэтому проверка валидности в коде `treg` проверяет JSON-поле `ok == true` (`token_verify_field="ok"`).

### Возможности постинга:
- `POST https://slack.com/api/chat.postMessage` — отправка сообщений, форматирование через Block Kit или Markdown, треды через `thread_ts`.
- `POST https://slack.com/api/files.upload` — загрузка картинок, документов и логов в канал.

---

## 2. Рекламные платформы социальных сетей (Social Ads)

В кодовой базе `treg` имеются провайдеры для рекламных кабинетов:
- `META_ADS` (Meta Ads: Facebook & Instagram Ads)
- `TIKTOK_ADS` (TikTok for Business)
- `PINTEREST_ADS` (Pinterest Ads API)
- `SNAPCHAT_ADS` (Snapchat Marketing API)

### Особенности авторизации Ads-провайдеров:
1. **Pinterest Ads**:
   - `auth_uri`: `https://www.pinterest.com/oauth/`
   - `token_uri`: `https://api.pinterest.com/v5/oauth/token`
   - Как и у X, эндпоинт обмена токенов Pinterest **требует аутентификации через HTTP Basic Auth** (`Authorization: Basic base64(client_id:client_secret)`).
   - Скоупы: `ads:read`, `billing:read`.
2. **TikTok Ads**:
   - `base_url`: `https://business-api.tiktok.com/open_api/v1.3`
   - В отличие от пользовательского TikTok API, передает токен в заголовке `Access-Token: {token}` (а не `Authorization: Bearer`).
3. **Meta Ads**:
   - Тот же host `graph.facebook.com`, но скоуп `ads_read` / `ads_management` и двухэтапный выбор рекламного аккаунта `act_{ad_account_id}`.

### Возможности взаимодействия:
В текущей версии каталога `treg` рекламные API каталогизированы преимущественно для **чтения аналитики кампаний, объявлений, затрат и таргетингов** (`campaigns`, `adgroups`, `ads`, `analytics`), а не для создания рекламных объявлений.

---

## 3. Внешние социальные шлюзы и парсеры (AnyAPI, TikHub, ScrapeCreators)

Для социальных сетей, где официальный OAuth либо не предоставляет публичных данных (Reddit, Threads, Telegram, Douyin, Xiaohongshu, Weibo, Bilibili, Kuaishou), либо где требуется сбор информации без авторизации аккаунта, `treg` интегрирует агрегаторы:
- **AnyAPI**: 362 нормализованных эндпоинта по социальным сетям (посты, профили, хэштеги в Facebook, Instagram, TikTok, YouTube, Threads).
- **TikHub**: глубокий доступ к азиатским и глобальным сетям (TikTok, Douyin, Xiaohongshu, Weibo, Bilibili, Telegram channels).
- **ScrapeCreators**: публичные данные инфлюенсеров и постов в 15+ платформах.

Эти провайдеры работают по фиксированному API-ключу (`uses_pasted_secret = True`) и используются для аналитики и мониторинга, а не для автопостинга от имени пользователя.
