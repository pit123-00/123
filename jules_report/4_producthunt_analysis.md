# Отчет: Анализ ProductHunt (ProductHunt.com-Scrapers)

Источник данных:
* ProductHunt.com-Scrapers: https://github.com/scraper-bank/ProductHunt.com-Scrapers

## 1. Паттерны классического парсинга DOM-узлов (BeautifulSoup)

В скрипте `python/BeautifulSoup/product_data/scraper/producthunt.com_scraper_product_v1.py` используется библиотека `BeautifulSoup` (`bs4`) для извлечения данных из сырого HTML страницы.

Вместо сложного взаимодействия с GraphQL или скрытыми API, скрипт применяет классический подход парсинга по CSS-селекторам и атрибутам:
*   **Извлечение из JSON-LD:** Изначально код ищет скрипты типа `application/ld+json`. Это распространенный стандарт SEO, который часто содержит уже готовую и чистую информацию (название, цена, бренд, описание, рейтинг).
*   **Поиск по селекторам:** Если в JSON-LD нужных данных нет, происходит fallback на CSS селекторы:
    *   `soup.select_one("a[href*='/categories/']")` — для категорий.
    *   `soup.find("meta", attrs={"name": "description"})` — для описания.
    *   `soup.select("div[data-test='launch'] section.flex-row.overflow-x-scroll img")` — поиск скриншотов по атрибутам `data-test` (очень часто используются в React-приложениях вроде ProductHunt для тестирования, поэтому они стабильны).
    *   `soup.select_one("div[data-test='comments-feed']")` — сбор отзывов.
*   **Очистка данных:** Используются методы `.get_text(strip=True)`, чтобы убрать лишние переносы и пробелы из HTML-сущностей, а также удаляются ненужные узлы (например, `.decompose()` для скриптов или стилей внутри текста).

## 2. Маршрутизация запросов через прокси-шлюз (Proxy APIs)

Для обхода защиты от ботов (Cloudflare, Rate Limiting), используется сторонний сервис (в данном случае `ScrapeOps Proxy API`).

*   Вместо классических HTTP-прокси (`http://user:pass@ip:port`), запрос отправляется через REST API шлюза:
    ```python
    payload = {
        "api_key": API_KEY,
        "url": url,
        "optimize_request": True,
    }
    proxy_url = 'https://proxy.scrapeops.io/v1/?' + urlencode(payload)
    response = requests.get(proxy_url, timeout=30)
    ```
*   Такой подход перекладывает ответственность за обход капчи, подмену User-Agent, ротацию IP и рендеринг JS (при необходимости) на сервис-прокси.

## 3. Использование `ThreadPoolExecutor` для многопоточности

Для ускорения парсинга нескольких URL-адресов реализована многопоточность через стандартный модуль `concurrent.futures`.

*   **Функция `concurrent_scraping`:**
    ```python
    from concurrent.futures import ThreadPoolExecutor

    def concurrent_scraping(urls: List[str], max_threads: int = 1, max_retries: int = 3):
        pipeline = DataPipeline(...)
        with ThreadPoolExecutor(max_workers=max_threads) as executor:
            futures = [executor.submit(scrape_page, url, pipeline, max_retries) for url in urls]
            for future in futures:
                future.result()
    ```
*   `ThreadPoolExecutor` создает пул потоков. Каждый URL передается в функцию `scrape_page`, где происходят HTTP-запросы (через шлюз) и парсинг, тем самым нивелируя задержки I/O (ожидание ответа сети).
*   В самом `scrape_page` реализован простейший механизм **Retries** (цикл `while tries <= retries`).

## Итог для Cherry-picking
1.  **Парсинг:** Использовать связку CSS-селекторов (`soup.select()`) с таргетингом на атрибуты `data-test` и парсинг `application/ld+json`, если он присутствует на странице.
2.  **Обход блокировок:** Если простой `requests` + `User-Agent` блокируется, использовать концепцию Proxy-шлюзов.
3.  **Асинхронность/Многопоточность:** Обернуть вызовы к скраперу в `ThreadPoolExecutor` или использовать `asyncio` (`httpx`) для ускорения сбора данных с сотен страниц.
