# Token Chunking & Pricing Logic (repo: feeds.fun)

Для мониторинга использования токенов в LLM (и предотвращения переполнения контекста) в репозитории `feeds.fun` (`ffun/ffun/openai/provider_interface.py`) реализована интеграция с библиотекой `tiktoken`. Это критически важный механизм для контроля затрат и отправки телеметрии (`reasoning.tokens_used`) в PBD.

## Взаимодействие с tiktoken

### 1. Получение кодировщика (Encoding)
Поскольку разные модели OpenAI используют разные словари, необходимо получить правильный энкодер.

```python
import tiktoken
import functools

@functools.cache
def _get_encoding(model: str) -> tiktoken.Encoding:
    """Get tiktoken encoding for a given model."""
    try:
        return tiktoken.encoding_for_model(model)
    except KeyError:
        # Fallback если модель неизвестна tiktoken
        return tiktoken.get_encoding(settings.fallback_model_encoding)
```
Использование `@functools.cache` оптимизирует процесс, так как загрузка словарей происходит один раз.

### 2. Оценка количества токенов (estimate_tokens)
Эта функция позволяет точно рассчитать количество токенов в промпте (system) и пользовательском вводе (user) до отправки запроса в API.

```python
class OpenAIInterface:
    additional_tokens_per_message: int = 10

    def estimate_tokens(self, config: LLMConfiguration, text: str) -> int:
        encoding = _get_encoding(config.model)

        system_tokens = (
            self.additional_tokens_per_message +
            len(encoding.encode("system")) +
            len(encoding.encode(config.system))
        )

        text_tokens = (
            self.additional_tokens_per_message +
            len(encoding.encode("user")) +
            len(encoding.encode(text))
        )

        return system_tokens + text_tokens
```
Здесь `additional_tokens_per_message` учитывает накладные расходы OpenAI на форматирование сообщений в формате ChatML.

### 3. Чанкинг (Token Chunking)
В файле `ffun/librarian/processors/llm_general.py` (как было описано в отчете по Tag Processors) используется функция `cut_text_to_max_tokens` (и `split_text_according_to_tokens`).
С помощью `estimate_tokens` система может заранее разбить слишком длинный `L2_normalized` текст на части (чанки), чтобы избежать ошибки переполнения контекста, и обрабатывать каждый чанк отдельным запросом.

## Как применить в PBD (ReasoningService & OpenTelemetry)

1. **Оценка до отправки**: Использовать логику `estimate_tokens` в `ReasoningService` перед вызовом OpenAI/Gemini. Это позволит:
   - Обрезать/разбить текст, если он превышает лимит модели (Token Chunking).
   - Заранее отклонять слишком большие "мусорные" статьи.
2. **OpenTelemetry**: После (или до) выполнения запроса, количество токенов (как рассчитанное, так и фактическое из ответа LLM `answer.usage.total_tokens`) можно пробрасывать в трейсы OpenTelemetry:
   ```python
   span.set_attribute("reasoning.tokens_used_prompt", prompt_tokens)
   span.set_attribute("reasoning.tokens_used_completion", completion_tokens)
   span.set_attribute("llm.model", model_name)
   ```
3. **Pricing Logic**: Зная количество потребленных токенов для каждого Универсального Атома, можно легко агрегировать данные в БД и рассчитывать фактическую стоимость работы AI конвейера за день/месяц.
