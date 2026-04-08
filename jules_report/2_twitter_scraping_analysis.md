# Отчет: Анализ Twitter Scraping (twscrape, twitter-rss)

Источник данных:
* twscrape: https://github.com/vladkens/twscrape
* twitter-rss: https://github.com/book000/twitter-rss

## 1. Управление пулом аккаунтов (Accounts Pool)
Реализация из `twscrape` — отличный пример работы с множеством сессий, обходом ограничений (rate limits) и ротацией аккаунтов.

### Архитектура (accounts_pool.py и account.py):
*   **Хранение данных:** Используется база данных SQLite (`accounts_pool.py`). Таблица хранит поля: username, password, email, user_agent, cookies, headers, stats (статистика запросов) и locks (блокировки по типам запросов).
*   **Состояние аккаунта (Account):** Объект `Account` хранит текущее состояние сессии. У каждого аккаунта есть свой жестко привязанный `user_agent` и сохраненные `cookies`.

### Работа с блокировками (Rate Limits) и очередями:
*   Разные эндпоинты GraphQL Twitter имеют разные лимиты. `twscrape` реализует концепцию "очередей" (queues), которые соответствуют типам запросов (например, `SearchTimeline`, `UserTweets`).
*   **Блокировка аккаунта (`lock_until`):** Когда возникает Rate Limit или ошибка, аккаунт блокируется для конкретной очереди:
    ```sql
    UPDATE accounts SET
        locks = json_set(locks, '$.{queue}', datetime({unlock_at}, 'unixepoch')),
        stats = json_set(stats, '$.{queue}', COALESCE(json_extract(stats, '$.{queue}'), 0) + {req_count})
    WHERE username = :username
    ```
*   **Получение свободного аккаунта (`get_for_queue`):** При новом запросе система берет аккаунт из БД, у которого блокировка для нужной очереди (`queue`) отсутствует либо уже истекла, и сразу ставит кратковременный lock (на 15 минут), чтобы другой поток его не взял.

## 2. Формирование запросов и сессий
Для обхода защиты используются правильные заголовки и токены (см. `account.py`).

*   **HTTP-клиент:** Используется `httpx.AsyncClient` для асинхронных запросов.
*   **Token:** Жестко зашит гостевой Bearer токен:
    `Bearer AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D1Zv7ttfk8LF81IUq16cHjhLTvJu4FA33AGWWjCpTnA` (используется повсеместно в веб-клиенте X).
*   **Заголовки:** Обязательно передаются `x-twitter-active-user: yes` и `x-twitter-client-language: en`.
*   **CSRF Токен (ct0):** Если в cookies аккаунта есть кука `ct0`, её значение обязательно копируется в заголовок `x-csrf-token`.
*   **Proxy:** Интегрирована поддержка прокси-серверов на уровне создания `httpx.AsyncClient`.

## 3. Обход пагинации (GraphQL Cursor)
Twitter использует пагинацию на основе курсоров для всех списков (твиты, подписчики, поиск).

### Механика (api.py):
*   **GraphQL запрос:** Делается POST/GET запрос к `https://x.com/i/api/graphql/{OP_NAME}`.
*   Передаются `variables` (параметры поиска/юзера) и `features` (флаги интерфейса).
*   **Извлечение курсора:**
    В возвращаемом JSON ответе парсится структура в поисках элемента `cursor`. Функция рекурсивно ищет объект с ключом `cursorType` (обычно нужен `"Bottom"` для загрузки старых твитов или `"Top"` для новых).
    ```python
    # Алгоритм из api.py
    def _get_cursor(self, obj: dict, cursor_type="Bottom") -> str | None:
        if cur := find_obj(obj, lambda x: x.get("cursorType") == cursor_type):
            return cur.get("value")
        return None
    ```
*   **Следующий запрос:** Найденное значение `value` (например, строка вида `cursor-bottom-1897829019747876852`) подставляется в `variables["cursor"]` при следующем запросе.
*   **Фильтрация курсоров:** Из списка "entries" (возвращаемых элементов) отфильтровываются сами узлы курсоров (`x["entryId"].startswith("cursor-")`), оставляя только полезные данные (твиты или пользователей).

## Итог для Cherry-picking
1.  Создать пул аккаунтов в БД SQLite с сохранением их кукисов и User-Agent.
2.  Написать менеджер, который будет выдавать аккаунты потокам и "замораживать" их на 15-20 минут (обновляя время в БД), если возвращается статус HTTP 429.
3.  Для GraphQL запросов использовать постоянный Bearer Token, переносить `ct0` из cookie в заголовок `x-csrf-token`.
4.  Для пагинации парсить JSON и извлекать `"cursorType": "Bottom"`, передавая его значение в следующий запрос.

## 4. Фильтрация рекламных твитов и санитаризация текста (избежание XSS)
* В запросах к GraphQL (как видно в `twscrape/api.py`), чтобы исключить рекламные твиты изначально, следует передавать параметр `includePromotedContent: False` внутри ключа `variables`. Если этого не сделать, Twitter будет возвращать объекты с полем `promotedMetadata`, которые необходимо отфильтровывать вручную при парсинге списка `entries`.
* Санитаризация XSS для RSS/HTML достигается за счет обработки и эскейпинга HTML сущностей и правильной обработки полей `full_text` и ссылок. Твиты в GraphQL возвращают entities (urls, hashtags), которые можно безопасно пересобрать в текст, не доверяя сырому HTML.
