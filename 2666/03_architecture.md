# Отчёт 3: Архитектурное устройство систем

В данном отчёте рассматривается общая системная архитектура обоих проектов: декомпозиция на слои, ключевые архитектурные паттерны, потоки данных (data flow), клиент-серверное взаимодействие и модели хранения состояния.

---

## 1. Архитектура `grok-build`

### 1.1. Архитектурная диаграмма слоев `grok-build`

```
+---------------------------------------------------------------------------------+
|                                USER INTERFACE                                   |
|   CLI Shell / Interactive Terminal (xai-grok-shell, xai-ratatui-inline)         |
+---------------------------------------------------------------------------------+
                                       |
                                       v
+---------------------------------------------------------------------------------+
|                            AGENT ORCHESTRATION LAYER                            |
|  - AgentBuilder & Agent (xai-grok-agent)                                        |
|  - Prompt Queue & Merging (xai-prompt-queue)                                    |
|  - Subagent Resolution & Lifecycle (xai-grok-subagent-resolution, lifecycle)    |
|  - Compaction Transcript (xai-compaction-transcript, xai-grok-compaction)        |
+---------------------------------------------------------------------------------+
          |                                                   |
          v                                                   v
+-----------------------------------+   +-----------------------------------------+
|     CONFIGURATION & POLICIES      |   |              MEMORY V2                  |
| - Layered Config Engine           |   | - Embedding & Search Pipeline           |
| - Ed25519 Signed Policy           |   | - Temporal Decay & MMR                  |
| - macOS MDM Preferences           |   | - Dream Consolidation Service           |
| - Permission Rule Engine          |   +-----------------------------------------+
+-----------------------------------+
          |
          v
+---------------------------------------------------------------------------------+
|                       TOOL RUNTIME & EXECUTION BOUNDARY                         |
|  - ToolBridge (Mutex-protected tool state)                                      |
|  - xai-tool-runtime & Protocol (Dispatch, Context, Streaming)                   |
|  - MCP Adapter & rmcp Quarantined Boundary (xai-grok-mcp)                       |
+---------------------------------------------------------------------------------+
          |                                                   |
          v                                                   v
+-----------------------------------+   +-----------------------------------------+
|     OS-LEVEL SANDBOX (nono)       |   |             PTY CONTROLLER              |
| - Linux Landlock (FS bounds)      |   | - Headless Alacritty Terminal Engine    |
| - Seccomp (Child network disable) |   | - Styled/VT Screen Capture              |
| - macOS Seatbelt profiles         |   | - ptyctl REST API Daemon                |
| - Egress HTTP/CONNECT Proxy       |   +-----------------------------------------+
+-----------------------------------+
```

### 1.2. Архитектурные принципы `grok-build`
1. **Compacted Core / Deep Modularization:** Система разделена на узкоспециализированные крейты с четкими границами ответственности. Исключены циклические зависимости.
2. **Fail-Closed Security Boundaries:** Архитектура безопасности не полагается на подсказки в системных промптах. Права доступа на чтение файлов, сеть и выполнение команд проверяются в runtime на стыке `ToolBridge` -> `xai-grok-sandbox` -> ядро ОС.
3. **Data-only Lifecycle Hooks (`xai-agent-lifecycle`):** Модули расширения получают строго данные только для чтения (`LocalTurnInputContributor`, `LocalSessionLifecycleContributor`) и никогда не владеют циклом управления (control loop) основного агента.
4. **Хранилище сессий:** SQLite-журнал (`xai-sqlite-journal`) для транзакционной фиксации событий диалога и состояния сессий.

---

## 2. Архитектура `opencode`

### 1.1. Архитектурная диаграмма слоев `opencode`

```
+---------------------------------------------------------------------------------+
|                             CLIENT INTERFACES                                   |
|     CLI (packages/cli)   |   TUI (packages/tui)   |   Web UI / Desktop (app)   |
+---------------------------------------------------------------------------------+
                                       |
                     HTTP / REST / SSE / RPC Transport
                                       v
+---------------------------------------------------------------------------------+
|                           SERVER & API RUNTIME LAYER                            |
|             packages/server (routes.ts, process.ts, event-feed.ts)               |
+---------------------------------------------------------------------------------+
                                       |
+---------------------------------------------------------------------------------+
|                         EFFECT SERVICE GRAPH (packages/core)                    |
|                                                                                 |
|  +----------------------------+             +--------------------------------+  |
|  |     SessionRunCoordinator  | <---------> |        Session.Service         |  |
|  | (Exclusive Fiber Execution)|             |  (Session Lifecycle & Prompts) |  |
|  +----------------------------+             +--------------------------------+  |
|               |                                              |                  |
|               v                                              v                  |
|  +----------------------------+             +--------------------------------+  |
|  |       Job.Service          |             |     SessionHistory / Storage   |  |
|  |  (SubagentJob / Background)|             |  (SQLite via Drizzle / Effect) |  |
|  +----------------------------+             +--------------------------------+  |
|               |                                              |                  |
|               v                                              v                  |
|  +----------------------------+             +--------------------------------+  |
|  |       Event Bus (Bus)      |             |         Agent.Service          |  |
|  |  (Realtime State Broadcast)|             |    (Definitions, Autocomplete) |  |
|  +----------------------------+             +--------------------------------+  |
+---------------------------------------------------------------------------------+
          |                                                   |
          v                                                   v
+-----------------------------------+   +-----------------------------------------+
|         AI PROVIDER LAYER         |   |         TOOL & CODE MODE ENGINE         |
|  - @opencode/ai (Unified Client)  |   | - Tool Registry & Runtime               |
|  - Model Adapters (OpenAI, Claude)|   | - Code Mode Sandbox (JS execution)      |
|  - AISDK Native Integration       |   | - MCP Client Manager                    |
|  - Token Tracker & Context Overflow|   | - Persistent PTY Daemon (Multi-runtime) |
+-----------------------------------+   +-----------------------------------------+
```

### 2.2. Архитектурные принципы `opencode`
1. **Реактивный сервисный граф на Effect:** Вся функциональность инкапсулирована в сервисах Effect, связанных через слои (`Layer`). Это обеспечивает легкую заменяемость реализаций (например, подмена реализации файловой системы для тестирования или переход на edge runtime).
2. **Централизация вокруг `SessionRunCoordinator`:** Защита от состояния гонки (race conditions) при параллельных обращениях пользователя и фоновых процессах за счет сериализации исполнения в рамках файбера с сохранением неблокирующего интерфейса.
3. **Единый Event Feed:** Любые мутации в ядре транслируются через `Bus.Service` и немедленно доставляются клиентам через Server-Sent Events (SSE).
4. **Двухуровневая модель исполнения инструментов:** Поддержка традиционных изолированных вызовов и интеграции через `Code Mode`, где рантайм компилирует код вызовов в пакетные операции.

---

## 3. Сравнительный анализ архитектурных паттернов

| Архитектурный аспект | `grok-build` | `opencode` |
|---|---|---|
| **Управление состоянием (State Management)** | Неизменяемые структуры с атомарными флагами и изолированными мьютексами в `ToolBridge`. Транзакционный лог в SQLite. | Реактивные ссылки (`Ref`), структуры Effect State и реляционная база SQLite через Drizzle ORM. |
| **Коммуникация между компонентами** | Системные каналы `tokio::sync::mpsc`, `broadcast`, прямые вызовы трейтов. | Effect Hub / `Bus.Service`, Effect Queue, типизированные события `Event`. |
| **Клиент-серверное расщепление** | Монолитный CLI с отдельными микродемонами (`ptyctl`, `egress-proxy`, `diag-server`). | Полноценный API-сервер (`packages/server`), готовый к размещению отдельно от UI (Web, Desktop, Remote Worker). |
| **Изоляция сбоев (Fault Tolerance)** | `xai-circuit-breaker`, строгие лимиты буферов памяти, изоляция паники в тредах. | Обработка ошибок в системе типов Effect (`Effect<Success, Failure>`), отсутствие необработанных исключений. |
