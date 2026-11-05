# Отчёт №1: Техническая реализация репозитория bb (get-bb/bb)

## 1. Введение и область анализа

Данный отчёт подготовлен на основе прямого глубокого анализа исходного кода репозитория **bb** (`get-bb/bb`). Анализ проведён исключительно по кодовой базе (файлы TypeScript/JavaScript в `packages/`, `apps/`, `plugins/`), без использования README или сторонних маркетинговых описаний.

Проект представляет собой распределённую среду исполнения и оркестрации автономных AI-агентов (таких как Claude Code, Codex, Pi, Cursor ACP и др.), поддерживающую удалённые и локальные машины (хосты), управление изолированными рабочими пространствами (workspaces), выполнение инструментов (tools), отслеживание системных событий и стриминг дельт пользовательского интерфейса.

---

## 2. Основные модули и устройство ключевых компонентов

Кодовая база организована как pnpm/turbo монорепозиторий со строгой слоистой моделью зависимостей.

### 2.1. `@bb/domain` (`packages/domain`) — Доменная модель и контракты валидации
Пакет `@bb/domain` служит ядром контрактов системы. Все типы определяются через схемы **Zod**, обеспечивая runtime-валидацию и TypeScript infer-типизацию без рассинхронизации.
- **Потоки (Threads) и Состояния**: `thread.ts`, `thread-status.ts`, `thread-runtime-state.ts`.
  - Статусы потоков: `idle`, `active`, `pending`, `held`.
  - Динамическое состояние рантайма: `ThreadRuntimeDisplayStatus` связывает состояние хоста, готовность окружения и статус выполнения агента.
- **События потока (Thread Events)**: `thread-events.ts`, `stored-thread-event.ts`.
  - События дискриминированы по `type` (`item/agentMessage/delta`, `item/tool/started`, `item/tool/completed`, `item/reasoning/textDelta`, `item/commandExecution/started`, `item/plan/delta` и др.).
  - Детерминированное разделение на scope (`threadScope` vs `turnScope`).
- **Схема входных данных (Prompt Input)**: `shared-types.ts` (`promptInputSchema`).
  - Поддерживаются типы ввода:
    - `text`: текст с поддержкой структурированных упоминаний (`mentions`).
    - `image`: URL-изображение.
    - `localImage`: файл изображения с привязкой к `hostId` и `sourceProjectId`.
    - `localFile`: локальный файл (размер, MIME-тип, путь, хост).
  - Вложенные упоминания (`mentions`): пути (`path`), команды (`command`), секции (`section`), проекты (`project`), потоки (`thread`), плагины (`plugin`), вложения (`attachment`).

### 2.2. `@bb/provider-bridge-protocol` (`packages/provider-bridge-protocol`)
Протокольный слой, определяющий взаимодействие между Agent Runtime и изолированными провайдерами (Claude Code, Pi, ACP-агенты).
- **Bridge Kit (`src/bridge-kit`)**:
  - `runtime-json-rpc.ts`: низкоуровневая реализация JSON-RPC 2.0 поверх потоков Node.js (`Readable` / `Writable` stdio). Поддерживает генерацию request ID, таймауты запросов (`JsonRpcRequestTimeoutError`), корректную сериализацию и парсинг строк (`parseJsonRpcLine`).
  - `bounded-line-reader.ts`: специализированный строковый ридер с ограничением максимального размера строки (`maxLineBytes`), защищающий буферы памяти от переполнения гигантскими JSON-пейлоадами.
  - `pending-tool-call-tracker.ts`: отслеживание асинхронных вызовов внешних инструментов, ожидающих разрешения (resolve/reject) или отмены (cancellation).
  - `provider-visibility.ts`: фильтрация видимости событий между провайдером и клиентом.
- **Delta Assembler (`src/assembler/delta-assembler.ts`)**:
  - Центральный механизм сборки потоковых дельт (`ThreadDelta`) в дискретные доменные события `ThreadEvent`.
  - Поддерживает дросселирование прогресса (`progressThrottleMs`), накопление текстовых чанков (`TEXT_DELTA_FLUSH_MS = 50ms`), дедупликацию и вычисление разницы текста (`diffCumulativeText`).
  - Связывание идентификаторов: двунаправленные маппинги между внутренними ID платформы (`bbItemId`, `bbTurnId`) и внешними ID провайдера (`providerItemId`, `providerTurnId`).

### 2.3. `@bb/agent-runtime` (`packages/agent-runtime`)
Изолированный рантайм выполнения агента.
- **Жизненный цикл процесса (`runtime-provider-process.ts`)**:
  - Запуск дочернего процесса провайдера через `child_process.spawn`.
  - Изоляция переменных окружения (`thread-shell-environment.ts`).
  - Контроль аварийного завершения: захват exit code, сигналов (`SIGTERM`, `SIGKILL`), захват `stderr` с ограничением хвоста через `sliceUtf16Tail`.
  - Перезапуск и сборщик простаивающих сессий (`reapIdleProviderSessions`).
- **Стейт-машины хода (`runtime-turn-state.ts`, `runtime-background-work-state.ts`, `runtime-thread-goal-state.ts`)**:
  - Управление текущим активным turn-ом.
  - Поддержка отмены хода (`runtime-tool-cancellation.ts`).
  - Механизм Watchdog для обнаружения зависания на старте хода (`runtime.turn-start-watchdog.ts`).
  - Поддержка фоновых задач (`RuntimeBackgroundWorkState`) с отслеживанием открытых процессов и автоматическим закрытием.

### 2.4. `@bb/host-daemon` (`apps/host-daemon`)
Демон, работающий непосредственно на целевой машине разработки (локальной или удалённой).
- **Связь с сервером**: клиент WebSocket с автоматическим переподключением, heartbeat-мониторингом и взаимной аутентификацией сессий через токены.
- **Serial Lane Execution (`serial-lane.ts`)**:
  - Очередь сериализации критических мутаций для предотвращения race conditions при конкурентных запросах к одной среде.
- **Изоляция и инжекция скиллов (`injected-skills.ts`, `skill-trees.ts`)**:
  - Подготовка промежуточных каталогов (staging dirs) для скриптов/скиллов.
  - Вычисление SHA256-хэша каталога скиллов для версионирования и обнаружения конфликтов параллельного обновления (`SkillCatalogConflictError`).
- **Файловая система и процессы**:
  - Потоковое чтение файлов чанками (`hostReadFileChunkCommandSchema`).
  - Безопасное исполнение терминальных команд и жизненный цикл Git worktrees.

### 2.5. `@bb/server` (`apps/server`)
Центральный сервер координации, написанный на фреймворке **Hono**.
- **Маршрутизация**: Hono HTTP API + WebSocket Hub.
- **Хранение данных**: Слой репозиториев поверх SQLite/PostgreSQL (`@bb/db`).
- **Управление очередью сообщений (`services/threads/queued-message-dispatch.ts`, `queued-messages.ts`)**:
  - Оркестрация отправки сообщений с учетом статуса агента, блокировок, необходимости подтверждения прав пользователем.

---

## 3. Глубокие настройки и параметры конфигурации

Вся конфигурация в коде является строго типизированной. Ниже приведены конкретные структуры настроек, выявленные при анализе.

### 3.1. Уровни рассуждения (Reasoning Levels)
Файл: `packages/domain/src/shared-types.ts`
```typescript
export const reasoningLevelValues = [
  "none",
  "low",
  "medium",
  "high",
  "xhigh",
  "ultracode",
  "max",
  "ultra",
] as const;
```
- В кодовой базе реализована функция ранжирования `standardReasoningLevelRank(level)` и компаратор `compareReasoningLevels(a, b)`:
  - Ранги от 0 (`none`) до 7 (`ultra`).
  - Позволяет сравнивать возможности моделей и производить fallback при смене провайдера.

### 3.2. Политика разрешений (Permission Modes & Runtime Permission Policy)
Файл: `packages/domain/src/shared-types.ts`
```typescript
export const permissionModeValues = ["accept-edits", "auto", "full"] as const;
```
Дискриминированный союз `runtimePermissionPolicySchema`:
1. `accept-edits`:
   - `permissionScope`: `"workspace"`
   - `approvalReviewer`: `"user"` (каждая критическая операция или запуск команды требует явного подтверждения пользователя)
   - `permissionEscalation`: `"ask"` | `"deny"`
2. `auto`:
   - `permissionScope`: `"workspace"`
   - `approvalReviewer`: `"automatic"` (операции в рамках воркспейса одобряются автоматически)
   - `permissionEscalation`: `"ask"` | `"deny"`
3. `full`:
   - `permissionScope`: `"full"` (полный доступ к хосту без запроса подтверждений)
   - `approvalReviewer`: `null`
   - `permissionEscalation`: `null`

Функция `clampPermissionModeToCeiling` гарантирует, что сессия не сможет запросить права выше потолка, установленного проектом или хостом.

### 3.3. Параметры исполнения потока (Thread Execution Options)
Структура `RuntimeThreadExecutionOptions`:
- `model`: строковый идентификатор модели (например, `claude-3-7-sonnet`, `o3-mini`, `gpt-4o`).
- `serviceTier`: строка, по умолчанию `"default"` (поддержка Fast/Priority tiers).
- `reasoningLevel`: один из уровней рассуждения.
- `promptMode`: режим работы промпта (`"plan"` — перевод агента в режим формирования плана без выполнения деструктивных действий).
- `sessionOptions`: ключ-значение параметров сессии (`Record<string, string | boolean>`).
- `providerOptions`: произвольный JSON-объект с вендор-специфичными параметрами.
- `permissionMode`, `permissionScope`, `approvalReviewer`, `permissionEscalation`.

### 3.4. Опции сессий (Thread Session Options)
Файл: `packages/domain/src/thread-session-option.ts`
Провайдер может объявлять глубокие настройки двух типов:
1. `select`:
   - `id`: уникальный ключ опции.
   - `label`, `description`, `category`.
   - `value`: выбранное строковое значение.
   - `values`: массив допустимых значений (`{ id, label, description, group }`, макс. 1000).
2. `boolean`:
   - `value`: логическое значение (true/false).
   - `fixed`: флаг блокировки изменения пользователем.

### 3.5. Тонкие тайминги рантайма и вотчдоги
- `TEXT_DELTA_FLUSH_MS = 50ms`: период сброса накопленного потокового текста.
- `CHILD_THREAD_TURN_NOTIFICATION_BATCH_DELAY_MS = 2000ms`: окно объединения (coalescing) уведомлений от дочерних потоков.
- `HEARTBEAT_INTERVAL_MS` и `LEASE_TIMEOUT_MS`: контроль доступности WebSocket-соединения демона.
- `PROVIDER_MAINTENANCE_IDLE_TIMEOUT_MS = 60000ms`: тайм-аут остановки неактивного служебного воркспейса.
- `turnStartWatchdog`: интервал и порог обнаружения подвисшего старта хода агента.

---

## 4. Механизмы взаимодействия и технические преимущества реализации

### 4.1. Преимущества технической реализации
1. **Типобезопасность на границах процессов (End-to-End Type Safety)**:
   - Использование Zod на всех сетевых стыках (HTTP, WebSocket, JSON-RPC stdio) исключает несовместимость версий и runtime-падения из-за некорректных форматов данных.
2. **Изоляция отказов процессов (Process Crash Resilience)**:
   - Провайдеры работают в изолированных подпроцессах Node.js. Сбой или зависание LLM CLI-утилиты перехватывается `RuntimeProviderProcessManager`, нормализуется в структурированное событие `provider.error` / `onProcessExit` и не крашит host-daemon или сервер.
3. **Эффективный стриминг дельт (Delta Streaming Engine)**:
   - Вместо постоянной перезаписи полного состояния потока используется дифференциальный протокол `ThreadDelta` (v3). Это минимизирует трафик по сети и нагрузку на БД.
4. **Защита от DOS и переполнения памяти (Memory Protection)**:
   - Внедрение `BoundedLineReader` защищает JSON-RPC парсер от атак с отправкой неограниченно длинных строк.
5. **Предотвращение состояний гонки (Serial Lanes)**:
   - `serial-lane.ts` сериализует взаимоисключающие операции над окружениями хоста, исключая повреждение локальных репозиториев Git и файлов конфигурации.
6. **Гранулярный контроль окружения (Contributed Environment Isolation)**:
   - Разделение переменных среды на базовые, хостовые, плагинные и проектные (`AgentRuntimeContributedEnvEntry`) с отслеживанием источника и причины добавления.
