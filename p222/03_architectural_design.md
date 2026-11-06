# Отчёт 3: Архитектурное устройство систем и жизненный цикл запросов

Настоящий отчёт рассматривает архитектуру проектов CLIProxyAPI и freellmapi: топологию компонентов, жизненный цикл запроса (Request Lifecycle), пайплайны трансляции данных, подсистемы хранения и механизмы расширения/плагинов.

---

## 1. Архитектура CLIProxyAPI (Go)

Архитектура CLIProxyAPI спроектирована по принципам **модульного ядра с конвейерной обработкой (Pipeline-First Architecture)** и изоляцией ответственности через строгие интерфейсы в пакете `sdk/`.

```
[ Downstream Client: Claude Code / Codex CLI / Cursor / WebRTC ]
                         │
                         ▼
        ┌───────────────────────────────────┐
        │        Protocol Multiplexer       │
        │   (Gin Engine / Mux Listener)     │
        └─────────────────┬─────────────────┘
                          │
                          ▼
        ┌───────────────────────────────────┐
        │       Auth & Token Handlers       │
        │     (Internal Access Manager)     │
        └─────────────────┬─────────────────┘
                          │
                          ▼
        ┌───────────────────────────────────┐
        │      Dynamic Plugin Host ABI      │
        │  (Before-Auth Hooks, Byte-Patch)  │
        └─────────────────┬─────────────────┘
                          │
                          ▼
        ┌───────────────────────────────────┐
        │       Conductor / Scheduler       │
        │  (LCP Session Tree, Weight, Cool) │
        └─────────────────┬─────────────────┘
                          │
                          ▼
        ┌───────────────────────────────────┐
        │     Schema & Payload Pipeline     │
        │  (Gemini Schema, Cloaking, zstd)  │
        └─────────────────┬─────────────────┘
                          │
                          ▼
        ┌───────────────────────────────────┐
        │         Executor Subsystem        │
        │    (uTLS Transport, WebSockets,   │
        │      Bootstrap Stream Buffer)     │
        └─────────────────┬─────────────────┘
                          │
                          ▼
       [ Upstream Providers: Anthropic / OpenAI / Gemini ]
```

### 1.1. Ключевые архитектурные узлы
1. **Mux Listener & Buffered Conn (`internal/api/mux_listener.go`):**
   - Перехватывает входящие соединения до HTTP-обработчиков. Позволяет на одном сетевом порту мультиплексировать сырые TCP-соединения, TLS, стандартный HTTP/1.1, HTTP/2 и WebSocket.
2. **Conductor & Scheduler (`sdk/cliproxy/auth/conductor.go`, `selector.go`):**
   - «Дирижер» выполнения запросов. Реализует выбор учетных данных по стратегиям (`round-robin`, `weighted-round-robin`, `fill-first`).
   - Управляет деревом сессий (`SessionAffinitySelector`), отслеживает кулдауны по моделям/аккаунтам (`CooldownState`), обрабатывает повторные раунды (`RequestRetry`).
3. **Session LCP Engine (`sdk/cliproxy/session/lcp.go`, `identity.go`):**
   - Движок вычисления наибольшего общего префикса разговора на базе хешей Merkle Tree.
   - Различает ветвления диалогов (`is_fork`), подзадачи агентов (`is_subagent`) и компактификацию контекста (`is_compaction`).
4. **Plugin Host (`internal/pluginhost/`):**
   - Модуль изолированного исполнения внешних плагинов по стандартизированному ABI (`sdk/pluginabi`).
   - Предоставляет плагинам 20+ точек расширения (перехватчик до/после авторизации, байт-патчинг тел запросов, кастомные планировщики, мониторы потоков).
5. **Watcher & Config Hot-Reload (`internal/watcher/watcher.go`):**
   - Фоновый реактивный демон на базе `fsnotify`. Автоматически синхронизирует состояние в памяти с файловой системой или удаленными хранилищами.

---

## 2. Архитектура freellmapi (TypeScript / Node.js)

Архитектура freellmapi ориентирована на **сервисно-ориентированную модель (Service-Oriented Architecture)** поверх Express 5 и реляционного ядра SQLite.

```
[ Clients / Agents / MCP Clients / Desktop App ]
                       │
                       ▼
      ┌──────────────────────────────────┐
      │     Express Routing Surfaces     │
      │   /v1, /v1beta, /api, /mcp, /t   │
      └────────────────┬─────────────────┘
                       │
                       ▼
      ┌──────────────────────────────────┐
      │   Security & Client Middleware   │
      │  (HostGuard, Helmet, ClientCtx)  │
      └────────────────┬─────────────────┘
                       │
                       ▼
      ┌──────────────────────────────────┐
      │          Router Service          │
      │  (Scoring Engine, Bandit Presets,│
      │   Peak Hours, Model Groups)      │
      └────────────────┬─────────────────┘
                       │
                       ▼
      ┌──────────────────────────────────┐
      │          Fallback Loop           │
      │   (runFallbackLoop, Time Budget, │
      │    Key Leases, Rate Limiting)    │
      └────────────────┬─────────────────┘
                       │
                       ▼
      ┌──────────────────────────────────┐
      │       Provider Adaptors          │
      │   (Groq, Cerebras, Pollinations, │
      │    AI Horde, Custom Endpoints)   │
      └────────────────┬─────────────────┘
                       │
                       ▼
      ┌──────────────────────────────────┐
      │    Response & Rescue Layer       │
      │ (Tool Call Rescue, Stream Keep)  │
      └────────────────┬─────────────────┘
                       │
                       ▼
      ┌──────────────────────────────────┐
      │  SQLite DB (Encrypted Keys, Log) │
      └──────────────────────────────────┘
```

### 2.1. Ключевые архитектурные узлы
1. **Express Routing Engine (`server/src/app.ts`):**
   - Разделяет трафик на публичные/проксируемые интерфейсы (`/v1`, `/v1beta`), административную панель (`/api`), MCP-сервер (`/mcp`) и токенизированные прокси-ссылки (`/v1/t/:token`).
2. **Router & Scoring Engine (`server/src/services/router.ts`, `scoring.ts`):**
   - Сервис динамической оценки моделей. Рассчитывает апостериорную надежность, штрафует задержки (TTFB), проверяет остаток квот провайдеров (`provider-quota.ts`) и дневные лимиты Cloudflare Neurons.
3. **Fallback Loop Execution Engine (`server/src/lib/fallback-loop.ts`):**
   - Высокоуровневый оркестратор выполнения попыток. Контролирует общий бюджет времени запроса (`fallback_time_budget_ms`), изолирует ключи через систему аренд (`acquireLease` / `releaseLease`), формирует историю попыток (`x-freellmapi-attempt-trail`).
4. **Tool Call Rescue Subsystem (`server/src/lib/tool-call-rescue.ts`):**
   - Конвейерный санитайзер ответов, спасающий агентские сессии от падений при смене провайдера на лету.
5. **Decoupled Client & Desktop Layers (`client/`, `desktop/`):**
   - Архитектура полностью отделяет презентационный слой: Web UI на React общается через REST API, а Electron-оболочка запускает локальный backend-процесс как дочерний воркер (`desktop/src/server-host.ts`).

---

## 3. Сравнение архитектурных подходов

| Архитектурный аспект | CLIProxyAPI | freellmapi |
| :--- | :--- | :--- |
| **Парадигма конструирования** | Pipeline/Middleware + ABI Plugin Driven (Go). | Service-Layer / Express Routes Driven (TypeScript). |
| **Жизненный цикл стрима** | С буферизацией заголовков и сокрытием ошибок через Bootstrap Buffer. | С прямой трансляцией SSE и инъекцией heartbeats (`stream-keepalive`). |
| **Масштабируемость хранилища** | Локальная bbolt (zero-dependency) с поддержкой распределенных S3, PostgreSQL и Git. | Локальная SQLite (WAL mode, single-file DB), оптимизированная для локального узла/десктопа. |
| **Механизм расширения** | Внешние динамические плагины через IPC ABI (не требуют перекомпиляции ядра). | Внутренние TypeScript-модули провайдеров (`providers/*.ts`) в монорепозитории. |
| **Связанность с UI** | Минимальная: ядро является headless-шлюзом, UI — опциональная вспомогательная страница. | Высокая: бэкенд, SPA-клиент и Electron-обертка тесно связаны через единый контракт API. |
