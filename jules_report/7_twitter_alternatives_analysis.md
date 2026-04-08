# Отчет: Анализ альтернативных библиотек Twitter (XRSS, TweeterPy)

Источники данных:
* XRSS: https://github.com/Thytu/XRSS
* TweeterPy: https://github.com/iSarabjitDhiman/TweeterPy
*(twitter-api-client недоступен для клонирования)*

## 1. XRSS (Генерация RSS и обработка Cookies)

XRSS — это обертка вокруг `twikit`, которая специализируется на конвертации Twitter GraphQL ответов в чистый RSS поток.

### Работа с авторизацией (Cookies & ct0):
*   XRSS хранит куки в JSON файле (по умолчанию `cookies.json`).
*   **Решение проблем с валидацией:** В файле `xrss/utils.py` есть важная функция `clean_cookies`, которая предотвращает ошибки множественных сессий. Если в файле скапливается несколько токенов `ct0` (CSRF токен), библиотека удаляет старые, так как дублирование `ct0` ведет к возврату статуса `403 Forbidden` от серверов X.
*   Если авторизация устарела, скрипт удаляет файл `cookies.json` и логинится заново через `twikit_client.login(...)` с использованием учетных данных (и опционально TOTP `mfa_secret`).

### Генерация RSS и санитаризация XSS:
*   Сам XRSS использует `feedgen.feed` (модуль `FeedGenerator`).
*   Особое внимание уделяется парсингу `full_text` и подстановке HTML. Если твит содержит медиа или ссылки, XRSS не доверяет сырому тексту от Twitter, а восстанавливает его (через функцию `clean_tweet`), заменяя сокращенные `t.co` ссылки на оригинальные (параметр `expanded_url`). Это отличная практика для предотвращения инъекций (XSS).

## 2. TweeterPy (Сессии на базе curl_cffi и Пагинация)

### Имитация TLS / JA3 Fingerprint:
*   TweeterPy использует `curl_cffi` (вместо стандартного `requests` или `httpx`), как видно в `tweeterpy/utils/session.py` (`from curl_cffi.requests.session import Session`).
*   Это позволяет эмулировать "отпечаток" TLS и HTTP/2 (JA3 fingerprint) реального браузера (например, Chrome 110), что значительно снижает вероятность блокировки (Rate Limits, 403 Forbidden).

### Механика пагинации:
В TweeterPy отлично реализована универсальная обработка курсоров для любых конечных точек GraphQL (в `tweeterpy/tweeterpy.py`).
*   Вместо написания уникального кода для каждого метода, используется декоратор и метод `_handle_pagination`, принимающий `data_path` — путь (кортеж ключей) к инструкциям в JSON, где нужно искать курсор.
*   Пример:
    ```python
    data_path = ('data', 'user', 'result', 'timeline', 'timeline', 'instructions')
    return self._handle_pagination(..., data_path=data_path)
    ```
*   Это позволяет быстро адаптироваться к изменениям структуры GraphQL: если X меняет вложенность, достаточно поменять `data_path`, а не переписывать логику поиска `cursorType: Bottom`.

## Итог для Cherry-picking
1.  **Обход защиты (TweeterPy):** Вместо `requests` использовать `curl_cffi.requests.Session(impersonate="chrome")` для обхода TLS fingerprinting защиты X/Twitter.
2.  **Очистка кукисов (XRSS):** Перед отправкой запросов проверять массив cookies. Оставлять только *один* (самый свежий) `ct0` токен.
3.  **Санитаризация текста:** При сборке текста твита использовать GraphQL entities (`expanded_url`), чтобы раскрывать `t.co` ссылки и избегать вставки "грязного" HTML/XSS.
4.  **Универсальная пагинация:** Хранить пути до инструкций с курсорами (например, `['data', 'search_by_raw_query', 'search_timeline', 'timeline', 'instructions']`) в константах и использовать универсальный парсер курсора по этому пути.
