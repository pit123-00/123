# Специальный технический отчёт: Очередь сообщений, Delta Streaming Engine и защита памяти (Memory Protection)

## 1. Введение

Данный отчёт содержит исчерпывающий технический разбор трёх ключевых подсистем платформы **bb** (`get-bb/bb`), выполненный непосредственно по исходному коду:
1. **Управление очередью сообщений и оркестрация диспетчеризации** (`apps/server/src/services/threads/queued-messages.ts`, `queued-message-dispatch.ts`, `dispatch-attempt.ts`, `packages/domain/src/queued-message.ts`).
2. **Эффективный стриминг дельт (Delta Streaming Engine)** (`packages/provider-bridge-protocol/src/thread-delta.ts`, `packages/provider-bridge-protocol/src/assembler/delta-assembler.ts`).
3. **Защита от DOS и переполнения памяти (Memory Protection & Bounded Line Reader)** (`packages/provider-bridge-protocol/src/bridge-kit/bounded-line-reader.ts`).

---

## 2. Управление очередью сообщений (Queued Message Dispatch System)

### 2.1. Концепция: Очередь как единый шлюз выполнения
В архитектуре `bb` отправка любого сообщения в поток моделируется не как моментальное действие, а как **попытка диспетчеризации** (`attemptDispatch`). Если поток свободен, хост онлайн, а у плагинов и политик безопасности нет претензий, сообщение выполняется немедленно без создания персистентной записи в очереди.

Если же обнаружено хотя бы одно препятствие, сообщение материализуется в виде строки очереди (`queued_thread_messages`) с указанием конкретной причины задержки (`waitingOn`).

### 2.2. Архитектура и состояния ожидания (`QueuedMessageWaitingOnKind`)
В файле `packages/domain/src/queued-message.ts` определён строгий набор причин ожидания (`queuedMessageWaitingOnKindValues`):

```typescript
export const queuedMessageWaitingOnKindValues = [
  "time",            // Сообщение запланировано на будущее (sendAt)
  "thread-busy",     // Поток выполняет turn; сообщение ждёт освобождения (idle)
  "stopping",        // Поток в процессе ручной остановки пользователем
  "provisioning",    // Рабочее пространство потока настраивается
  "host-offline",    // Целевая машина (хост) отключена или засыпает
  "interaction",     // Есть неразрешённый запрос подтверждения (approval/question)
  "plugin",          // Dispatch gate плагина вернул wait(reason)
  "superseded",      // Запрос устарел из-за появления более нового turn-а
] as const;
```

#### Тонкие различия состояний:
- `thread-busy` vs `stopping`: если пользователь нажал «Stop», все ожидающие сообщения ставятся на паузу. Но если пользователь отправил новое сообщение **после** нажатия «Stop» (или нажал «Send now»), такое сообщение помечается как ожидающее `stopping` и отправится сразу, как только процесс агента завершит прерывание.
- `provisioning` vs `host-offline`: при `provisioning` воркспейс инициализируется скриптами и завершится сам. При `host-offline` хост недоступен физически (выключен ноутбук, обрыв туннеля); здесь в очередь вшивается `host.displayName`, чтобы клиент отображал понятный статус ожидания машины без дополнительных запросов к БД.

### 2.3. Механизм транзакционного захвата (`Message Claim Lease`)
Файлы: `queued-messages.ts`, `queued-message-dispatch.ts`, `@bb/db`.

1. **Захват группы сообщений (`claimNextQueuedThreadMessageGroup`)**:
   - Диспетчер сообщений выполняет атомарный захват строк в БД с установкой временной метки аренды (`claimLease`).
   - Исключаются race conditions между несколькими экземплярами сервера или параллельными асинхронными обработчиками событий.
2. **Очистка зависших аренд (`releaseStaleQueuedMessageClaims`)**:
   - Если сервер аварийно завершился во время отправки, таймер аренды истекает, и фоновый сборщик `releaseStaleQueuedMessageDispatchClaims` возвращает сообщение обратно в статус доступных для отправки.
3. **Блокировка редактирования пользователем (`holdQueuedThreadMessageForEdit`)**:
   - Пользователь в UI может нажать «Редактировать» на сообщении в очереди. На строку накладывается блокировка редактирования `holdQueuedThreadMessageForEdit`, исключающая её отправку диспетчером во время правок текста пользователем.

### 2.4. Реактивная модель пробуждения очереди (`QueuedMessageDispatchWake`)
Вместо постоянного поллинга (busy-polling) базы данных модуль `queued-message-dispatch.ts` использует событийно-ориентированную реактивную модель:

```typescript
export type QueuedMessageDispatchWake =
  | { kind: "thread-ready"; threadId: string }
  | { kind: "turn-started"; threadId: string }
  | { kind: "workspace-ready"; threadId: string }
  | { kind: "provisioning-ended"; threadId: string }
  | { kind: "interaction-settled"; threadId: string }
  | { kind: "edit-released"; queuedMessageId: string; threadId: string }
  | { kind: "host-connected"; hostId: string }
  | { kind: "app-update-released" }
  | { kind: "time-reached"; now: number }
  | { kind: "plugin-recheck" }
  | { kind: "plugin-unregistered"; pluginId: string }
  | { kind: "idle-recovery"; now: number }
  | { kind: "failed-retry"; now: number }
  | { kind: "orphaned-plugin-recovery"; plugins: QueueWaitPluginDirectory };
```

Когда происходит любое из этих событий (например, пользователь подтвердил вызов опасного bash-скрипта через `interaction-settled`, или хост восстановил соединение через `host-connected`), диспетчер мгновенно очищает соответствующий wait (`clearQueuedMessageWait`) и вызывает `sendNextQueuedMessageIfPresent`.

### 2.5. Источники попытки диспетчеризации (`DispatchAttemptSource`)
В `dispatch-attempt.ts` реализован полиморфизм источников:
1. `inline`: немедленная отправка в момент вызова API пользователем.
2. `drain`: пакетный прогон сообщений из очереди, уже захваченных в транзакции. Поддерживает флаг `sendNow: boolean` — принудительный обход пользовательских задержек и плагинных gate-ов (но никогда не обходит инварианты ядра, такие как ожидание завершения интеракций).
3. `operation`: системные технические запуски (например, автоматическая компактификация контекста через `/compact` или восстановление истории после форка).

---

## 3. Эффективный стриминг дельт (Delta Streaming Engine)

### 3.1. Проблема традиционных подходов и архитектурное решение
Большинство агентных систем при стриминге ответа либо пересылают накопительный текст целиком (что приводит к квадратичному росту объёма трафика $O(N^2)$), либо заставляют сервер при каждом чанке делать `UPDATE` в базе данных.

В **bb** применён **Delta Streaming Engine** на основе спецификации **ThreadDelta v3** (`packages/provider-bridge-protocol/src/thread-delta.ts`) и ассемблера дельт (`delta-assembler.ts`).

### 3.2. Спецификация дельт `ThreadDelta`
Протокол оперирует атомарными операциями над потоком:
- `turn.open` / `turn.boundary`: объявление начала и конца хода агента с фиксацией контрольной точки провайдера `providerCheckpointId`.
- `item.open`: открытие сущности (инструмент, команда, сообщение агента, рассуждение, делегирование субагенту, поиск).
- `item.textDelta`: передача чистого дифференциала текста (добавочной строки) для конкретного канала (`agentMessage`, `reasoningSummary`, `reasoningText`, `plan`).
- `item.outputDelta`: потоковый вывод запущенной консольной команды (stdout/stderr).
- `item.progress`: обновление снапшота прогресса фоновой задачи или делегирования.
- `item.close`: фиксация статуса завершения (`completed`, `failed`) с кодом возврата, длительностью выполнения в миллисекундах и итоговым результатом.

### 3.3. Внутреннее устройство `DeltaAssembler` (`delta-assembler.ts`)

#### 1. Вычисление текстового дифференциала (`diffCumulativeText`):
```typescript
function diffCumulativeText(args: DiffCumulativeTextArgs): DiffCumulativeTextResult | null {
  const previousText = args.previousText ?? "";
  if (args.nextText.length === 0 || args.nextText === previousText) return null;
  if (previousText.length === 0) {
    return { delta: args.nextText, nextText: args.nextText, reset: false };
  }
  if (args.nextText.startsWith(previousText)) {
    const delta = args.nextText.slice(previousText.length);
    return delta.length > 0 ? { delta, nextText: args.nextText, reset: false } : null;
  }
  // Провайдер перезаписал буфер (reset)
  return { delta: args.nextText, nextText: args.nextText, reset: true };
}
```
Ассемблер прозрачно поддерживает как инкрементальных провайдеров (стримящих чанки), так и провайдеров, отдающих накопительный буфер (cumulation): вычисляется дельта `nextText.slice(previousText.length)` без дублирования трафика.

#### 2. Буферизация и дросселирование (Throttling & Flush Timer):
- Потоковый сброс текста происходит не на каждый байт, а пачками с таймингом `TEXT_DELTA_FLUSH_MS = 50ms`. Это предотвращает лавину микро-событий в сторону фронтенда и базы данных.
- События прогресса фоновых задач сглаживаются параметром `progressThrottleMs`.

#### 3. Связывание субагентов и инструментов (`parentRef` -> `parentToolCallId`):
- Когда провайдер запускает субагента (например, Cursor `task` или Claude Code `Agent`), в дельте передаётся `key.parentRef`.
- `DeltaAssembler` поддерживает двунаправленную таблицу идентификаторов:
  - `bbItemIdByProviderItemId: Map<string, string>`
  - `providerItemIdByBbItemId: Map<string, string>`
- Функция `mapParentRef(state, parentRef)` связывает дочерние операции (команды, вложенные рассуждения, выводы) с родительским инструментом. Все события размечаются полем `parentToolCallId`, позволяя строить вложенное дерево UI.

#### 4. Кольцевые буферы с вытеснением старых записей:
Для предотвращения утечек памяти при длительных сессиях мапы идентификаторов ограничены константами:
- `MAX_THREAD_STATES = 256`
- `MAX_ID_MAP_ENTRIES = 1024`
- `MAX_SETTLED_ITEM_KEYS = 512`
При превышении размера вызывается `trimOldestEntries(map, max)`, удаляющая старейшие итераторы Map по принципу LRU.

---

## 4. Защита от DOS и переполнения памяти (BoundedLineReader)

### 4.1. Архитектурная угроза при работе с JSON-RPC по stdio
Агенты общаются с bridge-процессами через стандартные потоки ввода-вывода (`stdin`/`stdout`) в формате строк JSON-RPC (разделённых `\n`).

Стандартные реализации (например, `readline` в Node.js) буферизуют входящие байты в строке до тех пор, пока не встретят символ перевода строки. Если сторонний CLI-агент, плагин или скомпрометированный процесс начнет непрерывно гнать миллионы байт без символа `\n` (или выведет бинарный блоб / дамп файла), стандартный буфер Node.js вызовет падение процесса с ошибкой `JavaScript heap out of memory` (Denial of Service).

### 4.2. Техническая реализация `BoundedLineReader`
Файл: `packages/provider-bridge-protocol/src/bridge-kit/bounded-line-reader.ts`.

```typescript
export const MAX_JSON_RPC_LINE_BYTES = 64 * 1024 * 1024; // Лимит 64 МБ

export interface BoundedLineReaderArgs {
  input: NodeJS.ReadableStream;
  onLine: (line: string) => void;
  onOverflow: (bytes: number) => void;
  onClose?: () => void;
  maxLineBytes?: number;
}
```

### 4.3. Алгоритмические особенности работы
1. **Декодирование UTF-8 чанков через `StringDecoder`**:
   - Используется `new StringDecoder("utf8")`.
   - В отличие от `Buffer.toString()`, `StringDecoder` корректно обрабатывает многобайтовые UTF-8 символы, которые могут быть разорваны на стыке двух сетевых чанков.
2. **Конечный автомат с состоянием отсечки (`discarding`)**:
   - Переменная `discarding: boolean` (по умолчанию `false`).
   - Переменная `discardedBytes: number` (счётчик пропущенного мусора).
   - `pending: string` — накопитель текущей строки.
3. **Поведение при превышении лимита**:
   - Когда размер накопленного буфера `Buffer.byteLength(pending)` превышает `maxLineBytes`, ридер переходит в режим `discarding = true`.
   - Буфер `pending` **мгновенно обнуляется** (`pending = ""`), немедленно высвобождая память для Garbage Collector V8.
   - Все последующие чанки до следующего символа перевода строки `\n` не конкатенируются в строку, а просто увеличивают целочисленный счётчик `discardedBytes += Buffer.byteLength(tail)`.
4. **Восстановление потока (Recovery)**:
   - Как только в потоке встречается символ `\n`:
     - Режим сброса отключается: `discarding = false`.
     - Вызывается колбэк `args.onOverflow(discardedBytes)`, регистрирующий инцидент в логах безопасности.
     - Чтение возобновляется со следующего байта после `\n` как чистая новая строка.
5. **Защита от висящего мусора в конце потока (`end`)**:
   - Если поток завершился (`end`), а ридер находится в режиме `discarding = true`, остаток строки никогда не эмитится.
   - Также удаляется завершающий `\r` (совместимость с Windows CRLF).

---

## 5. Заключение

Инженерные решения **bb** в части очередей, стриминга дельт и защиты памяти представляют собой целостный комплекс мер промышленного уровня:
1. Очередь сообщений гарантирует инварианты безопасности (невозможность нарушить изоляцию или пропустить интерактивное подтверждение прав пользователем) при сохранении высокой отзывчивости UI благодаря реактивным триггерам `QueuedMessageDispatchWake`.
2. Delta Streaming Engine решает проблему масштабируемости передачи данных, оптимизируя сетевую нагрузку и поддерживая глубокую вложенность субагентов.
3. BoundedLineReader устраняет критическую уязвимость переполнения кучи Node.js при обработке внешних LLM CLI-потоков.
