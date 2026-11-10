# Сводная таблица и навигация по отчетам: Автопостинг в соцсети

Данный каталог содержит подробные технические отчеты по результатам анализа исходного кода репозитория `treg` (`src/treg/oauth_providers.py`, `src/treg/catalog/`, `src/treg/application/connect.py`, `src/treg/infra/`).

Отчеты структурированы по отдельным файлам для каждой социальной сети с описанием процесса авторизации, форматов публикации контента, специфических ограничений API и готовых примеров кода на Python.

---

## Навигация по файлам отчетов

| Файл | Социальная сеть / Тема | Ключевые аспекты |
|---|---|---|
| **[01_ARCHITECTURE_OVERVIEW.md](./01_ARCHITECTURE_OVERVIEW.md)** | **Общая архитектура системы** | Жизненный цикл OAuth 2.0, безопасное хранение в БД, механизм `oauth_refresh` (фоновое продление токенов), прозрачная инъекция заголовков (`oauth_injector`), защита Meta `appsecret_proof`, пошаговое руководство по созданию аналогичной системы в своем проекте. |
| **[02_FACEBOOK.md](./02_FACEBOOK.md)** | **Facebook Pages** | OAuth 2.0, Long-Lived Token Exchange, обязательное извлечение **Page Access Token** через `/me/accounts` (User Token блокируется ошибкой 190), публикация текстовых постов (`/{page_id}/feed`), фото/альбомов (`/{page_id}/photos`), видео/Reels (`/{page_id}/videos`), редактирование и удаление. |
| **[03_INSTAGRAM.md](./03_INSTAGRAM.md)** | **Instagram** | Две модели авторизации: прямой Instagram Login (`instagram-login`) и через Facebook Page (`facebook-page`). Двухэтапный Container API (создание контейнера -> ожидание статуса -> `media_publish`), карусели до 10 элементов, Reels, лимит 100 постов в сутки, требование к прямым публичным URL для медиа. |
| **[04_TIKTOK.md](./04_TIKTOK.md)** | **TikTok** | Login Kit v2, параметр `client_key` вместо `client_id`, разделитель скоупов запятая `,`, PKCE. Разделение режимов: Direct Post (`video.publish`) vs Входящие черновики (`video.upload`). Ограничения неаудированных приложений (`SELF_ONLY` + приватный аккаунт). Чанковая загрузка видео (`upload_url`) и опрос статуса публикации (`publish_id`). |
| **[05_X_TWITTER.md](./05_X_TWITTER.md)** | **X (Twitter)** | OAuth 2.0 с обязательным PKCE, обязательный HTTP Basic Auth для обмена и продления токенов, скоуп `offline.access` для автопродления каждые 2 часа. Создание твитов `POST /2/tweets`, треды (`in_reply_to_tweet_id`), цитирование, опросы. Платная тарификация pay-per-use ($0.015 за твит, $0.200 за твит со ссылкой). Ограничения на загрузку медиа (требует отдельный скоуп `media.write`). |
| **[06_YOUTUBE.md](./06_YOUTUBE.md)** | **YouTube** | Google OAuth 2.0, скоупы `youtube.upload` vs `youtube` (manage). Протокол Resumable Upload (инициализация сессии `POST /upload/...` -> передача бинарных байтов `PUT Location`), установка обложек (`thumbnails/set`), жесткая система квот (10 000 units/день, загрузка видео стоит 1 600 units -> макс 6 видео/день), блокировка статуса `private` до аудита проекта. |
| **[07_LINKEDIN.md](./07_LINKEDIN.md)** | **LinkedIn** | OAuth 2.0, скоуп `w_member_social`. Отсутствие refresh token на бесплатном уровне (токен живет ровно 60 дней). Обязательные заголовки `LinkedIn-Version` и `X-Restli-Protocol-Version: 2.0.0`. Трехшаговый алгоритм публикации медиа (`initializeUpload` -> бинарный `PUT` на CDN -> прикрепление URN в `POST /rest/posts`). Экранирование спецсимволов `()[]{}<>@|~`. |
| **[08_SLACK_AND_OTHERS.md](./08_SLACK_AND_OTHERS.md)** | **Slack, Ads и агрегаторы** | Slack: модель "Bring-Your-Own-Bot" с предзаполненным манифестом (`chat:write`), проверка через `/auth.test`. Рекламные кабинеты: Pinterest Ads, TikTok Ads, Meta Ads. Внешние социальные шлюзы: AnyAPI, TikHub, ScrapeCreators. |

---

## Сравнительная матрица возможностей автопостинга

| Платформа | Метод авторизации | Срок жизни токена | Авто-продление | Форматы контента для автопостинга |
|---|---|---|---|---|
| **Facebook Pages** | OAuth 2.0 (Page Token) | 60 дней (долгоживущий) | Да (через повторный обмен) | Текст, ссылки, фото, альбомы/карусели, видео, Reels, планирование постов |
| **Instagram** | OAuth 2.0 (Direct / FB Page) | 60 дней | Да (через `refresh_access_token`) | Фото, карусели (2–10 фото/видео), Reels, Stories (только через публичный URL) |
| **TikTok** | OAuth 2.0 (Login Kit v2 + PKCE) | ~1 сутки (access), refresh 365 дн. | Да (через `client_key` refresh) | Видео напрямую в профиль, видео во входящие черновики, фотопосты/карусели |
| **X (Twitter)** | OAuth 2.0 PKCE (Basic Auth) | 2 часа | Да (со скоупом `offline.access`) | Текст (280 / 25k), треды/ответы, цитирование, опросы (медиа требует `media.write`) |
| **YouTube** | Google OAuth 2.0 | 1 час | Да (через Google refresh token) | Видео (Resumable Upload), кастомные превью (обложки), плейлисты |
| **LinkedIn** | OAuth 2.0 | 60 дней | Нет (требует ручного реконнекта) | Текст со ссылками, фото (до 9 шт), PDF-документы (карусели), видео |
| **Slack** | Bot Token (`xoxb-...`) | Бессрочный (до отзыва) | Не требуется | Сообщения в каналы, треды, файлы и документы |
