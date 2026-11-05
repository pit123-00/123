# Отчёт №5: Применение маскирования и механизмов безопасности из ai-proxy в EasyCLIProxyAPI

Данный отчёт содержит практическое руководство и архитектурный анализ того, как перенести ключевые защитные технологии из проекта `ai-proxy` (маскирование секретов, энтропийный анализ, потоковое восстановление данных, аудит вызовов инструментов) в экосистему `EasyCLIProxyAPI`.

Анализ выполнен строго на основе исходного кода обоих проектов.

---

## 1. Анализ точек расширения в архитектуре EasyCLIProxyAPI

В кодовой базе `EasyCLIProxyAPI` существуют три естественные архитектурные точки интеграции функционала:

1. **Система плагинов ядра на Go (`plugins/`)**:
   - Ядро `cli-proxy-api` поддерживает динамические нативные библиотеки (`.so`, `.dylib`, `.dll`) через стабильный C-ABI (`cliproxy_plugin_init`, `routeCall`, `pluginabi`, `pluginapi`).
   - Манифест плагина (`manifest.json`) поддерживает декларацию возможностей (`capabilities`).
   - Плагины получают входящие вызовы методов, имеют доступ к хукам вызовов моделей (`pluginabi.MethodModelRoute`, `pluginabi.MethodHostModelExecute`), а также к сетевому клиенту хоста (`pluginabi.MethodHostHTTPDo`).
2. **Встроенный Rust TLS Reverse Proxy (`reverse_proxy.rs`)**:
   - Внутри настольного клиента Tauri уже реализован полноценный обратный прокси на `hyper 1.0` и `rustls`.
   - Он принимает локальный трафик, производит hardening (привязка ядра к `127.0.0.1`, фильтрация доверенных прокси) и пересылает запросы в ядро.
   - Поскольку бэкенд EasyCLIProxyAPI написан на Rust, модули из `ai-proxy` (тоже на Rust!) переносятся сюда с практически нулевыми затратами на адаптацию интерфейсов.
3. **Конфигурация вышестоящего прокси (`requests.proxy-url`)**:
   - Ядро EasyCLIProxyAPI имеет штатную настройку `proxy-url` (в `config.yaml` и через `patch_core_proxy_url`), через которую все исходящие запросы к OpenAI, Anthropic, Gemini могут маршрутизироваться через внешний шлюз.

---

## 2. Вариант 1: Реализация плагина маскирования для ядра (C-ABI / Go)

### 2.1. Структура плагина `plugins/masking-redactor`
На основе кода существующих плагинов `semantic-router` и `weekly-reset-priority` создается новый плагин:

```
plugins/masking-redactor/
├── manifest.json
├── main.go
├── bridge.go
├── redactor.go
├── entropy.go
├── patterns.go
├── build.sh
└── go.mod
```

### 2.2. Манифест плагина (`manifest.json`)
```json
{
  "id": "masking-redactor",
  "name": "Secret Masking & DLP Redactor",
  "version": "0.1.0",
  "license": "MIT",
  "abi_version": 1,
  "schema_version": 6,
  "minimum_kernel": "8.0.48",
  "install_type": "local-native-library",
  "entrypoints": {
    "linux": "masking-redactor.so",
    "darwin": "masking-redactor.dylib",
    "windows": "masking-redactor.dll"
  },
  "capabilities": ["request_filter", "model_router"],
  "default_enabled": true
}
```

### 2.3. Механизм перехвата в Go
В `bridge.go` экспортируется стандартная функция `cliproxy_plugin_init`:
```go
//export cliproxy_plugin_init
func cliproxy_plugin_init(host *C.host_api, plugin *C.plugin_api) C.int {
    if host == nil || plugin == nil || host.abi_version != 1 {
        return 1
    }
    C.initialize(host, plugin)
    return 0
}
```

В обработчике `routeCall` перехватывается полезная нагрузка запроса модели:
1. Десериализуется тело запроса промпта (`messages` / `prompt`).
2. Запускаются регулярные выражения из `ai-proxy/src/middleware/regex_scanner.rs`:
   - AWS Keys: `AKIA[0-9A-Z]{16}`
   - GitHub Tokens: `gh[pousr]_[A-Za-z0-9_]{36,}`
   - Anthropic/OpenAI Keys: `sk-ant-[A-Za-z0-9-]{20,}`, `sk-[A-Za-z0-9]{20,}`
   - Private Keys: `-----BEGIN (RSA|EC|DSA|OPENSSH)? PRIVATE KEY-----`
   - Generic Secrets: `(?i)(password|secret|token|api_key|apikey)\s*[=:]\s*["'][^"']{8,}["']`
3. Запускается алгоритм расчета энтропии Шеннона (порт `ai-proxy/src/middleware/entropy_scanner.rs`):
   ```go
   func shannonEntropy(s string) float64 {
       if len(s) == 0 { return 0 }
       freq := make(map[rune]float64)
       for _, r := range s { freq[r]++ }
       var ent float64
       length := float64(len([]rune(s)))
       for _, count := range freq {
           p := count / length
           ent -= p * math.Log2(p)
       }
       return ent
   }
   ```
4. Если энтропия токена выше 4.5 и в пределах 50 символов присутствуют слова `secret`, `key`, `token`, `password` — строка маскируется на `***...***` или заменяется плейсхолдером `[REDACTED_SECRET_#]`.

---

## 3. Вариант 2: Прямое встраивание Rust-конвейера в `reverse_proxy.rs` (Наиболее эффективный путь)

Так как бэкенд `EasyCLIProxyAPI` (`src-tauri`) написан на Rust и уже содержит модуль `reverse_proxy.rs` на базе `hyper 1.0` и `tokio_rustls`:

### 3.1. Перенос Rust-модулей из `ai-proxy`
В `src-tauri/src/` можно напрямую перенести файлы из `ai-proxy/src/`:
* `redactor.rs`
* `redaction_context.rs`
* `middleware/regex_scanner.rs`
* `middleware/entropy_scanner.rs`
* `middleware/structural_scanner.rs`

### 3.2. Встраивание в цикл проксирования Hyper
В `reverse_proxy.rs` входящий запрос расшифровывается через `TlsAcceptor` перед отправкой в ядро:
```rust
// Внутри обработчика соединений reverse_proxy:
let (parts, body) = req.into_parts();
let body_bytes = http_body_util::BodyExt::collect(body).await?.to_bytes();

// 1. Применяем конвейер сканирования и маскирования
let mut context = RedactionContext::new(&request_id, &redaction_config);
let findings = scan_pipeline.scan_bytes(&body_bytes);
let sanitized_bytes = redactor.redact_bytes(&body_bytes, &findings, &mut context);

// 2. Пересылаем санитизированный запрос в локальное ядро (127.0.0.1:8317)
let upstream_req = Request::from_parts(parts, Full::new(sanitized_bytes));
let upstream_res = client.request(upstream_req).await?;

// 3. Если ответ потоковый (SSE) и включено восстановление:
let response_stream = StreamingRestore::new(upstream_res.into_body(), context);
```

### 3.3. Почему это архитектурно идеально:
* **Общий язык (Rust)**: Полная типобезопасность, переиспользование структур без CGO и FFI оверхеда.
* **Потоковое восстановление (StreamingRestore)**: Реализация скользящего окна из `ai-proxy/src/redaction_context.rs` позволяет возвращать в GUI десктопа или агенту неиспорченный текст, сохраняя конфиденциальность при отправке наружу.

---

## 4. Вариант 3: Каскадирование через `proxy-url` (Zero-Code Integration)

Самый быстрый способ получить 100% возможностей обоих проектов без единой правки кода:

1. Запускается `ai-proxy` на порту `127.0.0.1:8080`.
2. В интерфейсе `EasyCLIProxyAPI` в настройках сети (`Network Settings`) или в конфигурации ядра `config.yaml` указывается:
   ```yaml
   requests:
     proxy-url: "http://127.0.0.1:8080"
   ```
3. **Результат каскада**:
   - Все 16+ агентов управляются через красивый GUI `EasyCLIProxyAPI`.
   - Сессии, ротация десятков аккаунтов, обход лимитов и веса моделей обрабатываются ядром `cli-proxy-api`.
   - В момент отправки запроса в реальный интернет запрос проходит через `ai-proxy`, где из него вырезаются все секреты, ключи и приватные данные, а ответы безопасно демаскируются на лету.

---

## 5. Дополнительные полезные механики из `ai-proxy`, рекомендуемые для EasyCLIProxyAPI

1. **Глубокая модернизация модуля `Sensitive Words`**:
   - Сейчас в `EasyCLIProxyAPI` (`core_config/yaml.rs:352`) есть только статический список слов: `antigravity_sensitive_words`, `devin_sensitive_words`.
   - **Улучшение**: Заменить статический список на пресеты из `ai-proxy` (JWT, Connection Strings, приватные ключи PEM, энтропийный детектор случайных паролей).
2. **Аудит вызовов инструментов (Tool Events Tracking)**:
   - В `ai-proxy/src/telemetry.rs` реализован изящный парсер JSON `extract_tool_events_from_json`, который детектирует вызовы `tool_use`, `function_call`, имя инструмента и статус выполнения.
   - В `EasyCLIProxyAPI` сейчас учитываются только токены и цены (`agent_usage.rs`). Добавление трекинга вызовов инструментов обогатит экран `UsageEventsView.tsx` детальной аналитикой использования MCP-инструментов субагентами.
3. **Безопасная инспекция локальных баз SQLite Codex (`codex_sessions.rs`)**:
   - В `EasyCLIProxyAPI` есть `CodexSessionContextEditor`, позволяющий смотреть сообщения сессий.
   - Интеграция `Redactor` из `ai-proxy` в режим `privacy_mode.rs` позволит автоматически скрывать найденные в истории чата токены и ключи прямо при рендере в React-интерфейсе при включенном режиме демонстрации экрана.

---

## 6. Итоговые выводы

Внедрение механизмов `ai-proxy` в `EasyCLIProxyAPI` закрывает главную слабую сторону последнего — отсутствие контроля за утечками чувствительных данных (DLP). Наиболее перспективными путями реализации являются:
* **Краткосрочно**: Использование архитектуры каскадного прокси через штатную опцию `proxy-url`.
* **Долгосрочно**: Встраивание готовых Rust-модулей сканеров из `ai-proxy` прямо в защитный контур `reverse_proxy.rs` или выпуск официального Go-плагина через встроенную C-ABI систему расширений.
