# HTML Sanitization (repo: LetterFeed)

В репозитории `LetterFeed` в файле `backend/app/services/email_processor.py` реализован отличный и надежный механизм очистки и нормализации HTML контента (функция `_extract_and_clean_html`). Этот подход идеально подходит для `NormalizerService` в системе PBD (трансформация `l1_raw` HTML в `l2_normalized` текст).

## Архитектура очистки

Процесс состоит из трех основных шагов:
1. Декодирование (quopri decodestring).
2. Извлечение основного контента (устранение "шума" верстки) с помощью `readability-lxml`.
3. Жесткая санитаризация и удаление опасных/ненужных тегов с помощью `nh3` (Ammonia).

### Реализация

```python
import quopri
import re
from readability import Document
import nh3
from bs4 import BeautifulSoup

def _extract_and_clean_html(raw_html_content: str) -> dict[str, str]:
    """Decode, extract, and sanitize newsletter HTML."""
    try:
        # Декодирование quoted-printable (часто встречается в email)
        decoded_bytes = quopri.decodestring(raw_html_content.encode("utf-8"))
        clean_html_str = decoded_bytes.decode("utf-8", "ignore")
    except Exception:
        # Fallback
        clean_html_str = raw_html_content

    # Удаление NULL-байтов и управляющих символов, из-за которых может упасть парсер lxml
    # Оставляем только tab, newline и carriage return
    clean_html_str = re.sub(r"[\x00-\x08\x0b\x0c\x0e-\x1f]", "", clean_html_str)

    # Использование readability-lxml для извлечения главного контента статьи
    # Это вырезает шапки, подвалы, боковые панели и рекламу
    doc = Document(clean_html_str)
    extracted_body = doc.summary(html_partial=True)

    # Строгие белые списки для nh3 (Rust-based HTML sanitizer)
    ALLOWED_TAGS = {
        "p", "strong", "em", "u", "h3", "h4", "ul", "ol", "li",
        "a", "img", "br", "div", "span", "figure", "figcaption",
    }
    ALLOWED_ATTRIBUTES = {
        "a": {"href", "title"},
        "img": {"src", "alt", "width", "height"},
        "*": {"style"},
    }

    # Полная очистка: все теги не из ALLOWED_TAGS удаляются
    cleaned_body = nh3.clean(
        extracted_body, tags=ALLOWED_TAGS, attributes=ALLOWED_ATTRIBUTES
    )

    # Попытка извлечь заголовок
    title = doc.title()
    if not title or title == "no-title":
        soup = BeautifulSoup(cleaned_body, "html.parser")
        first_headline = soup.find(["h1", "h2", "h3"])
        title = first_headline.get_text(strip=True) if first_headline else "Newsletter"

    return {"title": title, "body": cleaned_body}
```

## Как применить в PBD (NormalizerService)

1. **Библиотеки**: Добавить в зависимости проекта `readability-lxml` и `nh3`. `nh3` — это Python-биндинг к библиотеке Ammonia на Rust, он работает на порядки быстрее и безопаснее, чем регулярные выражения или BeautifulSoup.
2. **Readability**: В PBD данные приходят не только из Email, но и из RSS или обычных веб-страниц. `Document(html).summary()` из `readability-lxml` — идеальный инструмент для извлечения основного текста статьи с любого веб-сайта. Он автоматически выкинет навигацию и рекламу.
3. **NH3**: Белые списки тегов и атрибутов из `LetterFeed` (`ALLOWED_TAGS`, `ALLOWED_ATTRIBUTES`) можно взять за основу и, при необходимости, адаптировать (например, запретить `style` или `img`, если мы хотим только чистый текст для векторации).
4. Этот пайплайн гарантирует, что на уровень `L2_normalized` попадет только чистый, безопасный и осмысленный контент, готовый для `ReasoningService` (LLM) и `EmbedderService` (векторизации).
