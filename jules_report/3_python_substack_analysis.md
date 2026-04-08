# Отчет: Анализ Substack (python-substack, substack-api)

Источник данных:
* python-substack: https://github.com/ma2za/python-substack
* substack-api: https://github.com/NHagar/substack_api

## 1. Взаимодействие с непубличным внутренним API Substack и работа с сессиями

В обеих библиотеках используется прямое обращение к внутренним эндпоинтам `https://<путь_издания>/api/v1/...` (например, `/api/v1/reader/posts` для ленты). Substack не предоставляет открытого REST API для записи/публикации, поэтому вся интеграция построена на эмуляции запросов из браузера.

### Механизм логина и получения cookies (`substack.sid`)

*   **Cookie `substack.sid`**: Главный ключ аутентификации. Библиотеки поддерживают вход через E-mail/Пароль, но из-за усиления защиты (CAPTCHA, "Magic Links", где пароль больше не требуется) лучшей практикой является экспорт cookies из реального браузера.
*   В `python-substack/substack/api.py` сессия создается с помощью `requests.Session()`.
*   Пользователь может загрузить куки из JSON файла или напрямую как строку (скопировав заголовок `Cookie` из DevTools: `cookie1=value1; cookie2=value2`).
*   **Итог:** Самый стабильный способ обхода базовой защиты Substack — это использовать Selenium/Playwright для первоначального логина (решение капчи/чтение magic link), после чего сохранять куки, вытаскивать `substack.sid` и передавать его в `requests.Session`.

### Эндпоинты API

*   **Получение ленты**:
    ```python
    response = self._session.get(f"{self.base_url}/reader/posts") # base_url: https://substack.com/api/v1
    ```
*   **Количество подписчиков**:
    Эндпоинт `/api/v1/publication_launch_checklist`. Парсится из `response["subscriberCount"]`.
    ```python
    response = self._session.get(f"{self.publication_url}/publication_launch_checklist")
    ```
*   **Управление черновиками (Drafts)**:
    Запросы отправляются на `/drafts`, `/drafts/{draft_id}/publish` или `/drafts/{draft_id}/schedule`.
*   **Загрузка картинок**:
    Картинки отправляются на `/image` как Base64 строка (`data:image/jpeg;base64,...`).

## 2. Абстракция Post (Сборка метаданных)

В Substack API посты представлены в виде блочной архитектуры (аналог Notion или блоков Gutenberg), основанной на спецификации ProseMirror.

В файле `python-substack/substack/post.py` реализована абстракция `Post`, которая:
*   Собирает метаданные: `title`, `subtitle`, `audience` (`everyone`, `only_paid`), `write_comment_permissions`.
*   Формирует `draft_body` в виде JSON объекта с полем `"type": "doc"` и массивом `"content"`.
*   Каждый абзац или элемент переводится в словарь, например:
    ```json
    {"type": "paragraph", "content": [{"type": "text", "text": "Текст"}]}
    {"type": "captionedImage", "src": "https://..."}
    {"type": "paywall"}
    ```
*   Для парсинга Markdown в этот JSON-формат реализована логика (`from_markdown`), которая разбирает блоки (заголовки, цитаты, списки) и конвертирует их в древовидную структуру узлов.

## Итог для Cherry-picking

1.  **Аутентификация:** Создать класс сессии на основе `requests.Session()`, который будет загружать `substack.sid` из БД (или файла cookies.json).
2.  **Эндпоинты:**
    *   Лента читателя: `GET https://substack.com/api/v1/reader/posts`
    *   Публикация черновика: `POST https://<domain>.substack.com/api/v1/drafts`
3.  **Конвертация контента:** Использовать блочную JSON структуру (ProseMirror format) для отправки форматированного текста, заголовков и картинок. При публикации подготавливать JSON, где `body` имеет структуру `"type": "doc", "content": [...]`.
