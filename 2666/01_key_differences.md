# Отчёт 1: Ключевые отличия между проектами xai-org/grok-build и anomalyco/opencode

Настоящий отчёт основан исключительно на анализе исходного кода обоих репозиториев (без использования поверхностных данных из README). Анализировались исходные файлы на языках Rust (`grok-build`) и TypeScript/Effect (`opencode`).

---

## 1. Концептуальное позиционирование и назначение

| Характеристика | `grok-build` (xAI) | `opencode` (Anomaly) |
|---|---|---|
| **Основной технологический стек** | **Rust** (Cargo workspace, ~103 крейта: codegen, common, build). | **TypeScript / Bun / Node.js / Cloudflare workerd** (Turborepo monorepo, ~36 пакетов). |
| **Парадигма программирования** | Системный Rust, строгая статическая типизация, `tokio`, акторы, разделение на крейты по архитектурным границам (`#![deny(clippy::indexing_slicing)]`). | Чистый функциональный TypeScript на базе экосистемы **Effect TS** (`Layer`, `Effect`, `Scope`, `Fiber`, `Ref`, `Schema`). |
| **Фокус модели выполнения** | Высокопроизводительное системное ядро, нативная изоляция процессов на уровне ядра ОС (Landlock, Seatbelt, seccomp), строгий корпоративный контроль (MDM, Ed25519-signed policy). | Универсальная многоплатформенная среда агентного взаимодействия (CLI, Web, Desktop, Cloudflare Workers), расширяемость через TypeScript-плагины и динамический Code Mode. |
| **Модель поставщика LLM (LLM Providers)** | Заточен под стек **xAI/Grok** (Responses API, нативные server-side инструменты типа WebSearch, кастомные протоколы сэмплера). Сторонние провайдеры подключаются как слой совместимости. | **Мультипровайдерная модель первого класса**: прямая поддержка Anthropic, OpenAI, GitHub Copilot, Bedrock, Groq, Mistral, Ollama, Google AI через `@opencode/ai` и адаптеры `AISDK` (Vercel AI SDK). |

---

## 2. Модель конфигурации и глубина настроек

### `grok-build`
Конфигурационная система в `grok-build` спроектирована как многоуровневая корпоративная пирамида политик (крейты `xai-grok-config`, `xai-grok-config-types`):
1. **Строгий порядок слияния слоев (Merge Hierarchy):**
   - Системный уровень: `/etc/grok/managed_config.toml`
   - Пользовательский managed-слой: `$GROK_HOME/managed_config.toml`
   - Локальный пользовательский конфиг: `$GROK_HOME/config.toml`
   - Удаленный кэш требований с криптографической подписью: `$GROK_HOME/requirements.toml` (подписан Ed25519)
   - Общесистемные требования: `/etc/grok/requirements.toml`
   - Системные политики macOS MDM (`ai.x.grok` admin-forced managed preferences)
   - Переменные окружения и runtime-оверлеи (`GROK_SANDBOX_MODE`, кампании, оверлеи версий `[[version_overrides]]`).
2. **Fail-Closed безопасность:** Если требования безопасности (`requirements.toml` или MDM) нарушены или не прошли валидацию подписи, агент аварийно завершает работу без возможности обойти блокировку.
3. **Глубокие тонкие параметры:**
   - Память V2: MMR (Maximal Marginal Relevance), экспоненциальный временной спад (temporal decay), Dream pipeline (фоновая консолидация памяти), пороги схожести эмбеддингов (`min_score`, `max_results`).
   - Circuit Breaker: sliding window, пороги минимального количества сэмплов (`sample_count >= min_samples`) и коэффициенты ошибок (`error_rate >= error_rate_threshold`) для каждого эндпоинта.
   - Per-command sandbox rollout: режимы `Off` -> `Observe` -> `Enforce`.

### `opencode`
Конфигурация в `opencode` построена на декларативных схемах **Effect Schema** (`packages/core/src/v1/config/`, `packages/schema/`):
1. **Динамическая схема валидации:**
   - Парсинг конфигураций осуществляется через Effect Schema с сохранением порядка следования правил: в политиках безопасности применяется правило **last-match-wins**, поэтому порядок следования ключей в объекте декодируется как упорядоченный кортеж `Array<[key, value]>`.
2. **Многоуровневый контекст конфигураций:**
   - Глобальный конфиг, конфигурация проекта, контекст рабочей директории (worktree), динамические оверлеи сессий.
   - `ManagedPolicy`: политики организаций (`entry instanceof ConfigIntegration.Document`), где правила имеют источник компании, и при отказе формируется понятный отчёт ("Blocked by [org]'s policy").
3. **Модель провайдеров и моделей:**
   - Глубокая параметризация моделей: `model`, `variant`, `temperature`, `top_p`, `steps` (лимит итераций до принудительного перехода на текст), `tools`, `permission`.
   - Поддержка динамической подгрузки npm-пакетов для кастомных провайдеров (`runtime-import`).

---

## 3. Выполнение инструментов и среда рантайма

### `grok-build`
- **Системные PTY и процессы:** Собственный демон терминалов `ptyctl`, базирующийся на крейте `alacritty_terminal`. Управление терминалом на уровне эмулятора: генерация ANSI/VT эмуляции, захват styled HTML, REST API для PTY-сессий.
- **ОС-изоляция (Sandboxing):** Прямая интеграция с ядром через библиотеку `nono` (Linux **Landlock**, macOS **Seatbelt**, seccomp для блокировки сетевых вызовов в дочерних процессах).
- **Сетевой контроль (Egress Proxy):** Встроенный локальный HTTP/CONNECT-прокси (`xai-grok-egress-proxy`) с per-call мандатами (per-call credentials в URL userinfo), валидацией хостов, отслеживанием нарушений политик и перехватом DNS/TLS.

### `opencode`
- **Persistent PTY Даемон:** Реализован на TypeScript/Node/Bun с абстракцией биндингов `#persistent-pty-binary`, поддержкой Node PTY и Bun PTY, а также фолбэком на Cloudflare workerd. Сессии PTY сохраняются в `$STATE/pty`.
- **Code Mode:** Уникальный механизм, позволяющий LLM исполнять JavaScript/TypeScript-код в изолированном пространстве для вызова комплексных цепочек инструментов за одну итерацию вместо отправки десятков разрозненных сообщений ToolCall.
- **Расширяемость через плагины:** Встроенная система плагинов через шину событий (`Bus.Service`) и жизненные циклы сессий Effect.

---

## 4. Парадигма сессий и жизненного цикла

- **`grok-build`:** Агент компилируется в жесткую, практически неизменяемую структуру (`Agent`, создаваемая через `AgentBuilder`). Все изменения состояния инструментов происходят через внутренние мьютексы `Arc<ToolBridge>`. Очередь промптов управляется крейтом `xai-prompt-queue` с умным слиянием запросов (`combine_queued_prompts`).
- **`opencode`:** Сессия — это динамический реактивный актор на базе Effect Fibers (`SessionExecution`, `SessionRunCoordinator`). Координатор поддерживает механизм "дверного звонка" (`wake`), позволяющий безопасно ставить задачи в очередь и объединять одновременные запросы без блокировки цикла ввода-вывода.

---

## Сводная таблица ключевых различий

| Критерий | `grok-build` | `opencode` |
|---|---|---|
| **Ядро** | Rust (системная надежность, нулевая стоимость абстракций) | TypeScript + Effect TS (высокая скорость разработки, функциональная композиция) |
| **Безопасность песочницы** | Аппаратная/ядерная: Landlock, Seatbelt, seccomp, egress proxy | Логическая / процессная: списки разрешений, wildcard-политики, PTY-демон |
| **Подключение LLM** | Нативное xAI API, server-side tools (Grok WebSearch), строгий контракт | Vercel AI SDK, 10+ провайдеров, локальные LLM (Ollama), Copilot |
| **Интеграция с ОС** | Глубокая: MDM macOS, `/etc`, headless PTY alacritty, Landlock | Кроссплатформенная: Bun/Node/workerd, локальный SQLite, Web UI, Desktop |
| **Вызов инструментов** | Строгий протокол `xai-tool-runtime`, JSON-RPC / MCP | Двойной: стандартный ToolCall + пакетный Code Mode (JS/TS sandbox) |
