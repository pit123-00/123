# Технический отчёт: Анализ кодовой базы WorkflowBuilder

## 1. Введение и область анализа
Данный отчёт содержит детальный технический анализ кодовой базы **WorkflowBuilder** (репозиторий `synergycodes/workflowbuilder`), выполненный непосредственно по исходному коду (без использования высокоуровневых README).

Репозиторий представляет собой модульный TypeScript-монорепозиторий (управляемый через `pnpm workspaces` / `turborepo`), включающий:
* Пакеты ядра исполнения и абстракций: `@workflow-builder/execution-core`, `@workflow-builder/types`, `@workflow-builder/ai-config`.
* Пакеты интеграции с оркестратором: `@workflowbuilder/temporal`, `@workflow-builder/temporal-connection`.
* Frontend SDK и компоненты: `@workflowbuilder/sdk`, `@workflowbuilder/ui`, `@workflowbuilder/tokens`.
* Приложения:
  * `apps/ai-studio` — полнофункциональная среда визуального конструирования и тестирования AI-воркфлоу.
  * `apps/execution-worker` — воркер выполнения операций (Temporal Worker + Vercel AI SDK + Tavily Search).
  * `apps/backend` — API-шлюз и слой персистентности на Hono + Drizzle ORM (PostgreSQL).
  * `packages/mcp-server` — сервер Model Context Protocol для экспорта документации и схем в LLM.

---

## 2. Каталог узлов (Node Types), схемы и детальные параметры

Каждый узел в системе описывается тремя ключевыми сущностями:
1. **JSON Schema (`schema.ts`)** — определяет контракт свойств узла для валидации и UI-генерации.
2. **UISchema (`uischema.ts`)** — спецификация рендеринга формы настройки в боковой панели редактора (`@workflowbuilder/sdk`).
3. **Исполнитель (`NodeExecutor` в `execution-worker`)** — логика выполнения шага при обработке графа.

### Общие свойства всех узлов (`sharedProperties`)
Все узлы наследуют базовые метаданные из `@workflowbuilder/sdk`:
* `label` (`string`): Название узла в графе.
* `description` (`string`): Описание назначения узла.
* `errorPolicy` (`'fail' | 'continue' | 'errorRoute'`, опционально, по умолчанию `'fail'`):
  * `'fail'` — немедленно прерывает выполнение всего графа при сбое узла.
  * `'continue'` — поглощает ошибку узла, узел считается завершенным с ошибкой, но зависимые ветки продолжают работу.
  * `'errorRoute'` — активирует специальный исходящий порт ошибки `RESERVED_ERROR_HANDLE = 'errorRoute'`.

---

### 2.1. Узел Trigger (`ai-studio/trigger`)
**Назначение:** Точка входа в выполнение воркфлоу, принимающая внешние входные данные.

* **Свойства конфигурации (`schema.ts`):**
  * `inputPrompt` (`string`): Пользовательский текст/запрос, инициирующий пайплайн.
* **UI-представление (`uischema.ts`):**
  * `Title` (текстовое поле `properties.label`).
  * `Input Prompt` (многострочное поле `properties.inputPrompt`, minRows: 4, maxRows: 10).
* **Схема выходов (`outputSchema`):**
  * `{ input: { type: 'string', label: 'Input' } }`.
* **Техническая реализация исполнителя (`executeTrigger`):**
  ```typescript
  export function executeTrigger(node: TriggerNode, context: ExecutionContext) {
    const input =
      typeof context.triggerPayload?.['input'] === 'string'
        ? context.triggerPayload['input']
        : node.config.inputPrompt;
    return { output: { input } };
  }
  ```
  *Приоритет данных:* если в `context.triggerPayload` передан параметр `input`, он переопределяет статически заданный `inputPrompt`.

---

### 2.2. Узел AI Agent (`ai-studio/ai-agent`)
**Назначение:** Запуск вызова языковой модели (LLM) с поддержкой шаблонизации промптов, веб-поиска (инструментов) и структурированного JSON-вывода.

* **Свойства конфигурации (`schema.ts`):**
  * `systemPrompt` (`string`): Системная инструкция агента с поддержкой интерполяции `{{...}}`.
  * `webSearch` (`boolean`): Флаг разрешения использования инструмента внешнего поиска через Tavily API.
  * `outputSchema` (`object`, JSON Schema): Схема структурированного ответа (Structured Output). Если `null` или отсутствует — агент возвращает неструктурированный текст в поле `response`.
* **UI-представление (`uischema.ts`):**
  * `Title` (текстовое поле).
  * `System Prompt` (многострочный редактор с поддержкой подсказок переменных `{{...}}`).
  * `Response format` (`ResponseSelect`): выбор между `Text (freeform)` и кастомными схемами Structured Outputs.
  * `Web search` (переключатель `Switch`).
* **Техническая реализация исполнителя (`executeAiAgent`):**
  * **Формирование контекста:** Все предыдущие выходы графа (`context.nodeOutputs`) объединяются в пользовательский промпт:
    `[<nodeId>]: <output_or_response>`.
  * **Промпт системы:** Выполняется разрешение шаблонов через `resolveTemplate(node.config.systemPrompt, context)`.
  * **Инструменты (Tools):**
    * Если `webSearch === true` и задан `TAVILY_API_KEY`, создается инструмент `webSearch` (`apps/execution-worker/src/tools/web-search.ts`).
    * Лимит шагов цикла: `MAX_TOOL_STEPS = 4`. Используется `stopWhen: stepCountIs(4)`.
    * **Защита от бесконечного цикла инструментов:** на предпоследнем шаге принудительно задается `{ toolChoice: 'none' }`, что гарантирует получение финального ответа модели без очередного вызова инструмента.
  * **Structured Outputs:**
    * Валидируется корень схемы: должен быть строго `type: 'object'`.
    * Используется Vercel AI SDK `Output.object({ schema: jsonSchema(outputSchema) })`.
    * Проверяется `finishReason`: если модель остановилась не с кодом `'stop'` (например, `length` или `content-filter`), выбрасывается `TransientNodeExecutionError('structured_output_incomplete')`.
  * **Классификация ошибок:** Ошибки LLM транслируются через `classifyProviderError` в постоянные (`PermanentNodeExecutionError`) или временные (`TransientNodeExecutionError`), что предотвращает бесконечные ретраи невалидных конфигураций.

---

### 2.3. Узел Decision (`ai-studio/decision`)
**Назначение:** Условная маршрутизация графа по веткам на основе выражений и предикатов.

* **Свойства конфигурации (`schema.ts`):**
  * `decisionBranches` (`Array`):
    * `id` (`string`): Идентификатор ветки.
    * `sourceHandle` (`string`): Имя исходящего порта для маршрутизации.
    * `label` (`string`): Пользовательское имя ветки.
    * `conditions` (`Array`):
      * `x` (`string`): Левый операнд (поддерживает интерполяцию `{{nodes.agent1.response}}`).
      * `comparisonOperator` (`string`): Оператор сравнения.
      * `y` (`string`): Правый операнд (строка или шаблон).
      * `logicalOperator` (`'AND' | 'OR'`): Логическая связка с последующим условием.
* **Поддерживаемые операторы сравнения (`evaluateCondition`):**
  * `isEqual` (`left === right`)
  * `isNotEqual` (`left !== right`)
  * `isContaining` (`left.toLowerCase().includes(right.toLowerCase())` — регистронезависимо)
  * `isNotContaining` (`!left.toLowerCase().includes(right.toLowerCase())`)
  * `isBefore` (`new Date(left).getTime() <= new Date(right).getTime()`)
  * `isAfter` (`new Date(left).getTime() >= new Date(right).getTime()`)
  * `isGreaterThan` (`Number(left) > Number(right)`)
  * `isLessThan` (`Number(left) < Number(right)`)
  * `isGreaterThanOrEqual` (`Number(left) >= Number(right)`)
  * `isLessThanOrEqual` (`Number(left) <= Number(right)`)
* **Техническая реализация исполнителя (`executeDecision`):**
  * Выполняет последовательный перебор веток в порядке их задания (`for...of`).
  * Для условий внутри ветки вычисляет выражение слева направо с учётом `logicalOperator`.
  * Возвращает `{ output: { matchedBranch: branch.sourceHandle }, nextPort: branch.sourceHandle }`.
  * **Fail-Fast политика при отсутствии совпадений:** Если ни одна ветка не совпала, система **не** переходит автоматически к первой ветке, а выбрасывает `PermanentNodeExecutionError('no_branch_matched')`. Это защищает от скрытых ошибок логики воркфлоу.

---

### 2.4. Узлы Human Decision и Review (`ai-studio/human-decision`, `ai-studio/review`)
**Назначение:** Приостановка выполнения воркфлоу для получения решения человека (Human-in-the-loop / HITL).

* **Свойства конфигурации:**
  * `decisionRequest` (`DecisionRequestContract`):
    * `proposal`: данные, которые показываются оператору (текст, ссылки, сводки).
    * `fields`: поля ввода формы для человека (текст, чекбоксы, селекты, замечания).
    * `actions`: доступные действия. Допустимые эффекты:
      * `resume`: продолжить пайплайн по указанному порту (`port`). Может требовать причину (`reasonRequired`).
      * `reject`: отклонить выполнение и направить по порту отказа с фиксацией вердикта.
      * `rerun-source`: запросить повторный прогон шага-источника.
* **Техническая реализация:**
  * Исполнитель в воркере:
    ```typescript
    export function executeHumanDecision(node: Pick<HumanDecisionNode, 'id' | 'decisionRequest'>): NodeExecutionResult {
      if (node.decisionRequest === undefined) {
        throw new PermanentNodeExecutionError('decision_request_missing', ...);
      }
      return { waiting: true };
    }
    ```
  * Узел сигнализирует ядру `{ waiting: true }`. Ядро переводит узел в состояние ожидания (`node_waiting`), статус воркфлоу — в `'waiting'`.
  * Temporal Workflow паркует выполнение через `await condition(() => waits.get(nodeId)?.status === 'resolved')`.
  * Решение принимается через REST API бэкенда (`POST /api/executions/:id/decision`), который валидирует отправленные поля и вызывает Temporal Update `resolveNodeUpdate`.

---

### 2.5. Узел Visualize (`ai-studio/visualize`)
**Назначение:** Терминальный или промежуточный узел отображения результатов.

* **Конфигурация (`mode`):**
  * Режимы: `'auto'`, `'markdown'`, `'text'`, `'json'`, `'table'`, `'stat-cards'`, `'chart'`, `'diagram'`.
* **Техническая реализация:**
  * Исполнитель в воркере возвращает пустой вывод `{ output: {} }`.
  * Логика отображения полностью выполняется на стороне UI (`apps/ai-studio/src/utils/adapt-visualization.ts`), который считывает вывод предшествующего узла и динамически адаптирует представление под выбранный режим.

---

## 3. Механизм шаблонизации и интерполяции переменных

Шаблонизация реализована в модуле `packages/execution-core/src/templates/resolve-template.ts`.

### 3.1. Грамматика выражений
Шаблонные выражения имеют вид:
`{{ <namespace>.<path> [?] [| default:'<value>'] }}`

1. **Пространства имен (`namespace`):**
   * `nodes`: выходы завершенных узлов (`context.nodeOutputs[nodeId]`).
   * `trigger`: полезная нагрузка триггера запуска (`context.triggerPayload`).
   * `variables`: переменные окружения графа (`context.variables`).
   * `global`: глобальные переменные тенанта/системы (`context.global`).
2. **Путь доступа (`path`):**
   * Поддерживает точечную нотацию и дефисы в slug-идентификаторах узлов (например, `nodes.ai-agent-1.response.summary`).
3. **Безопасная навигация (`?`):**
   * Пример: `{{nodes.classifier.category?}}`. Если значение не найдено, возвращается пустая строка `''` вместо ошибки.
4. **Значение по умолчанию (`| default:'...'`):**
   * Пример: `{{trigger.priority | default:'normal'}}`. Если значение не найдено, подставляется указанный литерал.
5. **Строгий режим по умолчанию:**
   * Простое выражение `{{nodes.agent.output}}` при отсутствии ключа или узла выбрасывает `PermanentNodeExecutionError('template_unresolved')`. Это предотвращает передачу необработанных литералов в LLM.
6. **Двухфазный парсер:**
   * Внешний Regex (`OUTER_TEMPLATE_REGEX`) находит потенциальные блоки с точкой.
   * Внутренний Regex (`PARSE_REGEX`) валидирует точную структуру. Ошибки синтаксиса приводят к `template_malformed`, исключая «тихие» пропуски повреждённых шаблонов.

---

## 4. Преимущества технической реализации

1. **Строгая типизация без привязки к конкретному домену:**
   * `packages/execution-core` параметризован универсальным дженериком `TNode extends BaseNode`. Ядро графа не знает про `AiAgentNode` или `DecisionNode` — оно оперирует лишь структурой `{ id, type, config }`. Все доменные узлы AI Studio изолированы в `apps/execution-worker`.
2. **Разделение песочниц выполнения (Sandbox Boundary):**
   * Файл `packages/execution-core/src/workflow.ts` экспортирует код, гарантированно детерминированный и безопасный для запуска в изолированной V8 песочнице Temporal Workflow (без Node I/O, Web API, `Math.random` и недетерминированного времени).
   * Все недетерминированные и ресурсоемкие операции (сетевые вызовы LLM, поиск, доступ к БД) изолированы в Temporal Activities.
3. **Классификация ошибок выполнения:**
   * Разделение на `PermanentNodeExecutionError` (неверная конфигурация, невалидная схема) и `TransientNodeExecutionError` (сетевые сбои, таймауты). Постоянные ошибки сразу завершают узел без бессмысленных повторов (retries), экономя бюджет токенов и квоты API.
4. **Контроль зацикливания и лимитов инструментов:**
   * Детерминированное ограничение шагов цикла агента (`MAX_TOOL_STEPS = 4`) с принудительным запретом инструментов на последнем шаге гарантирует завершение вызова даже при склонности модели к рекурсивным поискам.
5. **Комплексная защита от деградации памяти и размера сообщений:**
   * Логирование предупреждений о размере пейлоада (`store-payload-warning.ts`) при превышении допустимых лимитов базы данных и сообщений шины событий.
