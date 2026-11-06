# Отчёт 1: Ключевые отличия между hermes-agent и deepseek-harness

Данный сравнительный отчёт составлен на основе глубокого анализа исходного кода репозиториев [nousresearch/hermes-agent](https://github.com/nousresearch/hermes-agent) и [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness). В центре внимания — реальная кодовая база, используемые парадигмы, структуры данных, модели выполнения, настройки и интерфейсы управления агентами.

---

## 1. Концептуальное позиционирование и парадигма реализации

### hermes-agent (Nous Research)
* **Языковой стек и среда:** Python (3.11–3.14), строгая модель управления зависимостями через `uv.lock` и точное версионирование `==X.Y.Z` в `pyproject.toml` (запрет диапазонов версий обусловлен защитой от атак на цепочку поставок).
* **Парадигма кода:** Монолитное ядро с композицией через Python Mixins (`run_agent.py` объединяет `ClientLifecycleMixin`, `StreamDeliveryMixin`, `InterruptControlMixin`, `TurnFacadeMixin`, `SessionPersistenceMixin`, `CompressionFacadeMixin` и др.).
* **Философия исполнения:** Автономный самосовершенствующийся агент («self-improving agent»), ориентированный на накопление навыков (skills), локальную базу сессий в SQLite с WAL-режимом, терминальное и TUI взаимодействие, а также интеграцию с локальными и внешними LLM-провайдерами (OpenAI, Anthropic, Ollama, LMStudio, OpenRouter).
* **Организация логики:** Модули прямого вызова функций и процедурный пайплайн обработки сообщений с динамической подгрузкой инструментов и плагинов.

### deepseek-harness (DeepSeek AI)
* **Языковой стек и среда:** TypeScript / Node.js (monorepo под управлением `pnpm` workspaces, сборка через `tsdown`, валидация схем через `@deepseek-ai/schemastery`, строгий TypeScript `strict: true`).
* **Парадигма кода:** Микроядерная архитектура на базе Inversion of Control (IoC) фреймворка **Cordis** (`@deepseek-ai/cordis`). Вся система построена вокруг реестра сервисов (`extends Service`) и так называемых «capability seams» (швов возможностей).
* **Философия исполнения:** Высокоструктурированная, промышленная платформа-рантайм (Harness) для многосессионных, распределённых сценариев, десктопных и веб-интерфейсов, с изолированными воркерами (PTC — Programmatic Tool Calling), поддержкой протокола ACP (Agent Client Protocol) и гибкой сменой подсистем (хранилище, песочница, LLM-транспорт).
* **Организация логики:** Событийно-ориентированная архитектура (Event Sourcing) с неизменяемыми логами сессий (`SessionLogOffset`, `SessionSeq`), проекциями (Projections) и реактивными очередями сообщений (Inbox).

---

## 2. Сравнительная таблица ключевых различий

| Параметр | `nousresearch/hermes-agent` | `deepseek-ai/deepseek-harness` |
| :--- | :--- | :--- |
| **Основной язык** | Python 3.11–3.14 | TypeScript (ESNext, Node.js >= 22) |
| **Структура репозитория** | Крупный модуль с директориями `agent/`, `tools/`, `hermes_cli/`, плоскими вспомогательными файлами `hermes_state_*.py` | Монорепозиторий (`packages/`, `apps/`) из 50+ модульных пакетов |
| **Ядро композиции** | Класс `AIAgent`, собирающий 14+ Mixin-классов в один рантайм | Контекст Cordis (`Context`) со слабосвязанными сервисами (`Service`), регистрируемыми в рантайме |
| **Модель состояния сессии** | Реляционная SQLite база данных (`sessions`, `messages`, `tasks`, WAL-режим) с процедурными запросами | Event-sourced журнал событий (append-only events), проецируемый в память (`SessionSurface`), с адаптерами хранения (JSON, SQLite) |
| **Исполнение инструментов** | Прямой вызов Python-функций (`tools/`), bash-песочницы (Docker, Modal, Daytona, Local) через `terminal_tool.py` | Native Tool Execution + PTC (Programmatic Tool Calling) через изолированные Node-процессы/воркеры с RPC |
| **Управление субагентами** | Инструмент `delegate_task` (`tools/delegate_tool*.py`), создающий дочерний инстанс `AIAgent` с усечёнными наборами тулов | Пакет `@deepseek-ai/dsh-subagent` с реестром провайдеров (`spawn-in-process`, `fork-in-process`, `codex`, `claude-code`, `acp`) |
| **Конфигурация** | `config.yaml` (`hermes_cli/config_defaults.py`), профили, CLI-аргументы | Реактивные профили Cordis (`packages/settings`), схемы `@deepseek-ai/schemastery`, динамические патчи конфигураций |
| **Управление контекстом** | Эвристическое сжатие диалога (`trajectory_compressor.py`, `context_compressor.py`), отсечение старых ходов, локальная память | Мультиуровневое сжатие (`compaction-basic`, `compaction-tool-result-pruner`, `compaction-image-offload`), отслеживание токенов |
| **Пользовательские интерфейсы** | CLI (`hermes_cli`), Rich TUI (`ui-tui`), фоновый шлюз WebSocket/Gateway | CLI (`apps/cli`), полноценный Electron Desktop (`apps/desktop`), Web UI (`apps/web`) |

---

## 3. Различия в модели выполнения (Agent Loop)

### hermes-agent: Цикл на основе Mixin-фасада
В `hermes-agent` цикл выполнения сосредоточен в методе `run_conversation` класса `AIAgent`.
1. **Процесс шага (Turn):**
   - Формирование контекста через `TurnFacadeMixin` и `prompt_builder.py`.
   - Запрос к модели через абстракцию `ClientLifecycleMixin` (поддержка параллельных фоллбэков при ошибках рейтов).
   - Парсинг ответов: разделение блока `<think>` от содержательной части (`think_scrubber.py`).
   - Исполнение тулов: последовательное или батчевое (`_execute_tool_calls`), запись результатов в историю, проверка guardrail-правил (`stall_guards`).
2. **Синхронность/Асинхронность:** Код активно использует многопоточность Python (`threading.Thread`, очереди фоновых проверок `_spawn_background_review`), блокирующие вызовы сессий SQLite с механизмом блокировок (`hermes_state_lockguard.py`).

### deepseek-harness: Реактивный цикл с проекциями (ReactLoopInbox)
В `deepseek-harness` цикл выполнения в `packages/core/agent-loop/src/agent.ts` не хранит прямое состояние в едином классе:
1. **Процесс шага (Turn):**
   - Источником истины является **Session Log** (append-only журнал). Входные данные попадают во входящую очередь `ReactLoopInbox`.
   - Перед каждым шагом вычисляются проекции (`LoopProjections`: `modelSelection()`, `inbox()`, `RuntimeContextProjection`).
   - Если есть новые входящие сообщения или результаты вызовов тулов, они верифицируются через шлюз пре-шагового решения (`PreStepDecision`).
   - Запрос к модели выполняется через потоковый драйвер `AssistantStreamAttempt`.
2. **Потоковая обработка:** Чистая асинхронность через `async/await` и события Node.js EventEmitter. Отсутствие блокировок потока ОС; все взаимодействия построены на промисах и подписках (`ctx.on('session/event', ...)`).

---

## 4. Различия в философии расширения и плагинов

### hermes-agent
* Плагины и навыки (`skills/`, `plugins/`, `optional-mcps/`) подгружаются динамически через каталог плагинов (`plugin-catalog/`).
* Навыки представляют собой скрипты/модули с метаданными, проверяемые AST-аудитом (`tools/skills_ast_audit.py`, `tool_linter.py`).
* Агент может генерировать новые навыки «на лету» в процессе выполнения и регистрировать их в собственной базе знаний.

### deepseek-harness
* Архитектура расширения базируется на плагинах Cordis. Любой пакет (MCP-клиент, песочница, парсер документов, CUA — Computer Use Agent) является плагином Cordis, объявляющим зависимости через сервисный интерфейс (`ctx.inject = ['subagents', 'llm', 'fs']`).
* Схемы настроек строго типизированы: изменение конфигурации плагина происходит через транзакционный `SettingsForms` с защитой от конфликтов версий (`SettingsConflictError`).

---

## 5. Итоговый вывод по ключевым отличиям
`hermes-agent` — это прагматичный, ориентированный на исследовательскую работу и практическое автономное применение инструмент с акцентом на самообучение и быструю интеграцию с Python-экосистемой (терминалы, системные скрипты, базы данных).

`deepseek-harness` — это высокоинженерная enterprise-платформа с микроядерным дизайном, строгой типизацией, событийным хранением состояния и четким разграничением ответственности слоев (Core, Shell, API, Presentation, Sandboxing), ориентированная на масштабируемые клиент-серверные и кроссплатформенные среды.
