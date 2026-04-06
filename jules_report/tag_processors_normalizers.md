# Tag Processors & Normalizers (repo: feeds.fun)

## Логика Librarian

В репозитории `feeds.fun` в модуле `ffun.librarian` реализована продвинутая архитектура процессоров тегов и нормализации текста. Это может послужить мощнейшим бустом для `ReasoningService` (автогенерации K1/K2 тегов) в системе Personal Browser Dashboard (PBD).

### Очистка текста (Normalizers)

Логика очистки реализована в файле `ffun/librarian/text_cleaners.py`. Используется библиотека `BeautifulSoup`:

```python
def clear_html(text: str) -> str:
    soup = BeautifulSoup(text, "html.parser")

    # Удаляем служебные и медиа теги (скрипты, стили, мета, iframe, картинки)
    decomposed_tags = cast(list[Tag], soup.find_all(["script", "style", "meta", "iframe", "img"]))
    for decomposed_tag in decomposed_tags:
        decomposed_tag.decompose()

    tags = cast(list[object], soup.find_all())

    # Разворачиваем (unwrap) все теги, кроме списка разрешенных структурных элементов
    for tag in tags:
        if not isinstance(tag, Tag):
            continue

        if tag.name in {"h1", "h2", "h3", "h4", "h5", "h6", "a", "p", "li", "ul", "ol"}:
            continue

        tag.unwrap()

    simplified_html = str(soup)

    # Убираем сломанные суррогатные Unicode-символы
    unicoded_html = simplified_html.encode("utf-16", "surrogatepass").decode("utf-16", "ignore")

    # Убираем лишние пробелы и переносы
    cleaned_text = re.sub(r"\s+", " ", unicoded_html)

    return cleaned_text.strip()
```

Этот подход позволяет преобразовать `L1_raw` HTML в `L2_normalized` чистый текст, сохраняя только базовую семантическую структуру текста (заголовки, абзацы, списки, ссылки), что критично для последующего анализа LLM.

### Обработка через LLM (Tag Processors)

Логика автогенерации тегов реализована в файле `ffun/librarian/processors/llm_general.py`. Основной класс `Processor` отвечает за подготовку запросов к LLM и извлечение тегов:

1. **Подготовка текста**: `_text_to_process` форматирует текст по `entry_template`, применяет `text_cleaner` (например, `clear_html`) и обрезает его до `max_tokens` (используя `cut_text_to_max_tokens`).
2. **Отправка запросов (Чанкинг)**: Большие тексты разбиваются на чанки (`text_parts_intersection`), и для каждого создается свой `ChatRequest`.
3. **Промптинг (LLMGeneral)**: Запросы отправляются через выбранного `llm_provider`.
4. **Извлечение тегов**: `extract_tags` обрабатывает ответ с помощью функции `tag_extractor` (например, `dog_tags_extractor`), которая парсит ответ LLM и генерирует набор уникальных `RawTag`.

```python
    def extract_tags(self, responses: Sequence[ChatResponse]) -> list[RawTag]:
        raw_tags = set()
        tags: list[RawTag] = []

        for response in responses:
            raw_tags.update(self.tag_extractor(response.response_content()))

        for raw_tag in raw_tags:
            tags.append(
                RawTag(
                    raw_uid=raw_tag,
                    categories={TagCategory.free_form},
                )
            )

        return tags
```

### Как применить в PBD
- Использовать пайплайн `template -> cleaner -> token_cutter -> chunk_splitter` для `ReasoningService`.
- Внедрить `clear_html` (или его аналог с `readability`/`nh3`) для подготовки текстов для LLM, чтобы не тратить контекст (токены) на мусорный HTML.
- Заимствовать идею извлечения тегов (`tags_extractor`) для генерации иерархии тегов K1/K2, где промпт жестко задает формат возврата тегов, а регулярное выражение или парсер (как `dog_tags_extractor`) надежно их извлекает.
