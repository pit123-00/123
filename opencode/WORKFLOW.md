# Workflow: От задачи до реализации

Полная инструкция по последовательному использованию всех команд OpenCode.

---

## Общая схема

```
Задача пользователя
       │
       ▼
  ┌─────────────┐
  │  /research  │ ← Фаза 0: Сбор фактов о кодовой базе
  └──────┬──────┘
         │ thoughts/research/YYYY-MM-DD-topic.md
         ▼
  ┌──────────────────────┐
  │  /generate-standards │ ← Фаза 0.5: Извлечение стандартов из кода
  └──────┬───────────────┘
         │ prompts/*.md (7 файлов)
         ▼
  ┌───────────────┐
  │  /enrich-task │ ← Фаза 1: Выявление пробелов в задаче (итеративно)
  └──────┬────────┘
         │ обогащённый файл задачи
         ▼
  ┌──────────────────┐
  │  /design-feature │ ← Фаза 2: Проектирование архитектуры
  └──────┬───────────┘
         │ docs/feature/README.md
         ▼
  ┌────────────────┐
  │  /plan-feature │ ← Фаза 3: Пошаговый план реализации
  └───────┬────────┘
          │ docs/feature/plan/README.md
          ▼
  ┌─────────────────────┐
  │  /implement-feature │ ← Фаза 4: Mob-programming реализация
  └─────────────────────┘
```

---

## Фаза 0: Research — Сбор фактов

**Команда:** `/research`

**Что делает:** Исследует кодовую базу по заданной задаче. Спавнит субагентов `codebase-researcher` для параллельного анализа. Выдаёт ТОЛЬКО факты с `file:line` ссылками.

**Аргументы:**
```
/research <задача или путь к файлу задачи> [scope]
```
- `задача` — (ОБЯЗАТЕЛЬНО) текст задачи или путь к файлу (например `docs/tickets/add-notifications.md`)
- `scope` — (опционально) стартовая директория (например `services/`, `bot/`)

**Пример:**
```
/research docs/tickets/add-notifications.md services/
```

**Результат:** `thoughts/research/YYYY-MM-DD-topic-name.md`

**Когда запускать:**
- Один раз в начале работы над новой задачей
- Повторно — если задача существенно изменилась

---

## Фаза 0.5: Generate Standards — Извлечение стандартов

**Команда:** `/generate-standards`

**Что делает:** Читает research-документ + сканирует кодовую базу → генерирует 7 файлов стандартов в `prompts/`. Стандарты извлекаются ИЗ кода — не из учебников.

**Аргументы:**
```
/generate-standards <путь к research-документу> [scope]
```
- `research_path` — (ОБЯЗАТЕЛЬНО) путь к файлу ресёрча
- `scope` — (опционально) ограничить область

**Пример:**
```
/generate-standards thoughts/research/2026-03-01-complete-codebase-architecture.md
```

**Результат:** 7 файлов в `prompts/`:
```
prompts/
├── README.md              — Индекс
├── architecture.md        — Слои, модули, направление зависимостей
├── clean-architecture.md  — Границы, DI, правила слоёв
├── domain-models.md       — Сущности, value objects, инварианты
├── pydantic-schemas.md    — Схемы, валидация, DTO
├── sqlalchemy-models.md   — ORM модели, маппинг, миграции
├── python-style.md        — Стиль кода, именование, типы
└── tests-style.md         — pytest, фикстуры, моки
```

**Когда запускать:**
- Один раз на старте проекта
- Повторно — после крупных архитектурных изменений

> ⚠️ **Важно:** Запускай ПОСЛЕ research, потому что стандарты генерируются из research-документа.

---

## Фаза 1: Enrich Task — Выявление пробелов

**Команда:** `/enrich-task`

**Что делает:** Анализирует задачу на пробелы по 5 измерениям (Data Flow, Integration, Architecture, Business Logic, Edge Cases). Генерирует структурированные вопросы с приоритетами. Итеративно обогащает задачу ответами пользователя.

**Аргументы:**
```
/enrich-task <путь к задаче> <путь к ресёрчу> [путь к Q&A файлу]
```
- `task_path` — (ОБЯЗАТЕЛЬНО) путь к файлу задачи
- `research_path` — (ОБЯЗАТЕЛЬНО) путь к файлу ресёрча
- `qa_feedback_path` — (опционально) заполненный Q&A файл от предыдущей итерации

**Пример — первый запуск:**
```
/enrich-task docs/tickets/add-notifications.md thoughts/research/2026-03-01-notifications.md
```

**Пример — повторный запуск с ответами:**
```
/enrich-task docs/tickets/add-notifications.md thoughts/research/2026-03-01-notifications.md docs/tickets/add-notifications-questions-01.md
```

**Результат:** Q&A файл(ы) рядом с задачей:
```
docs/tickets/add-notifications-questions-01.md           ← одиночный файл (≤10 вопросов)
docs/tickets/add-notifications-questions-02-part-01.md   ← multi-part (>10 вопросов)
docs/tickets/add-notifications-questions-02-part-02.md
```

**Лимит вопросов на файл:**

> ⚠️ Максимум **10 вопросов на файл** (`MAX_QUESTIONS_PER_FILE = 10`).  
> Если вопросов больше — они автоматически разбиваются на part-файлы (`-part-01`, `-part-02`, ...).  
> Каждый part самодостаточный (свой header, инструкции, статистика).  
> Нумерация вопросов глобальная: Q1–Q10 в part-01, Q11–Q20 в part-02.  
> При заполнении — заполняйте **ВСЕ** parts, затем запускайте `/enrich-task` **ОДИН РАЗ** с любым part-файлом.  
> Система автоматически найдёт и применит все части через `discover_all_parts()`.

**Цикл работы:**
```
1. Запустить /enrich-task (без Q&A)
2. Открыть сгенерированный файл вопросов
3. Заполнить ответы (отметить [x] вариант + написать пояснение)
4. Запустить /enrich-task снова (с Q&A файлом)
5. Повторять пока 🔴 критических вопросов = 0
6. Перейти к /design-feature
```

**Субагенты:** `data-flow-analyst`, `arch-analyst`, `edge-case-analyst` (для сложных задач)

> 💡 **Совет:** Если ты уже знаешь ответы — впиши их прямо в файл задачи и пропусти итерации Q&A.

---

## Фаза 2: Design — Проектирование

**Команда:** `/design-feature`

**Что делает:** Создаёт архитектурный дизайн фичи. C4 Model (Context → Container → Component → Code), DFD, Sequence диаграммы, ADR. Проходит ревью через субагент `architect-reviewer`.

**Аргументы:**
```
/design-feature <имя-фичи> <путь-к-сервису> <описание или путь к задаче>
```
- `feature_name` — (ОБЯЗАТЕЛЬНО) slug-имя фичи (для директории)
- `service_path` — (ОБЯЗАТЕЛЬНО) путь к сервису (например `services/prediction_markets` или `bot`)
- `description` — (ОБЯЗАТЕЛЬНО) описание, требования или путь к обогащённому файлу задачи

**Пример:**
```
/design-feature add-notifications bot docs/tickets/add-notifications.md
```

**Результат:** Дизайн-документация:
```
bot/docs/add-notifications/
├── README.md              — Основной дизайн (C4, DFD, последовательности)
├── 01-logical-view.md     — C4 модель
├── 02-process-view.md     — DFD + Sequence диаграммы
├── 03-decisions.md        — ADR (Architectural Decision Records)
├── 04-quality.md          — Стратегия тестирования
└── 07-standards.md        — Применимые стандарты
```

**Когда запускать:**
- ПОСЛЕ того как задача обогащена (0 критических вопросов)
- ПОСЛЕ того как стандарты сгенерированы (они нужны для Phase 0.3 дизайна)

> ⚠️ **Gate:** Дизайн требует одобрения пользователя перед переходом к плану.

---

## Фаза 3: Plan — Пошаговый план

**Команда:** `/plan-feature`

**Что делает:** Читает дизайн-документ и создаёт детальный пошаговый план реализации. Разбивает на фазы, каждая фаза — на конкретные изменения файлов.

**Аргументы:**
```
/plan-feature <путь к design README.md>
```
- `design_readme` — (ОБЯЗАТЕЛЬНО) путь к основному дизайн-документу

**Пример:**
```
/plan-feature bot/docs/add-notifications/README.md
```

**Результат:** План в поддиректории `plan/`:
```
bot/docs/add-notifications/plan/
├── README.md              — Основной план (фазы, порядок, зависимости)
├── phase-01.md            — Конкретные изменения файлов фазы 1
├── phase-02.md            — Конкретные изменения файлов фазы 2
└── ...
```

**Когда запускать:**
- ТОЛЬКО после одобрения дизайна пользователем

> ⚠️ **Gate:** План требует одобрения пользователя перед переходом к реализации.

---

## Фаза 4: Implement — Реализация

**Команда:** `/implement-feature`

**Что делает:** Mob-programming реализация по плану. Создаёт команду из 5 субагентов (implementer + 4 reviewer), выполняет план фаза за фазой, каждый коммит проходит 4 ревью.

**Аргументы:**
```
/implement-feature <путь к plan README.md>
```
- `plan_readme` — (ОБЯЗАТЕЛЬНО) путь к основному файлу плана

**Пример:**
```
/implement-feature bot/docs/add-notifications/plan/README.md
```

**Результат:**
- Реализованный код по всем фазам плана
- Пройденные ревью (build, architecture, security, plan compliance)
- Smoke test
- Коммит с дизайн-документацией

**Команда субагентов:**

| Роль | Что делает |
|------|-----------|
| `implementer` | Пишет код по плану |
| `rv-build` | Проверяет: собирается ли, проходят ли тесты |
| `rv-arch` | Проверяет: соответствует ли дизайну и стандартам |
| `rv-sec` | Проверяет: безопасность, инъекции, утечки |
| `rv-plan` | Проверяет: всё ли из плана реализовано |

> ⚠️ **Gate:** Финальный human review перед push — ОБЯЗАТЕЛЕН.

---

## Полный пример: от задачи до кода

```bash
# 1. Написать задачу
# Создай файл docs/tickets/add-telegram-notifications.md с описанием

# 2. Research — собрать факты
/research docs/tickets/add-telegram-notifications.md bot/

# 3. Standards — сгенерировать стандарты (если ещё не сделано)
/generate-standards thoughts/research/2026-03-01-telegram-notifications.md

# 4. Enrich — найти пробелы
/enrich-task docs/tickets/add-telegram-notifications.md thoughts/research/2026-03-01-telegram-notifications.md

# 5. Заполнить ответы в файле вопросов...
# 6. Повторить enrich с ответами
/enrich-task docs/tickets/add-telegram-notifications.md thoughts/research/2026-03-01-telegram-notifications.md docs/tickets/add-telegram-notifications-questions-01.md

# 7. Когда 0 критических вопросов → Design
/design-feature add-telegram-notifications bot docs/tickets/add-telegram-notifications.md

# 8. Ревью дизайна → Одобрение → Plan
/plan-feature bot/docs/add-telegram-notifications/README.md

# 9. Ревью плана → Одобрение → Implement
/implement-feature bot/docs/add-telegram-notifications/plan/README.md

# 10. Human review → Push
```

---

## Быстрая справка

| Фаза | Команда | Вход | Выход |
|------|---------|------|-------|
| 0 | `/research` | задача + scope | `thoughts/research/*.md` |
| 0.5 | `/generate-standards` | research-файл | `prompts/*.md` |
| 1 | `/enrich-task` | задача + research + [Q&A] | Q&A файл |
| 2 | `/design-feature` | имя + сервис + задача | дизайн docs |
| 3 | `/plan-feature` | design README | план по фазам |
| 4 | `/implement-feature` | plan README | код + ревью + коммит |

## Субагенты

| Агент | Используется в | Роль |
|-------|---------------|------|
| `codebase-researcher` | `/research`, `/generate-standards` | Сканирование файлов, сбор фактов |
| `architect-reviewer` | `/design-feature` | Ревью дизайна |
| `data-flow-analyst` | `/enrich-task` | Анализ потоков данных + интеграций |
| `arch-analyst` | `/enrich-task` | Анализ архитектуры + бизнес-логики |
| `edge-case-analyst` | `/enrich-task` | Анализ граничных случаев + ошибок |

## Gates (точки одобрения)

```
Research    ──→ Standards   (автоматический переход)
Standards   ──→ Enrich Task (автоматический переход)
Enrich Task ──→ Design      (автоматический: когда 🔴 = 0)
Design      ──→ Plan        (⛔ требует одобрения пользователя)
Plan        ──→ Implement   (⛔ требует одобрения пользователя)
Implement   ──→ Push        (⛔ требует human review)
```

---

## Чего НЕ делать

❌ Пропускать research и сразу идти в design — дизайн будет основан на догадках, а не на фактах  
❌ Пропускать generate-standards — дизайн не будет знать паттерны проекта  
❌ Пропускать enrich-task для сложных задач — реализация столкнётся с неопределёнными требованиями  
❌ Запускать implement без одобрения плана — код может пойти не в ту сторону  
❌ Push без human review — всегда проверяй финальный результат  

## Когда можно сократить

✅ Простая задача (баг-фикс, мелкий рефакторинг) — можно пропустить enrich-task  
✅ Standards уже сгенерированы — не надо запускать повторно  
✅ Ты уже знаешь все ответы — вписать в задачу руками вместо итераций Q&A  

---

# Bug Fix Pipeline

> *"Фича и баг это совершенно разные задачи. У фичи нужна архитектура, C4, API контракты. А у бага — как воспроизвести, какие причины, какое минимальное исправление."*

Баг-фикс — это **хирургическая операция**: войти, починить, написать тест, проверить что ничего не сломалось, выйти.

---

## Схема Bug Fix

```
Баг-репорт
     │
     ▼
  ┌──────────────────────┐
  │  /enrich-bug-report  │ ← Фаза B0.5: Обогащение баг-репорта (итеративно)
  └──────┬───────────────┘
         │ обогащённый баг-репорт
         ▼
  ┌──────────────┐
  │  /research   │ ← Фаза B0: Сбор фактов, трассировка бага
  └──────┬───────┘
         │ thoughts/research/YYYY-MM-DD-topic.md
         ▼
  ┌───────────────────┐
  │  /design-bug-fix  │ ← RCA + Impact + Minimal Fix Strategy
  └──────┬────────────┘
         │ docs/bug-name/README.md (один файл!)
         ▼
  ⛔ Human approve RCA + стратегию фикса
         │
         ▼
  ┌─────────────────┐
  │  /plan-bug-fix  │ ← 1-3 фазы (не 5-8 как у фичи!)
  └──────┬──────────┘
         │ docs/bug-name/plan/
         ▼
  ⛔ Human approve план
         │
         ▼
  ┌─────────────┐
  │  /fix-bug   │ ← Mob team: impl + build + minimality + security
  └──────┬──────┘
         │
         ▼
  ⛔ Human review → push
```

---

## Ключевые отличия от Feature Pipeline

| Аспект | Feature | Bug Fix |
|--------|---------|---------|
| **Research** | Что строим | Где баг, почему |
| **Standards** | Нужны | Уже сгенерированы |
| **Enrich** | `/enrich-task` — 5 измерений пробелов | `/enrich-bug-report` — 6 измерений баг-репорта |
| **Design** | C4, DFD, ADR, 5+ файлов | RCA + Impact + Fix, 1 файл |
| **Plan** | 5-8 фаз | 1-3 фазы |
| **Implement** | 5 агентов (impl, build, arch, sec, plan) | 4 агента (impl, build, **minimality**, sec) |
| **Ревьюер** | `rv-arch` (архитектура) | `rv-minimality` (минимальность) |

---

## Фаза B0.5: Enrich Bug Report — Обогащение баг-репорта

**Команда:** `/enrich-bug-report`

**Что делает:** Анализирует баг-репорт на пробелы по 6 измерениям (Symptom, Reproduction, Environment, Frequency, Data, Timeline). Генерирует структурированные вопросы с приоритетами. Итеративно обогащает баг-репорт ответами пользователя.

**Аргументы:**
```
/enrich-bug-report <путь к баг-репорту> <путь к ресёрчу> [путь к Q&A файлу]
```
- `bug_path` — (ОБЯЗАТЕЛЬНО) путь к файлу баг-репорта
- `research_path` — (ОБЯЗАТЕЛЬНО) путь к файлу ресёрча
- `qa_feedback_path` — (опционально) заполненный Q&A файл от предыдущей итерации

**Пример — первый запуск:**
```
/enrich-bug-report docs/bugs/portfolio-rounding.md thoughts/research/2026-03-01-portfolio.md
```

**Пример — повторный запуск с ответами:**
```
/enrich-bug-report docs/bugs/portfolio-rounding.md thoughts/research/2026-03-01-portfolio.md docs/bugs/portfolio-rounding-questions-01.md
```

**Результат:** Q&A файл(ы) рядом с баг-репортом:
```
docs/bugs/portfolio-rounding-questions-01.md           ← одиночный файл (≤10 вопросов)
docs/bugs/portfolio-rounding-questions-02-part-01.md   ← multi-part (>10 вопросов)
docs/bugs/portfolio-rounding-questions-02-part-02.md
```

**Лимит вопросов на файл:**

> ⚠️ Максимум **10 вопросов на файл** (`MAX_QUESTIONS_PER_FILE = 10`).  
> Если вопросов больше — они автоматически разбиваются на part-файлы (`-part-01`, `-part-02`, ...).  
> Каждый part самодостаточный (свой header, инструкции, статистика).  
> Нумерация вопросов глобальная: Q1–Q10 в part-01, Q11–Q20 в part-02.  
> При заполнении — заполняйте **ВСЕ** parts, затем запускайте `/enrich-bug-report` **ОДИН РАЗ** с любым part-файлом.  
> Система автоматически найдёт и применит все части через `discover_all_parts()`.

**6 измерений анализа:**

| Измерение | Что проверяет |
|-----------|--------------|
| **Symptom** | Что ИМЕННО не работает? Ошибка? Скриншот? |
| **Reproduction** | Шаги воспроизведения? Какие данные? |
| **Environment** | Prod/staging/local? Версия? OS? |
| **Frequency** | Всегда? Иногда? При каких условиях? |
| **Data** | Какой пользователь? Какие ID? Входные данные? |
| **Timeline** | Когда началось? Работало ли раньше? Что изменилось? |

**Цикл работы:**
```
1. Запустить /enrich-bug-report (без Q&A)
2. Открыть сгенерированный файл вопросов
3. Заполнить ответы (отметить [x] вариант + написать пояснение)
4. Запустить /enrich-bug-report снова (с Q&A файлом)
5. Повторять пока 🔴 критических вопросов = 0
6. Перейти к /research + /design-bug-fix
```

**Отличия от `/enrich-task`:**
- 6 измерений (баг-специфичные) вместо 5 (фича-специфичных)
- Без субагентов — всегда последовательный анализ (легче)
- Без стандартов — не нужны `prompts/*.md`, только баг-репорт + research
- Фокус: недостающие ФАКТЫ для диагностики, а не архитектурные пробелы

> 💡 **Совет:** Если ты точно знаешь всю информацию о баге — впиши всё в баг-репорт и пропусти этот шаг.

> ⚠️ **Когда нужен:** Когда баг-репорт неполный — пользователь не знает откуда баг, нет шагов воспроизведения, нет данных. Для детальных баг-репортов — можно пропустить.

---

## Фаза B0: Research (тот же)

```
/research <баг-репорт или файл> [scope]
```

Фокус: найти ГДЕ баг живёт, трассировка кода, стек вызовов.

---

## Фаза B1: Design Bug Fix — Диагностика

**Команда:** `/design-bug-fix`

**Что делает:** Root Cause Analysis, Impact Analysis, Minimal Fix Strategy.

**Аргументы:**
```
/design-bug-fix <имя-бага> <путь-к-сервису> <описание или путь к файлу>
```

**Пример:**
```
/design-bug-fix fix-portfolio-calc bot docs/bugs/portfolio-rounding.md
```

**Результат:** Один файл `{service}/docs/{bug-name}/README.md` с:
- Reproduction steps (как воспроизвести)
- Root Cause Analysis (ПОЧЕМУ, не ГДЕ)
- Impact Analysis (что ещё затронуто)
- Fix Strategy (1-2 минимальных варианта)
- Regression Test Plan (обязателен)

**Фазы внутри:**
```
Phase 1: Reproduce   → шаги воспроизведения + трассировка
Phase 2: RCA         → цепочка: origin → propagation → symptom
Phase 3: Impact      → blast radius + риски от фикса
Phase 4: Fix         → минимальный фикс + regression test plan
```

---

## Фаза B2: Plan Bug Fix

**Команда:** `/plan-bug-fix`

**Что делает:** Превращает одобренный RCA в исполняемый план. 1-3 фазы максимум.

**Аргументы:**
```
/plan-bug-fix <путь к design README.md>
```

**Пример:**
```
/plan-bug-fix bot/docs/fix-portfolio-calc/README.md
```

**Результат:**
```
bot/docs/fix-portfolio-calc/plan/
├── README.md     — обзор плана
├── phase-01.md   — фикс (конкретные file:line before/after)
├── phase-02.md   — regression test
└── phase-03.md   — верификация (если высокий риск)
```

**Правила кол-ва фаз:**
- 1-2 файла, низкий риск → **1 фаза** (фикс + тест вместе)
- Несколько файлов → **2 фазы** (фикс → тест)
- Высокий риск → **3 фазы** (фикс → тест → верификация)

---

## Фаза B3: Fix Bug — Реализация

**Команда:** `/fix-bug`

**Что делает:** Mob programming с командой из 4 агентов. Ключевой ревьюер — `rv-minimality`.

**Аргументы:**
```
/fix-bug <путь к plan README.md>
```

**Пример:**
```
/fix-bug bot/docs/fix-portfolio-calc/plan/README.md
```

**Команда:**

| Роль | Что делает |
|------|-----------|
| `implementer` | Пишет ТОЛЬКО то, что в плане |
| `rv-build` | Build + тесты + lint |
| `rv-minimality` | ⭐ Не добавил ли лишнего? Файлы только из плана? |
| `rv-sec` | Безопасность |

> ⭐ **`rv-minimality`** — ключевое отличие от feature pipeline. Он проверяет что фикс МИНИМАЛЕН и точно соответствует плану.

---

## Полный пример: от бага до фикса

```bash
# 1. Написать баг-репорт
# Создай файл docs/bugs/portfolio-rounding.md с описанием

# 2. Research — первичный сбор фактов (нужен для enrich-bug-report)
/research docs/bugs/portfolio-rounding.md services/

# 3. Enrich Bug Report — обогатить баг-репорт (если он неполный)
/enrich-bug-report docs/bugs/portfolio-rounding.md thoughts/research/2026-03-01-portfolio.md

# 4. Заполнить ответы → повторить enrich → пока 🔴 = 0

# 5. Research — повторный (если баг-репорт существенно дополнился)
/research docs/bugs/portfolio-rounding.md services/

# 6. Design Bug Fix — RCA + минимальный фикс
/design-bug-fix fix-portfolio-calc services/prediction_markets docs/bugs/portfolio-rounding.md

# 7. Ревью RCA → Одобрение → Plan
/plan-bug-fix services/prediction_markets/docs/fix-portfolio-calc/README.md

# 8. Ревью плана → Одобрение → Fix
/fix-bug services/prediction_markets/docs/fix-portfolio-calc/plan/README.md

# 9. Human review → Push
```

---

## Быстрая справка: Bug Fix

| Фаза | Команда | Вход | Выход |
|------|---------|------|-------|
| B0 | `/research` | баг-репорт + scope | `thoughts/research/*.md` |
| B0.5 | `/enrich-bug-report` | баг-репорт + research + [Q&A] | Q&A файл (обогащённый баг-репорт) |
| B1 | `/design-bug-fix` | имя + сервис + баг | 1 файл: RCA + fix strategy |
| B2 | `/plan-bug-fix` | design README | план 1-3 фазы |
| B3 | `/fix-bug` | plan README | фикс + regression test + коммит |

## Субагенты и ревьюеры Bug Fix

| Агент | Используется в | Тип | Роль |
|-------|---------------|-----|------|
| `codebase-researcher` | `/research` | Внешний субагент | Трассировка бага |
| `rv-minimality` | `/fix-bug` | Inline (в команде) | ⭐ Проверка минимальности фикса |
| `rv-build` | `/fix-bug` | Inline (в команде) | Build + тесты + lint |
| `rv-sec` | `/fix-bug` | Inline (в команде) | Безопасность |

---

## Чего НЕ делать при баг-фиксе

❌ Рефакторить "заодно" — если код некрасивый, создай отдельный тикет  
❌ Добавлять фичи — баг-фикс это ТОЛЬКО фикс  
❌ Чинить симптом — найди root cause  
❌ Пропускать regression test — без теста фикс не принимается  
❌ Делать 5+ фаз — если план такой большой, это не баг, а фича  

## Когда эскалировать в фичу

🔄 Root cause — архитектурная проблема → `/design-feature`  
🔄 Фикс требует новые зависимости → `/design-feature`  
🔄 Фикс затрагивает 5+ файлов → пересмотри, может это фича  

---

## Расположение файлов

```
.opencode/
├── WORKFLOW.md                  ← Этот файл
├── agents/
│   ├── arch-analyst.md          — Gap-аналитик: архитектура + бизнес-логика
│   ├── architect-reviewer.md    — Ревьюер дизайна фичи
│   ├── codebase-researcher.md   — Сканирование файлов, сбор фактов
│   ├── data-flow-analyst.md     — Gap-аналитик: потоки данных + интеграции
│   └── edge-case-analyst.md     — Gap-аналитик: граничные случаи + ошибки
├── commands/
│   ├── research.md              — Фаза 0: Research
│   ├── generate-standards.md    — Фаза 0.5: Генерация стандартов
│   ├── enrich-task.md           — Фаза 1: Обогащение задачи
│   ├── design-feature.md        — Фаза 2: Дизайн фичи
│   ├── plan-feature.md          — Фаза 3: План фичи
│   ├── implement-feature.md     — Фаза 4: Реализация фичи
│   ├── enrich-bug-report.md     — Фаза B0.5: Обогащение баг-репорта
│   ├── design-bug-fix.md        — Фаза B1: Диагностика бага (RCA)
│   ├── plan-bug-fix.md          — Фаза B2: План фикса
│   └── fix-bug.md               — Фаза B3: Реализация фикса
```
