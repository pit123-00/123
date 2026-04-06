# Технический отчет: postgres-graph-rag

Этот отчет описывает технические и логические аспекты библиотеки `postgres-graph-rag` (версия для работы полностью внутри PostgreSQL). Он составлен в соответствии с запрошенной структурой и основан на анализе исходного кода проекта.

## 1. Техническая реализация и архитектура

### Максимализм на базе PostgreSQL
Проект следует философии "Postgres Maximalism", избегая внешних графовых баз данных. Архитектура основывается на расширении `pgvector` и стандартных возможностях SQL.
Библиотека написана на асинхронном Python с использованием `psycopg` (версии 3) для взаимодействия с базой данных, что позволяет эффективно использовать пулы соединений (`AsyncConnectionPool`) в высоконагруженных асинхронных приложениях (например, с FastAPI).

### Схема данных (Forever Schema)
Схема данных спроектирована так, чтобы минимизировать необходимость миграций. Используется тип `JSONB` для хранения произвольных метаданных как для узлов (Entities), так и для ребер (Relationships).
Для пространств имен (мультиарендности) во всех таблицах присутствует обязательное поле `namespace`, и оно включено в уникальные индексы, чтобы гарантировать строгую изоляцию данных между арендаторами.

**Таблица `graph_nodes` (Узлы/Сущности):**
```sql
CREATE TABLE IF NOT EXISTS graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace VARCHAR(255) NOT NULL,
    content TEXT NOT NULL, -- Содержимое узла (название сущности)
    embedding VECTOR(n),   -- Семантический вектор (размерность 'n' зависит от модели)
    metadata JSONB DEFAULT '{}', -- Метаданные (например, источник)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
)
```
Индексы:
* `UNIQUE INDEX idx_graph_nodes_namespace_content ON graph_nodes (namespace, content)`: Защищает от дублирования узлов в одном пространстве имен.
* `INDEX idx_graph_nodes_embedding ON graph_nodes USING hnsw (embedding vector_cosine_ops)`: Индекс HNSW для быстрого векторного поиска (косинусное сходство).

**Таблица `graph_edges` (Связи/Ребра):**
```sql
CREATE TABLE IF NOT EXISTS graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace VARCHAR(255) NOT NULL,
    source_node_id UUID REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id UUID REFERENCES graph_nodes(id) ON DELETE CASCADE,
    relation TEXT NOT NULL, -- Тип связи (предикат)
    weight FLOAT DEFAULT 1.0, -- Вес связи
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(namespace, source_node_id, target_node_id, relation) -- Защита от дублей связей
)
```
Индексы созданы для быстрого обхода графа по `source_node_id`, `target_node_id` и `namespace`.

## 2. Функции и логическая операция

### Загрузка и извлечение данных (Инкрементальная загрузка)
Процесс загрузки состоит из нескольких этапов, которые можно распараллелить (см. `core.py: PostgresGraphRAG.add_texts`):
1. **Chunking (Разбиение):** Текст разбивается на части (по умолчанию простой алгоритм, но можно внедрить свой через инверсию управления).
2. **Extraction (Извлечение через LLM):** Облачные модели (OpenAI/Google) параллельно анализируют чанки с помощью `LLMExtractor`. Они возвращают список триплетов (Subject -> Predicate -> Object). Для этого используются `Structured Outputs` или `GenerateContentConfig` с возвратом JSON (`Pydantic` модели `ExtractionResult` / `Triplet`).
3. **Embedding (Векторизация):** Уникальные извлеченные узлы (субъекты и объекты) пакетом отправляются на векторизацию (получение эмбеддингов).
4. **Атомарный Upsert:** В базе данных (в `database.py`) выполняется логика `ON CONFLICT DO UPDATE`.
   * **Узлы (`upsert_nodes_batch`):** При конфликте по `(namespace, content)` обновляются `embedding` и `metadata` (используется объединение JSONB: `metadata || EXCLUDED.metadata`). Возвращаются их `id`.
   * **Ребра (`upsert_edges_batch`):** При конфликте по `(namespace, source_node_id, target_node_id, relation)` обновляется вес и метаданные.

### Выполнение запросов (Гибридный поиск)
Поиск (в `core.py: query`) происходит в два этапа прямо внутри Postgres, избегая множественных запросов от LLM-агента:
1. **Векторный поиск (Semantic Search):**
   * Векторизуется запрос пользователя (`query_embedding`).
   * В `database.py: vector_search` выполняется поиск ближайших узлов по косинусному сходству с использованием `pgvector` (`embedding <=> %s::vector`). Выбираются `top_k` начальных узлов (seed nodes).

2. **Рекурсивный обход (Graph Traversal):**
   * В `database.py: traverse_graph` выполняется многошаговый обход графа (до `max_hops` шагов) от начальных узлов.
   * **Логика Recursive CTE (`WITH RECURSIVE graph_expansion AS ...`):**
     * **Base case:** Выбираются `seed_nodes`. Они получают `depth = 0` и массив посещенных узлов `visited = ARRAY[id]`.
     * **Recursive step:** Ищутся соседи для текущих узлов в `graph_expansion`, проверяя связи в `graph_edges` в обоих направлениях (`JOIN` по `source_node_id` или `target_node_id`). Увеличивается глубина (`depth + 1`), текущий узел добавляется в `visited`, предотвращая зацикливание (`NOT (n.id = ANY(ge.visited))`). Обход останавливается при достижении `max_hops`.
   * Результат обхода объединяется с таблицей связей, чтобы вернуть как узлы, так и отношения между ними в виде контекста для RAG.

## 3. Обмен информацией, память и потоки данных

Потоки данных внутри `postgres-graph-rag`:
* **Входной текст** -> `Chunker` -> Массив чанков.
* Массив чанков -> Запросы к **LLM (OpenAI/Google Gemini)** с системным промптом -> JSON (Pydantic: `List[Triplet]`). Сущности очищаются и нормализуются.
* Уникальные сущности из триплетов -> Запросы к **Embedding API** -> Массив векторов (размерности, например, 1536).
* Пакет узлов и связей (с векторами и метаданными) -> **PostgreSQL (AsyncConnectionPool)** -> Таблицы `graph_nodes` и `graph_edges` через атомарные upsert-операции в рамках одной транзакции (ACID).
* **Пользовательский запрос** -> Запрос к Embedding API -> Вектор запроса.
* Вектор запроса -> **PostgreSQL `pgvector` (HNSW)** -> Seed nodes -> **PostgreSQL Recursive CTE** -> Подграф (Узлы и ребра в пределах `max_hops`).
* Подграф (через Pandas DataFrame для форматирования) -> Форматированный текстовый контекст (Entities and Relationships) -> Передается на вход LLM для финального ответа (этот шаг выполняется вызывающим кодом, сама библиотека возвращает обогащенный контекст `format_context`).

Архитектура строго асинхронна (используется `asyncio`), минимизирует количество походов в базу данных с помощью пакетирования и объединения запросов в транзакции. Сохраняет целостность графа и семантических векторов в едином хранилище без внешних графовых движков.
