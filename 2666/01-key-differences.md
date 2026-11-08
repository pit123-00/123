# Отчёт 1: Ключевые отличия между Dokku и Dokploy на основе исходного кода

Настоящий отчёт подготовлен на основе прямого и глубокого анализа кодовых баз проектов:
* **Dokku** (`https://github.com/dokku/dokku/`) — анализ Go-исходников плагинов, bash-скриптов оркестрации, утилиты plugn, системных триггеров и шаблонов.
* **Dokploy** (`https://github.com/dokploy/dokploy/`) — анализ TypeScript/Node.js монорепозитория (`pnpm workspace`), API-сервисов на Hono/tRPC, моделей Drizzle ORM, очередей Inngest, интеграций Dockerode / Docker Swarm и конфигураций Traefik.

---

## 1. Концептуальная парадигма и архитектурная философия

| Параметр сравнения | Dokku (по коду) | Dokploy (по коду) |
| :--- | :--- | :--- |
| **Базовая парадигма** | **CLI & Git-centric Unix-way PaaS**. Dokku спроектирован как расширяемый командный процессор на базе Bash и скомпилированных бинарных файлов на Go. Управление завязано на пользователя системы `dokku`, хуки `git-receive` и плагины. | **API/Web & Fleet-centric Modern Cloud PaaS**. Dokploy спроектирован как централизованная управляющая платформа с веб-дашбордом, строгой типизацией на TypeScript, клиент-серверной архитектурой (Client -> tRPC/Hono -> Queue -> Worker -> Target Server). |
| **Точка входа (Entrypoint)** | Файл `/dokku` (Bash) выполняет валидацию окружения, разбор флагов, смену пользователя через `sudo -u dokku` и делегирование в бинарные файлы плагинов (`$PLUGIN_ENABLED_PATH/$PLUGIN_NAME/subcommands/...`). | `apps/dokploy/server/index.ts` и `apps/api/src/index.ts` — HTTP-серверы на Node.js (Next.js server + Hono REST API + tRPC API), слушающие системные порты и управляющие процессами асинхронно. |
| **Единица изоляции и владения** | Приложение (app) в файловой системе хоста (`$DOKKU_ROOT/<app>`). Пользователи привязаны к SSH-ключам через утилиту `sshcommand`. Многопользовательский веб-доступ «из коробки» в ядре отсутствует. | Мультитенантная иерархия в БД: Организация (`organization`) -> Проект (`project`) -> Окружение (`environment`) -> Приложения / Базы данных / Compose-стеки. Полноценная ролевая модель RBAC и Enterprise SSO. |

---

## 2. Технологический стек и управление зависимостями

### Dokku:
1. **Языки реализации:**
   * **Go (Golang):** Используется для всех основных плагинов (`common`, `config`, `ps`, `builder`, `network`, `logs`, `scheduler-k3s`, `storage`). Бинарники компилируются через `go build -mod=readonly -ldflags="-s -w"` в многоцелевые исполняемые файлы (`commands`, `subcommands`, `triggers`).
   * **Bash (POSIX-совместимый bash с `pipefail` и `set -eo`):** Реализует высокоуровневый дирижёр `dokku`, функции `functions`, триггерные скрипты и клей между подсистемами.
2. **Модульность ядра:**
   * Механизм плагинов реализован через утилиту `plugn` (`github.com/dokku/dokku/plugins/common/plugn.go`).
   * Плагины объявляются через `plugin.toml`, лежат в `/var/lib/dokku/core-plugins` и `/var/lib/dokku/plugins`, связываясь симлинками (`enabled/available`).
3. **Зависимости времени выполнения:** Docker CLI, `plugn`, `procfile-util`, `sshcommand`, `parallel` (GNU parallel для параллельного деплоя контейнеров), `gliderlabs/herokuish`.

### Dokploy:
1. **Языки реализации:**
   * **TypeScript / Node.js (v20+ / v24):** 100% строгая типизация, монорепозиторий на базе `pnpm workspaces` (`apps/dokploy`, `apps/api`, `packages/server`).
2. **Бэкенд-фреймворки и протоколы:**
   * **Hono & Node Server:** Легковесный производительный HTTP-движок для микросервиса очередей (`apps/api`).
   * **tRPC v10/v11:** Сквозная типизация вызовов между дашбордом и бэкендом (`packages/server/src/services/**` -> `apps/dokploy/server/api/routers/**`).
   * **Drizzle ORM:** Type-safe SQL-драйвер для PostgreSQL с автоматическими миграциями (`drizzle/`).
   * **Inngest:** Распределенный движок очередей и функций (`deploy-application`) с поддержкой паузы, отмены деплоя (`deployment/cancelled`) и ограничения параллелизма по серверу.
3. **Зависимости времени выполнения:** Node.js runtime, PostgreSQL (хранилище метаданных Dokploy), Traefik (встроенный обратный прокси), Docker Engine / Docker Swarm (через `dockerode` и удалённые SSH-сессии), rclone (для резервного копирования в S3).

---

## 3. Модель хранения конфигурации и состояния (State Management)

### Dokku (Файловая система хоста):
* Все настройки приложений хранятся в плоских файлах на диске в директории `$DOKKU_ROOT/<app>/` (обычно `/home/dokku/<app>/` или `/var/lib/dokku/data/<plugin>/<app>`):
  * `ENV` — переменные окружения, экранированные и заблокированные от параллельной записи через `common.LockDirectory` (`plugins/config/config.go`).
  * `PORT`, `CONTAINER_PORT`, `VHOST`, `SCHEDULER`, `BUILDER` — отдельные текстовые маркеры и свойства.
  * `PropertyGet` / `PropertyWrite` (`plugins/common/properties.go`) сохраняет значения в файловую структуру `data/<plugin>/<app>/<property>`.
* **Плюсы и свойства:** Полный GitOps на файловой системе, нулевая зависимость от внешней базы данных, состояние хоста можно резервировать обычным копированием директорий.

### Dokploy (Реляционная СУБД PostgreSQL):
* Все метаданные, состояние развертываний, пользователи, ключи API, токены провайдеров и настройки хранятся в реляционной базе данных (`packages/server/src/db/schema/`):
  * Таблицы: `organization`, `user`, `project`, `environment`, `application`, `compose`, `postgres`, `mysql`, `redis`, `mariadb`, `mongo`, `libsql`, `deployment`, `server`, `sshKey`, `destination`, `domain`, `certificate`, `notification`, `schedule`, `volumeBackups`, `ai`.
  * Переменные окружения хранятся в столбце `env` таблицы `application` или `compose` и могут наследоваться от уровня `project` и `environment` (`prepareEnvironmentVariablesForShell`).
  * Секреты (токены провайдеров, пароли) шифруются с использованием `AES-256` (`packages/server/src/lib/encryption.ts`).
* **Плюсы и свойства:** Богатые связи (foreign keys, cascading deletes), аудит, гибкие запросы, независимость метаданных от состояния конкретного хоста с Docker.

---

## 4. Сборка артефактов (Builders)

В коде обоих проектов заложены богатые возможности сборки, но их реализация принципиально отличается:

| Builder | Реализация в Dokku | Реализация в Dokploy |
| :--- | :--- | :--- |
| **Dockerfile** | Плагин `plugins/builder-dockerfile`: вызывает CLI команду `docker image build` с генерацией меток OCI, поддержкой `--build-arg`, секретов, ssh-агентов сборки и кэширования BuildKit. | Файл `packages/server/src/utils/builders/docker-file.ts`: генерирует команду `docker build` через shell-скрипт с динамической передачей build-args и путей к контексту. |
| **Nixpacks** | Плагин `plugins/builder-nixpacks`: бинарник `nixpacks build` с автоматической генерацией промежуточного `builder-build.Dockerfile` для инжекции dokku-меток и runtime-entrypoint. | Файл `packages/server/src/utils/builders/nixpacks.ts`: генерирует bash-пайплайн с вызовом `nixpacks build`, поддержкой статических сайтов (`publishDirectory`) и извлечением артефактов через `docker cp`. |
| **Cloud Native Buildpacks (Pack)** | Плагин `plugins/builder-pack`: нативная интеграция с `pack build` (Paketo, Heroku builders). | Модуль `paketo.ts`: сборка через `pack build` с билдером `paketobuildpacks/builder:base`. |
| **Herokuish (Классический Heroku)** | Плагин `plugins/builder-herokuish`: использование контейнера `gliderlabs/herokuish` для эмуляции рантайма Heroku и запуска `compile` скриптов buildpack. | Модуль `heroku.ts`: вызов `pack build` с билдером `heroku/buildpacks:20`. |
| **Railpack** | Плагин `plugins/builder-railpack`: интеграция движка сборки от Railway. | Модуль `railpack.ts`: прямая интеграция движка Railway `railpack`. |
| **Static Sites** | Отсутствует как отдельный тип (решается через `nginx-buildpack` или Dockerfile с nginx). | Встроенный тип билдера `static` (`static.ts`): генерация временного контейнера, копирование артефактов в директорию хоста и запуск Nginx-контейнера для раздачи статики. |

---

## 5. Оркестрация и развертывание (Schedulers)

### Dokku:
* Модульная архитектура планировщиков через триггеры `scheduler-deploy`, `scheduler-stop`, `scheduler-enter` и т.д.:
  1. **`scheduler-docker-local`:** Запуск обычных контейнеров Docker на локальном хосте. Включает сложнейший механизм Zero-Downtime деплоя (`check-deploy`), запуск проверок готовности через `docker-container-healthchecker`, параллельный запуск сервисов через GNU `parallel`, и поэтапное мягкое гашение старых контейнеров с отправкой `SIGTERM` и задержкой `wait-to-retire` секунд.
  2. **`scheduler-k3s`:** Полноценный встроенный планировщик в легковесный Kubernetes (k3s). Dokku компилирует конфигурацию приложения в Helm-чарты (`plugins/scheduler-k3s/templates/chart/`), деплоит Ingress, Service, Deployment, KEDA (для автомасштабирования до нуля и обратно), Longhorn (хранилище) и Traefik/Ingress-Nginx.

### Dokploy:
* Поддерживает две основные модели выполнения:
  1. **Standalone Docker Engine:** Управление контейнерами напрямую через Docker API (`dockerode`) или генерация/запуск Docker Compose проектов на целевом сервере.
  2. **Docker Swarm Mode (`cluster.ts` / `swarm.ts`):** Dokploy нативно развертывает приложения как сервисы Swarm (`docker service create / update`), что даёт репликацию, rolling updates, health checks на уровне swarm-демона и размещение по нодам (`PlacementSwarm`, `EndpointSpecSwarm`).
* **Мультисерверность (Remote Servers):** Dokploy из коробки умеет управлять удаленными серверами по SSH (`packages/server/src/services/server.ts`). Команды сборки и деплоя выполняются через `execAsyncRemote` с аутентификацией по SSH-ключам.

---

## 6. Сетевой уровень, маршрутизация и SSL

### Dokku:
* Dokku поддерживает сменные бэкенды обратного прокси через плагины vhosts:
  * `nginx-vhosts` (дефолтный, генерирует конфигурации `/etc/nginx/conf.d/...` через движок шаблонов Sigil).
  * `caddy-vhosts`, `traefik-vhosts`, `haproxy-vhosts`, `openresty-vhosts`.
* SSL-сертификаты управляются через плагин `certs` или плагин `letsencrypt` (через certbot acme challenge).

### Dokploy:
* Глубоко завязан на **Traefik** как единый шлюз маршрутизации:
  * Конфигурация генерируется динамически через файлы конфигурации Traefik (`/etc/dokploy/traefik/dynamic/*.yaml`) или через Docker Labels (`dokploy-labels.ts`).
  * Полная автоматизация Let's Encrypt через Traefik CertResolver (HTTP-01, TLS-ALPN-01 или DNS-01 с 30+ DNS-провайдерами: Cloudflare, Route53, DigitalOcean и др.).
  * Встроенные Middleware: Basic Auth, Forward Auth (SSO), компрессия, перенаправления (Redirects), пользовательские заголовки и защита от перегрузок.

---

## 7. Сводная таблица ключевых различий

| Критерий | Dokku | Dokploy |
| :--- | :--- | :--- |
| **Основной стек** | Go + Bash + Linux Coreutils | TypeScript + Node.js + Hono + Next.js |
| **Управление** | CLI через SSH (`ssh dokku@host <cmd>`) или локально в bash | Веб-интерфейс, tRPC, OpenAPI REST API, планируемый MCP |
| **Очередь задач** | Синхронное выполнение в CLI-сессии (блокирующий вызов) | Асинхронная распределенная очередь Inngest (с отменой задач) |
| **Базы данных** | Через внешние datastore-плагины (`dokku-postgres`, `dokku-redis` и т.д.) | Первоклассные встроенные сервисы (Postgres, MySQL, MariaDB, Mongo, Redis, LibSQL) с автобэкапами в S3 |
| **Масштабирование** | Локальные Docker-контейнеры либо K3s/Kubernetes чарты | Docker Swarm (кластер нод) и мультисерверный SSH-деплой |
| **AI/LLM-функции** | Отсутствуют | Встроенный модуль `services/ai.ts` (генерация Docker Compose и конфигов) |
| **Лицензия ядра** | MIT | Открытое ядро (Apache 2.0 / MIT) + коммерческие Enterprise-модули (SSO, Whitelabeling, Audit Log) |
