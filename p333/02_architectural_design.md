# Архитектурный отчёт: Архитектура WorkflowBuilder

## 1. Введение и архитектурные цели
Система **WorkflowBuilder** спроектирована как распределенная отказоустойчивая платформа для проектирования, оркестрации и исполнения графов рабочих процессов (DAG-пайплайнов) с поддержкой автономных AI-агентов и участия человека (Human-in-the-loop).

Главные архитектурные цели:
1. **Durable Execution (Надёжное исполнение):** Устойчивость к падениям серверов, сбоям внешних API, долгим ожиданиям решений людей без потери состояния.
2. **Изоляция домена от движка выполнения:** Ядро графа (`execution-core`) абстрагировано от конкретного оркестратора и типов узлов.
3. **Реактивность и потоковая передача:** Потоковое обновление UI в реальном времени с минимальной задержкой через Server-Sent Events (SSE).
4. **Безопасность и многопользовательская изоляция:** Поддержка Multi-tenancy, разграничение доступа через порты авторизации, rate-limiting и защита от ботов.

---

## 2. Архитектура монорепозитория и слоёв (Packages & Apps)

Кодовая база структурирована в соответствии с принципами чистой архитектуры и модульного монорепозитория:

```
[ Frontend: React + xyflow / ReactFlow ]
       │
       ▼ (REST API / SSE)
[ apps/backend: Hono API Gateway + Drizzle ORM ]
       │                               ▲
       ▼ (Temporal Client)             │ (Database Activities: Events & Status)
[ Temporal Server (Cluster / Cloud) ]  │
       │                               │
       ▼ (gRPC / Task Queue)           │
[ apps/execution-worker: Temporal Worker ]
  ├── @workflowbuilder/temporal (Plugin & Workflow Definitions)
  ├── @workflow-builder/execution-core (Pure DAG Runner)
  └── Concrete Executors (Vercel AI SDK, Tavily, Decision Logic)
```

### Слои монорепозитория:
* **`packages/types`**: Базовые структуры данных и контракты: модели графа (`BaseNode`, `WorkflowEdgeDefinition`), события выполнения (`ExecutionEvent`), контракты решений (`DecisionRequestContract`).
* **`packages/execution-core`**: Чистое ядро выполнения графа. Не зависит от Node.js I/O, внешних API или Temporal. Реализует топологический планировщик, работу с шаблонами и детерминированный цикл обхода графа.
* **`packages/temporal`**: Плагин и адаптер для интеграции `execution-core` с Temporal TypeScript SDK. Предоставляет workflow-функцию `runWorkflow` и активности сохранения состояния.
* **`packages/temporal-connection`**: Утилиты безопасного и типизированного подключения к кластеру Temporal (mTLS, namespace, address resolution).
* **`packages/ai-config`**: Централизованная конфигурация OpenAI-совместимых провайдеров.
* **`packages/sdk` & `packages/ui`**: React SDK на базе `@xyflow/react` для визуального редактирования графов, компонентов свойств и тем оформления.
* **`apps/backend`**: Высокопроизводительный сервер на Hono, управляющий жизненным циклом запусков, персистентностью в PostgreSQL и стримингом событий.
* **`apps/execution-worker`**: Воркер, подписывающийся на Temporal Task Queue и исполняющий задачи узлов (LLM вызовы, ветвления, веб-поиск).

---

## 3. Архитектура портов и адаптеров (Hexagonal Architecture)

В `packages/execution-core` взаимодействие с внешней средой вынесено в порты (интерфейсы TypeScript):

```
┌────────────────────────────────────────────────────────┐
│             @workflow-builder/execution-core           │
│                                                        │
│                    runGraph()                          │
│                        │                               │
│        ┌───────────────┼───────────────┐               │
│        ▼               ▼               ▼               │
│  [ActivityRunner] [EventEmitter] [VersioningPort]      │
└────────┬───────────────┬───────────────┬───────────────┘
         │               │               │
         ▼               ▼               ▼
┌────────────────────────────────────────────────────────┐
│               @workflowbuilder/temporal                │
│  (proxyActivities) (SequencedEmitter) (patched())      │
└────────────────────────────────────────────────────────┘
```

1. **`ActivityRunnerPort<TNode>`**:
   * Абстрагирует выполнение узла. В песочнице Temporal реализация оборачивает вызов в `proxyActivities`.
   * Метод `awaitResolution(nodeId)`: порт перевода узла в режим ожидания (парковки) до поступления внешнего вердикта.
2. **`EventEmitterPort`**:
   * Абстракция эмиссии событий жизненного цикла: `node_started`, `node_completed`, `node_failed`, `node_skipped`, `node_waiting`, `execution_started`, `execution_completed`, `execution_failed`, `execution_cancelled`.
3. **`VersioningPort`**:
   * Абстракция поддержки версионирования и миграции графов при живом выполнении (обертка вокруг `patched()` из Temporal).
4. **`WorkflowEnginePort`**:
   * Порт на стороне бэкенда (`apps/backend/src/engine`), абстрагирующий вызовы к движку оркестратора (запуск воркфлоу, передача решения через Update).

На стороне бэкенда (`apps/backend`) также применен паттерн Ports and Adapters:
* **`AuthPort`** (`auth-port.ts`): абстрагирует систему аутентификации/авторизации (предоставлен `AllowAllAuthPort` для локальной разработки и расширяемый интерфейс для production-провайдеров).
* **`TenantContextPort`** (`tenant-context-port.ts`): абстрагирует извлечение ID тенанта для обеспечения строгой многопользовательской изоляции данных в СУБД.

---

## 4. Модель исполнения графа: Волновой топологический планировщик

В `packages/execution-core/src/graph-runner.ts` реализован высокопроизводительный детерминированный волновой планировщик:

### 4.1. Внутреннее состояние планировщика (`SchedulerState`)
* `adjacency`: Карта смежности `Map<nodeId, Array<{ node, sourceHandle }>>`.
* `inDegree`: Начальная степень захода (количество входящих связей).
* `pendingPredecessors`: Счетчик незавершенных предшественников. Декрементируется, когда узел-источник завершился (успешно, с ошибкой или был пропущен).
* `liveIncoming`: Счетчик активных («живых») входящих ребер. Инкрементируется только тогда, когда связь реально сработала (не была отсечена условием Decision).
* `livePruneKind`: Фиксирует причину отсечения (`'branch'` или `'error'`).
* `status`: Статус каждого узла в рантайме: `'pending' | 'completed' | 'skipped'`.

### 4.2. Алгоритм волнового исполнения
1. **Поиск стартовых узлов:** Находятся узлы с `inDegree === 0` (обычно `TriggerNode`).
2. **Формирование волны (Wave):** Все готовые узлы исполняются конкурентно (`Promise.allSettled`).
3. **Распространение результатов (`propagate`):**
   * При завершении узла его исходящие связи анализируются функцией `isEdgeLive()`.
   * Для всех целевых узлов уменьшается `pendingPredecessors`.
   * Если `pendingPredecessors === 0`, узел готов к вычислению:
     * Если `liveIncoming > 0` — узел ставится в следующую волну выполнения.
     * Если `liveIncoming === 0` — узел помечается как `'skipped'`, формируется причина (`'branch_not_taken'` или `'error_route_not_taken'`), и через итеративную очередь запускается распространение пропуска по всему нисходящему подграфу.
4. **Обнаружение тупиков (Dead Ends):**
   * Если узел указал целевой порт (`nextPort`), но к нему не подключено ни одного ребра, он фиксируется как тупик (`deadEnds`).
   * Воркфлоу завершается со статусом `'incomplete'`, что позволяет корректно фиксировать незавершенные ветки без падения всего пайплайна.

---

## 5. Интеграция с Temporal (Durable Workflows & Activities)

Архитектура опирается на движок Temporal для обеспечения абсолютной надежности:

```
                  ┌───────────────────────────────┐
                  │   Temporal Workflow Sandbox   │
                  │  runGraph() pure scheduler    │
                  └──────────────┬────────────────┘
                                 │ proxyActivities
                  ┌──────────────▼────────────────┐
                  │    Node.js Activity Worker    │
                  │  executeNode(node, context)   │
                  └──────┬─────────────────┬──────┘
                         │                 │
                ┌────────▼────────┐ ┌──────▼────────┐
                │ Vercel AI SDK   │ │ Tavily Search │
                │ (OpenAI Compat) │ │ REST API      │
                └─────────────────┘ └───────────────┘
```

1. **Песочница V8 Workflow:**
   * Воркфлоу-код `runWorkflow` компилируется в единый детерминированный бандл.
   * Все вызовы ввода-вывода строго запрещены — они транслируются через `proxyActivities`.
2. **Профили активности (`activity-profiles.ts`):**
   * Раздельные политики для разных типов операций:
     * `DEFAULT_DATABASE_ACTIVITY_PROFILE`: частые короткие операции сохранения событий с быстрыми таймаутами и агрессивным ретраем.
     * `NodeActivityProfiles`: настраиваемые таймауты (`startToCloseTimeout`, `scheduleToCloseTimeout`) и retry-политики для тяжелых LLM-вызовов.
3. **Секвенированная эмиссия событий (`SequencedEventEmitter`):**
   * Чтобы порядок событий в распределенной среде не нарушался из-за сетевых задержек, каждое событие снабжается строго монотонным порядковым номером (`sequenceNumber`), гарантируя консистентный лог в БД.

---

## 6. Архитектура Human-in-the-Loop (Temporal Updates)

WorkflowBuilder реализует паттерн ожидания человека через **Temporal Updates** (а не через традиционные Signals):

1. **Парковка воркфлоу:**
   * Узел `human-decision` возвращает `{ waiting: true }`.
   * Workflow-функция регистрирует ожидание в мапе `waits.set(nodeId, { status: 'waiting' })`.
   * Воркфлоу засыпает на `await condition(() => waits.get(nodeId)?.status === 'resolved')`.
2. **Прием решения через API:**
   * Пользователь отправляет вердикт через UI (`POST /api/executions/:id/decision`).
   * Бэкенд находит ожидающий узел и проверяет валидность данных по схеме `decisionRequest`.
   * Бэкенд вызывает Temporal Update `resolveNodeUpdate`:
     ```typescript
     await handle.executeUpdate(resolveNodeUpdate, {
       args: [{ nodeId, resolution }],
     });
     ```
3. **In-Flight валидация (Temporal Update Validator):**
   * Temporal выполняет синхронную валидацию `validateVerdict()` еще до записи события в Event History:
     * Проверяется, известен ли узел в графе.
     * Проверяется, действительно ли узел находится в статусе `'waiting'`.
     * Если узел не ждет решения или уже разрешен — Update отклоняется синхронно, а вызывающий API получает статус 409 Conflict без загрязнения истории воркфлоу.
4. **Пробуждение и продолжение:**
   * Хэндлер обновляет статус в `waits.set(nodeId, { status: 'resolved', resolution })`.
   * `condition()` разблокируется, узел завершается и планировщик передает управление по выбранному человеку порту.

---

## 7. Бэкенд и стриминг данных в реальном времени

Архитектура бэкенда (`apps/backend`) построена на современных легковесных технологиях:
* **Hono:** сверхбыстрый веб-фреймворк с минимальным оверхедом.
* **Drizzle ORM + PostgreSQL:** типобезопасный доступ к данным, транзакции, поддержка JSON-полей для графов и событий.
* **SSE-стриминг с защитой порядка событий (`SerializedDrainer`):**
  * Поток событий выполнения транслируется в браузер через Server-Sent Events (`/api/executions/:id/stream`).
  * `SerializedDrainer` гарантирует сериализованный порядок вычитки событий из БД и защиту от гонок между фоновой записью из Temporal Activity и активным HTTP-стримом.
  * Фронтенд (`apps/ai-studio`) подключает EventSource и реактивно отображает подсветку узлов, бейджи статусов и лог-панель.
