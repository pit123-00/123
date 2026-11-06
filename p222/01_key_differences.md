# Отчёт 1: Ключевые отличия между CLIProxyAPI и freellmapi

Настоящий отчёт подготовлен исключительно на основе анализа исходного кода обоих репозиториев (Go в `CLIProxyAPI` и TypeScript/Node.js в `freellmapi`) без обращения к описательной документации в README.

---

## 1. Концептуальное позиционирование и базовая философия

| Критерий | CLIProxyAPI (v8) | freellmapi |
| :--- | :--- | :--- |
| **Основной технологический стек** | **Go (Golang)** (Gin Web Framework, net/http, uTLS, gorilla/websocket, bbolt, minio-go). | **TypeScript / Node.js** (Express 5, better-sqlite3 / node:sqlite, vitest, Vite/React, Electron). |
| **Целевое назначение** | **Высокопроизводительный Enterprise-шлюз и протокольный мультиплексор/транслятор** для CLI-инструментов разработки (Claude Code, OpenAI Codex CLI, Devin, Kimi, Gemini CLI, Antigravity) с авторизацией через OAuth-токены аккаунтов и пулы ключей. | **Smart-маршрутизатор и агрегатор бесплатных провайдеров** (Groq, Cerebras, Pollinations, DeepInfra, HuggingFace, Cloudflare, AI Horde, Glif и др.) с графической панелью (Web Dashboard), десктопным приложением (Electron tray) и MCP-сервером. |
| **Модель управления состоянием** | **Декларативно-реактивная на файлах и памяти**: конфигурация в YAML (`config.yaml`), токены и OAuth-сессии в файлах директории `auths/`, локальный bbolt KV-кэш или распределённые хранилища (S3/PostgreSQL/Git). Динамический `fsnotify`-вотчер перечитывает файлы на лету без перезапуска процесса. | **Реляционная в базе данных**: локальная SQLite (`freeapi.db`) с миграциями через runner. Модели, ключи, лимиты, кулдауны, статистика запросов, логи и конфигурации хранятся в таблицах SQLite. Переменные окружения задаются через `.env`. |
| **Потребительский интерфейс** | TUI (Terminal UI), легковесный встраиваемый HTML Management Panel, REST API Management (`/v8/management/*`), mDNS Discovery для локальной сети. | Полноценный SPA-клиент на React/Vite/Tailwind (`/client`), десктопный клиент на Electron с системным треем (`/desktop`), CLI-утилита проверки окружения (`/cli`), Web Dashboard (`/api/*`). |

---

## 2. Глубокие различия в конфигурации и настройках

### 2.1. CLIProxyAPI (`internal/config/config.go`, `config_types.go`, `sdk_config.go`)

Конфигурация CLIProxyAPI рассчитана на экстремальный контроль за низкоуровневым сетевым взаимодействием, заголовками и маскировкой клиентов:

1. **Маскировка клиентов (Cloaking Engine):**
   - Параметр `cloak`: режимы `auto`, `always`, `never`, `uncloak`.
   - Позволяет сторонним клиентам выдавать себя за официальный CLI (например, Claude Code CLI или ChatGPT Codex).
   - `strict-mode`: вырезает пользовательские системные промпты, оставляя исключительно биллинговые и идентификационные заголовки Claude Code.
   - `sensitive-words`: обфускация запрещённых слов в системных инструкциях с помощью нуль-ширинных символов (Zero-Width Characters) для обхода цензуры upstream-провайдеров.

2. **Тонкая настройка сетевых протоколов и буферизации стримов:**
   - `stream-bootstrap-buffering`: перехват и задержка передачи первых чанков стрима (`response.created`, `response.in_progress`, пустые события) до момента появления первого реального сгенерированного токена. Это позволяет перехватить скрытую ошибку `server_is_overloaded` от ChatGPT внутри HTTP 200 и незаметно переключить запрос на другой аккаунт из пула без падения клиентской сессии.
   - `stream-bootstrap-timeout`: таймаут bootstrap-буферизации (например, 20s), спасающий от тайм-аутов обратных прокси (Nginx `proxy_read_timeout`).
   - `tls-handshake-timeout-seconds`: таймаут uTLS-рукопожатия на уровне сырого сокета для предотвращения зависания при блокировках соединений.
   - `upstream-request-compression`: сжатие тел запросов через zstd для ChatGPT Codex.
   - `websocket-compression`: управление сжатием кадров permessage-deflate в полнодуплексных WebSocket-каналах.

3. **Сложная многоуровневая ротация и кулдауны:**
   - `model-level-cooling`: изоляция ошибки квоты (`usage_limit_reached`) конкретной моделью, не замораживая всю учётную запись целиком.
   - `request-retry`, `max-retry-credentials`, `max-retry-interval`: многокруговая ротация с ограничением перебираемых учетных данных за раунд.
   - `antigravity-credits`: каскадный фолбэк на платные кредиты Google One AI при исчерпании бесплатных квот Claude/Gemini.
   - `oauth-request-scoped-errors`: настраиваемые правила перехвата ошибок по статусам (например, 400), регулярным выражениям текста ошибки с действиями (`stop`, `stop-and-cooldown`, `continue`, `continue-and-cooldown`).

### 2.2. freellmapi (`server/src/env.ts`, `server/src/services/router.ts`, `server/src/services/scoring.ts`)

Конфигурация freellmapi ориентирована на управление пулом разнородных провайдеров, алгоритмическую маршрутизацию и квотирование:

1. **Алгоритмический роутер (Multi-Armed Bandit / Весовые скоринги):**
   - Стратегии: `balanced` (0.5 надежность, 0.25 скорость, 0.25 интеллект), `smartest` (0.55 интеллект), `fastest` (0.55 скорость), `reliable` (0.7 надежность), `priority` (ручной приоритет по списку).
   - Динамический учет времени суток (`peak-hours`): автоматический сдвиг весов со скорости на надежность в часы пиковой нагрузки в заданном IANA часовом поясе.
   - Разделение по типу задач (`task: 'code' | 'chat'`): калибровка приоритетов в зависимости от характера запроса.
   - Thompson Sampling / Beta-распределение: апостериорный расчет вероятности успеха (`sampleBeta(successes + 1, failures + 1)`).

2. **Экономика токенов и лимиты:**
   - `PROXY_RATE_LIMIT_RPM`: ограничение частоты запросов на клиента.
   - `monthly_budget` и `rate_limit_rpm` для каждого отдельного ключа и провайдера.
   - Отслеживание Cloudflare Neurons (суточный лимит 10,000 нейронов) и автоматическое отсечение запросов при исчерпании суточной квоты.
   - Точный учет TTFB (Time to First Token) и скорости генерации токенов в секунду (`tok/s`).

3. **Служебные параметры среды:**
   - `ENCRYPTION_KEY`: симметричный AES-256 ключ шифрования для безопасного хранения провайдерских API-ключей в SQLite.
   - `STREAM_KEEPALIVE_MS`: периодическая отправка пустых SSE-комментов (heartbeat) для предотвращения отключения агентов (например, Claude Code watchdog) во время долгих рассуждений reasoning-моделей.

---

## 3. Поддерживаемые протоколы и API-поверхности

### CLIProxyAPI:
- **OpenAI Compatible:** `/v1/chat/completions`, `/v1/completions`, `/v1/models`, `/v1/images/*`, `/v1/audio/*`, `/v1/videos/*`.
- **OpenAI Responses (Codex):** `/v1/responses` (HTTP и полнодуплексный WebSocket), `/v1/responses/compact`, `/v1/alpha/search`.
- **Realtime API (WebRTC & WebSockets):** `/v1/realtime`, `/v1/realtime/calls/*`, `/v1/realtime/sessions`, WebRTC Media Relay для аудио/видео стримов в реальном времени.
- **Anthropic Claude Native:** `/v1/messages`, `/v1/messages/count_tokens`.
- **Google Gemini / Interactions Native:** `/v1beta/models/*`, `/v1beta/interactions`.
- **Ollama Protocol Multiplexing:** поддержка формата Ollama через динамический мультиплексор заголовков и путей.
- **Dynamic Plugin API:** бинарный протокол плагинов через gRPC/IPC (`pluginhost/abi.go`).

### freellmapi:
- **OpenAI Compatible:** `/v1/chat/completions`, `/v1/models`, `/v1/embeddings`.
- **Anthropic Messages:** `/v1/messages` (транслируется в OpenAI-совместимый бэкенд через `anthropic-map.ts`).
- **Google Gemini:** `/v1beta/models/*` (транслируется через `gemini-map.ts`).
- **Ollama Endpoints:** `/api/tags`, `/api/show`, `/api/generate`, `/api/chat` для прозрачной интеграции с локальными клиентами.
- **Model Context Protocol (MCP):** `/mcp` (Streamable HTTP, JSON-RPC 2.0) с готовыми инструментами для агентов (`ask_freellmapi`, `healthcheck`, `list_models`, `provider_health`, `routing_info`, `set_routing_strategy`).
- **Media & Embeddings Services:** поиск картинок, генерация, эмбеддинги через HuggingFace/Jina.

---

## 4. Сравнительная сводная таблица функциональности

| Область | CLIProxyAPI | freellmapi |
| :--- | :--- | :--- |
| **Язык разработки** | Go | TypeScript (Node.js) |
| **Ключевой фокус** | Обход ограничений CLI (Claude/Codex), эмуляция официальных клиентов, агрегация платных/OAuth аккаунтов. | Агрегация сотен открытых/бесплатных API, оптимизация затрат, распределение нагрузки. |
| **Управление сессиями агентов** | LCP (Longest Common Prefix) Merkle Tree, привязка родитель/потомок (`subagent`), детекция разветвлений (`is_fork`). | `context-handoff.ts`, привязка сессий по `session_id`, сжатие контекста (`compression/`). |
| **Восстановление вызовов инструментов** | Жесткая нормализация схем (`gemini_schema.go`, 82 КБ кода валидации и санитизации). | Эвристический парсер `tool-call-rescue.ts` (восстановление XML, DeepSeek/Kimi токенов, markdown JSON). |
| **Поддержка MCP (Model Context Protocol)** | Нет встроенного MCP-сервера (ориентирован на клиенты CLI). | Полноценный stateless MCP-сервер (`/mcp`) с набором из 14 инструментов диагностики и выполнения. |
| **Шифрование данных** | На уровне файловой системы / базы bbolt (без принудительного симметричного шифрования полей). | Обязательное шифрование ключей в SQLite с помощью AES-GCM (`ENCRYPTION_KEY`). |
| **Плагинная расширяемость** | Мощная архитектура внешних плагинов (`pluginapi`, `pluginhost`) с возможностью перехвата запросов, стримов, AST байт-патчинга. | Архитектура внутренних модулей-провайдеров (`providers/*.ts`) внутри исходного кода Node.js. |
