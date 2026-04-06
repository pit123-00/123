# Отчет о технической реализации парсеров ProductHunt и их адаптации для PostgreSQL + Python

В данном отчете представлен подробный технический и архитектурный анализ репозитория `ProductHunt.com-Scrapers`, а также схема адаптации описанных подходов (проксирование запросов, сбор в `dataclass`, потоковое сохранение) для использования в архитектуре вашего проекта на базе PostgreSQL и Python.

## 1. Техническая реализация и архитектура

Репозиторий `ProductHunt.com-Scrapers` не является монолитным приложением. Это коллекция легковесных, независимых скриптов, разделенных по языкам (Node.js, Python) и используемым технологиям скрапинга (BeautifulSoup, Playwright, Selenium).

### Формирование запроса и обход защиты
Вместо прямых HTTP-запросов к ProductHunt, скрипты используют **прокси-API ScrapeOps**.
Целевой URL кодируется и передается в качестве параметра к эндпоинту `https://proxy.scrapeops.io/v1/` вместе с API-ключом (`api_key`) и флагами оптимизации (`optimize_request=True`). Эта архитектура позволяет делегировать задачи ротации IP-адресов, обхода Cloudflare и управления заголовками на сторону стороннего провайдера, возвращая чистое тело ответа (HTML).

### Структуры данных и сохранение
Для типизации и структурирования собираемых данных в Python используются **Dataclasses** (модуль `dataclasses`).
Ключевой объект, например `ScrapedData`, содержит типизированные поля для метаданных поиска, пагинации, хлебных крошек и списка продуктов.
Для хранения используется формат **JSON Lines (`.jsonl`)**. Данные из `dataclass` сериализуются в словарь (`asdict`), затем в строку JSON и дописываются (`mode="a"`) в конец файла. Построчная запись в `.jsonl` позволяет обрабатывать огромные объемы информации без необходимости загрузки всего массива данных в оперативную память.

---

## 2. Функции и логическая работа

Процесс сбора данных разделен на несколько логических этапов и изолированных функций:

1. **`concurrent_scraping(urls, max_threads, max_retries)`**
   Является точкой входа. Использует `ThreadPoolExecutor` из `concurrent.futures` для организации пула потоков. Позволяет параллельно обрабатывать массив стартовых URL-адресов, распределяя задачи.

2. **`scrape_page(url, pipeline, retries)`**
   Выполняет сетевое взаимодействие с механизмом *Retry* (по умолчанию 3 попытки). В цикле `while` скрипт пытается получить ответ `HTTP 200`. В случае успеха ответ передается парсеру (`extract_data`), а полученный результат отправляется в конвейер сохранения (`pipeline.add_data`).

3. **`extract_data(soup)`**
   Ядро бизнес-логики скрапера. Получает объект `BeautifulSoup` и извлекает информацию, используя гибридный подход:
   - Парсинг микроразметки **JSON-LD** (`<script type="application/ld+json">`).
   - Использование **CSS-селекторов** (`soup.select()`, `soup.find()`) для извлечения данных из DOM-дерева (кнопки пагинации, карточки продуктов, отзывы).
   Возвращает заполненный экземпляр датакласса (например, `ScrapedData`).

4. **Класс `DataPipeline` (Дедупликация и запись)**
   Отвечает за потоковую запись и обеспечение уникальности записей.
   Использует метод `is_duplicate()`, который вычисляет хеш от сериализованных данных (`hash(json.dumps(..., sort_keys=True))`) и проверяет его наличие во внутреннем множестве (Set).

---

## 3. Обмен информацией, память и потоки данных

* **Поток данных (Data Flow):**
  `Стартовый URL -> Запрос к Proxy API (ScrapeOps) -> Получение HTML -> Парсинг (BeautifulSoup) -> Создание объекта ScrapedData -> Pipeline (Дедупликация) -> Сериализация в JSON -> Запись на диск (.jsonl)`.
* **Память в рантайме (In-memory Storage):**
  Хранение состояния для дедупликации осуществляется в памяти в рамках одной сессии работы скрипта. Класс `DataPipeline` использует `self.items_seen = set()` для хранения хешей уже сохраненных элементов.
* **Отсутствие долгосрочной памяти:** Скрипты являются "stateless" между запусками. Каждый новый запуск создает новый файл `.jsonl` (с таймстемпом) и очищает историю дедупликации.

---

## 4. Схема адаптации для PostgreSQL и Python

Чтобы перенести эту логику в вашу гибридную архитектуру (PostgreSQL, графы, асинхронные фоновые задачи), предлагается следующая схема:

### 1. Переход от In-memory дедупликации к Базе Данных
Вместо хранения хешей в `items_seen` (что сбрасывается при рестарте), уникальность должна контролироваться на уровне PostgreSQL с использованием ограничений `UNIQUE` и операций "Upsert" (`ON CONFLICT DO UPDATE`).

### 2. Структура Базы Данных (Гибридная реляционно-документная модель)
Вместо `.jsonl` файлов, данные будут сохраняться в таблицы PostgreSQL. Учитывая изменчивую структуру ProductHunt (разные теги, отзывы, медиа), идеально использовать комбинацию жестко заданных колонок (для поиска/графов) и поля `JSONB` для гибких метаданных.

**Пример SQL схемы:**

```sql
CREATE TABLE producthunt_products (
    id SERIAL PRIMARY KEY,
    product_id VARCHAR(255) UNIQUE NOT NULL, -- Внешний ID ProductHunt
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    description TEXT,
    category VARCHAR(100),
    scraped_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    -- JSONB поле для хранения полной структуры из dataclass
    -- (images, specifications, badges, aggregateRating, variants и т.д.)
    raw_metadata JSONB NOT NULL
);

-- Индексы для ускорения работы с JSONB и поиска
CREATE INDEX idx_producthunt_metadata ON producthunt_products USING GIN (raw_metadata);
CREATE INDEX idx_producthunt_category ON producthunt_products(category);
```

### 3. Адаптация Python кода (Пайплайн данных)
* **Dataclasses:** Оставить логику сбора в `dataclass` для удобной валидации внутри парсера.
* **ORM / Драйвер:** Использовать `asyncpg` или SQLAlchemy 2.0 (асинхронный движок).
* **Многопоточность -> Асинхронность:** Заменить `ThreadPoolExecutor` (блокирующий I/O) на `asyncio` (например, `aiohttp` или асинхронный Playwright), что позволит эффективнее управлять тысячами соединений. Либо, как описано в вашей архитектуре, переложить запуск парсеров на фоновые воркеры (наподобие Airflow/Celery).
* **Новый DataPipeline:**
```python
# Пример логики сохранения в PostgreSQL
async def save_to_postgres(db_pool, scraped_data: ScrapedData):
    async with db_pool.acquire() as conn:
        await conn.execute("""
            INSERT INTO producthunt_products (product_id, name, url, raw_metadata)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (product_id) DO UPDATE SET
                name = EXCLUDED.name,
                url = EXCLUDED.url,
                raw_metadata = EXCLUDED.raw_metadata,
                scraped_at = CURRENT_TIMESTAMP
        """,
        scraped_data.productId,
        scraped_data.name,
        scraped_data.url,
        json.dumps(asdict(scraped_data)) # Сохранение всей структуры в JSONB
        )
```

Такой подход сохраняет преимущества гибкости `JSONL/Dataclasses`, но добавляет транзакционность, надежную дедупликацию и мощные возможности поиска по JSONB внутри PostgreSQL.