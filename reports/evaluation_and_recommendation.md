# Сравнительная оценка и руководство по реализации системы

На основе детального анализа исходного кода трех проектов (`bytedance/deer-flow`, `anomalyco/opencode`, `deepseek-ai/deepseek-harness`) и сопоставления с требованиями спецификации `AGENT_WORKFLOW_FRAMEWORK_RU.md`.

---

## 1. Сравнительная матрица соответствия требованиям фреймворка

| Требование / Принцип фреймворка (`AGENT_WORKFLOW_FRAMEWORK_RU.md`) | `deer-flow` (ByteDance) | `opencode` (Anomaly) | `deepseek-harness` (DeepSeek) |
|---|---|---|---|
| **Языковой стек и скрипты** (`runner.py`, `registry.py`, `delivery.py`) | **100% Python** — прямое совпадение со спецификацией | TypeScript / Node / Bun | TypeScript / Node / Cordis |
| **Изоляция контекста субагента** (`fork_turns="none"`, чистый старт) | `ParentContextSnapshot` с очисткой служебных сообщений | Новая дочерняя сессия `SubagentJob` | `subagent-spawn-in-process` (zero context) |
| **Детерминированные квитанции** (`ToolReceipt`, `output_sha256`, `r1..rN`) | **Готовая реализация** (`tool_receipt.py`, `receipt_verification.py`) | Отсутствует | Отсутствует |
| **Ограничение прав и защита от перезаписи** (Read-Before-Write, path guard) | **Готовая реализация** (`read_before_write_middleware.py`) | Отличная модель RBAC (`permission.ts`, `file-access.ts`) | Ограничение глубины (`depth.ts`) |
| **Контракт отчета и цитирование** (проверка сдачи без LLM) | **Готовая реализация** (`report_contract.py`, `batch_acceptance.py`) | Нет (LLM сам пишет ответ) | Схема структурированного вывода (`structured.ts`) |
| **Конечно-автоматное управление** (State machine, атомарные переходы) | `SubagentBatchSubmitter`, транзакции, лимиты | `SessionRunCoordinator` (сериализация, wake) | `TeamTaskBoard`, журнал событий |

---

## 2. Вердикт: какой проект технически больше подходит?

### Победитель: **`bytedance/deer-flow`**

### Обоснование выбора:
1. **Прямое архитектурное родство:**
   Спецификация `AGENT_WORKFLOW_FRAMEWORK_RU.md` буквально построена на концепциях:
   - детерминированных квитанций вызова инструментов (`tool_receipt`),
   - проверки цитирования квитанций родителем (`receipt_verification`),
   - шлюза обязательного предварительного чтения (`read_before_write`),
   - очистки снимка родительского контекста (`ParentContextSnapshot`),
   - программной приемки критериев выполнения (`batch_acceptance`).
   В `deer-flow` **все эти модули уже написаны на Python и протестированы**.
2. **Языковая экосистема:**
   И скрипты фреймворка (`workflow/scripts/*.py`), и бэкенд `deer-flow` написаны на Python. Вам не придется переписывать сложные асинхронные структуры Effect-TS или Cordis с TypeScript на Python.
3. **Готовые механизмы безопасности:**
   В `deer-flow` решена проблема prompt injection в критериях приемки: системные промпты строго отделены от пользовательских данных, теги нейтрализуются (`neutralize_untrusted_tags`).

---

## 3. Что взять из двух других проектов (гибридный подход)

Хотя основным фундаментом выступает `deer-flow`, из других проектов крайне полезно взять архитектурные решения:

1. **Из `anomalyco/opencode`:**
   - **Строгие лексические границы песочницы (`file-access.ts`):** Реализовать в `scripts/validate.py` нормализацию абсолютных путей и жесткую проверку того, что путь назначения не покидает разрешенную папку `outputs/.workflow/runs/<run_id>/candidates/` (защита от `../` и symlink-атак).
   - **Паттерн очереди и координации сессий (`run-coordinator.ts`):** Гарантия того, что в один момент времени выполняется не более одного субагента, а повторный вызов идемпотентно возвращает сохраненную квитанцию.
2. **Из `deepseek-ai/deepseek-harness`:**
   - **Принцип `subagent-spawn-in-process`:** Запуск субагента в режиме полного нуля истории (`zero parent context`). Субагент стартует без истории диалога оркестратора, получая строго файл контракта задания через CLI-команду чтения.
   - **Контроль глубины делегирования (`depth.ts`):** Запрет субагентам порождать дочерних субагентов (фиксируется в контракте и проверяется в реестре: если `agent_id != orchestrator`, создание субагентов блокируется).

---

## 4. Пошаговый план практической реализации фреймворка

### Шаг 1. Структура проекта и схемы контрактов (JSON Schema)
Создать структуру:
```text
workflow/
  contracts/
    assignment.schema.json
    result.schema.json
    patch.schema.json
  scripts/
    registry.py
    delivery.py
    validate.py
    patch.py
    runner.py
```
Взять из `deer-flow/subagents/report_contract.py` и `deepseek-harness/subagent/structured.ts` принципы валидации JSON через JSON Schema.

### Шаг 2. Реализация реестра состояния (`registry.py`)
- Взять за основу идемпотентную запись и блокировки запусков из `deer-flow` (`persistence/`, `batch_lease.py`).
- Хранить состояние запуска в `outputs/.workflow/runs/<run_id>/registry.json`.
- Запись выполнять атомарно: запись во временный файл `registry.json.tmp` -> атомарное переименование `os.replace`.
- Поддерживать граф переходов:
  `PREPARED → BOUND → READING → WORKING → CHECKING → COMPLETED` (и возврат в `REPAIR` при замечаниях).

### Шаг 3. Реализация управляемого чтения порциями (`delivery.py`)
- Реализовать постраничную разбивку контента UTF-8 с лимитом (например, 12000 байт).
- Каждому чанку присваивать `part_id`, `content_sha256`, обрамление маркерами `=== PART {id} ===` и `=== END_PART {id} ===`.
- Выдавать субагенту команду следующего шага:
  `python workflow/scripts/delivery.py ack --run-id <id> --task-id <id> --part-id <part_id>`
- Пока все части обязательного контекста не подтверждены, `registry.py` блокирует прием кандидата.

### Шаг 4. Слой квитанций и Read-Before-Write (`validate.py`)
- Портировать логику из `deerflow/agents/middlewares/tool_receipt.py` и `receipt_verification.py`.
- Каждый выполненный инструмент пишет результат и получает квитанцию с `sha256`.
- Валидатор проверяет, что автор приложил валидные `[rN]` квитанции на созданные файлы и результаты.
- Защита Read-Before-Write (`read_before_write_middleware.py`): если режим `PATCH`, исполнитель обязан предъявить хеш прочитанного исходника перед фиксацией замены.

### Шаг 5. Применение точечных правок (`patch.py`)
- Скрипт принимает JSON с заменами:
  `{ "replacements": [{ "section_id": "...", "replacement_text": "..." }] }`
- Проверяет `allowed_sections` из контракта задания. Если субагент попытался изменить другой раздел, патч отклоняется со статусом `ERROR`.

### Шаг 6. Оркестратор (`runner.py`)
- Читает текущее состояние запуска из реестра.
- Выдает агенту одну четкую следующую операцию: `READ`, `WRITE_RESULT`, `CHECK`, `REPAIR_THEN_RETRY` или `FINISH`.
