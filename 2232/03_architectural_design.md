# Отчёт 3: Архитектурное устройство систем

Настоящий отчёт содержит всесторонний анализ архитектуры **deepseek-harness** и **opencode**, исследуя организацию кода, топологию монорепозиториев, слои абстракций, шины событий, модели взаимодействия компонентов и протоколы связи.

---

## 1. Архитектура deepseek-harness: Микроядро на базе Cordis IoC

### 1.1. Топология монорепозитория
Кодовая база deepseek-harness организована по принципу «мелкозернистых модулей» (fine-grained modularity) с 59 доменами пакетов в `packages/*` и приложениями в `apps/*`:
- `apps/cli`: единая точка входа для консольного запуска, сборки профилей и CLI-интерфейса.
- `apps/desktop` и `apps/desktop-host`: Electron-приложение с хост-процессом Node.js.
- `apps/web`: веб-клиент на React/Vite.
- `packages/core/*`: фундаментальные строительные блоки — цикл агента (`agent-loop`), системные промпты (`system-prompt`), реестр инструментов (`tools`), модель по умолчанию (`agent-default-model`).
- `packages/subagent/*`, `packages/team/*`, `packages/workflow/*`: уровни оркестрации.
- `packages/boot/*`: фазы загрузки, компиляция профилей, валидация зависимостей.

### 1.2. Модель контекстов Cordis и шина событий
Архитектура строится вокруг иерархического контекста:
```
                       [Root Cordis Context]
                                 │
         ┌───────────────────────┼────────────────────────┐
         │                       │                        │
  [Core Services]       [Session Context]        [Tool Registry]
  - LLM Provider         - Event Log              - Native Tools
  - Storage Backend      - State Projections      - PTC Isolated Tools
  - Sandbox Policy       - Inbox Queue            - MCP Adapters
                                 │
                 ┌───────────────┴───────────────┐
                 │                               │
        [Child Fork Context]           [Child Fork Context]
        (Subagent / Team Member)       (Workflow PTC Realm)
```
- **Связывание через сервисы**: Сервисы объявляются через `declare module '@deepseek-ai/cordis' { interface Context { ... } }`.
- **Событийная модель**: Компоненты общаются через типизированную шину событий (`ctx.emit('session/event', ...)`), что обеспечивает полную развязку модулей: подписчики могут собирать телеметрию, обновлять UI или логировать события без жестких ссылок на издателя.

### 1.3. Слой представления и изоляции (PTC Architecture)
- Уникальный архитектурный слой — разделение на **Host** (хост-среда Node.js, имеющая доступ к диску и сети) и **Guest** (песочница `vm.createContext`).
- Гостевой код (`guest-source.ts`, `realm.ts`) получает безопасный контекст `dsh`, содержащий только разрешенные методы (`dsh.agent()`, `dsh.parallel()`, `dsh.tool()`). Все вызовы прозрачно сериализуются через IPC/PTC JSON-канал.

---

## 2. Архитектура opencode: Функциональная сервисно-ориентированная архитектура на Effect-ts

### 2.1. Топология монорепозитория
Монорепозиторий opencode организован по функционально-слоистому принципу:
- `packages/schema`: единый источник правды для всех типов данных системы (Entity schemas, DTO, API contracts) на базе `Effect.Schema`.
- `packages/ai`: автономный протокольный шлюз к нейросетям (Streaming, Protocols, Providers, Image/Audio/Video engines).
- `packages/core`: бизнес-логика (Sessions, Agents, Workspace, VCS/Git, FileSystem, Permissions, Jobs, PTY).
- `packages/server`: HTTP/WebSocket сервер (Hono/Workerd/Node) для связи CLI, Desktop и Web.
- `packages/protocol`: RPC-клиент и протоколы обмена сообщениями между сервером и клиентами.
- `packages/cli`, `packages/tui`, `packages/app`, `packages/desktop`: слои представления.

### 2.2. Сервисная модель Layers & Dependency Graph
Вся архитектура opencode представляет собой направленный ациклический граф сервисов (DAG of Layers):
```
                       [Config.Service]
                              │
               ┌──────────────┴──────────────┐
               │                             │
       [Database.Service]             [LLMClient.Service]
       (Drizzle / SQLite)             (@opencode/ai)
               │                             │
       [SessionStore.Service]        [ToolRuntime.Service]
               │                             │
               └──────────────┬──────────────┘
                              │
                    [SessionRunner.Service]
                              │
                 ┌────────────┴────────────┐
                 │                         │
         [Bus.Service]             [Job.Service]
         (Event Stream)            (Background Fibers)
```
- **Чистота и эффекты**: Каждая функция декларирует требуемые сервисы в типе возвращаемого значения: `Effect.Effect<Output, Error, Session.Service | Job.Service>`.
- **Жизненный цикл ресурсов (Scope)**: Создание сокетов, транзакций БД и подпроцессов оборачивается в `Scope.Scope`, что обеспечивает гарантированную очистку при ошибке или прерывании родительского контекста.

### 2.3. Клиент-серверная шина и сетевой слой
- opencode изначально спроектирован как распределенная система:
  - Сервер (`@opencode/server`) может работать локально или удаленно.
  - Клиенты (`@opencode/tui`, `@opencode/web`, `@opencode/desktop`, `@opencode/sdk`) подключаются по HTTP REST и WebSocket Event Feed.
  - Все мутации состояния инициируются через типизированные RPC-эндпоинты (`@opencode/protocol`).

---

## 3. Сравнительный анализ архитектурных паттернов

| Архитектурный аспект | deepseek-harness | opencode |
| :--- | :--- | :--- |
| **Паттерн проектирования** | Микроядро (Microkernel) + Plugin Container (Cordis IoC). | Сервисно-слоистая архитектура (Layered Services) + Algebra of Effects. |
| **Связанность компонентов (Coupling)** | Крайне слабая (Loose coupling via Context & Events). Плагины могут быть удалены без падения сборки. | Сильная статическая типизация зависимостей через типы TypeScript и `Context.Service`. |
| **Управление состоянием** | Event Sourcing + In-memory Projections. | Реляционная СУБД (SQLite / Drizzle) + Transactional State Machine. |
| **Управление конкурентностью** | Node.js Event Loop + Promisified locks (`ChildLock`, `ActivationPool`). | Легковесные волокна (Fibers) + Structured Concurrency (`Effect.forkIn`, `Scope`). |
| **Механизм расширения** | Плагины Cordis с хуками жизненного цикла и HMR. | Кастомные инструменты, внешние плагины и кастомные Agent Definitions. |

---

## 4. Выводы по третьему отчёту
- **deepseek-harness** воплощает гибкую архитектуру расширяемого плагинного хоста. Она позволяет собирать различные типы рабочих окружений (CLI, Headless, ACP-агент, Desktop) простой комбинацией независимых пакетов-плагинов в профиле.
- **opencode** реализует строгую промышленную enterprise-архитектуру с гарантированной типобезопасностью на всех уровнях от базы данных до веб-сокетов, исключающую непредвиденные сбои в продакшене благодаря математически строгому контролю эффектов в Effect-ts.
