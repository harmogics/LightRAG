# Entity Extraction: Извлечение Сущностей и Отношений с Помощью LLM

## Обзор

Entity Extraction Layer - это критический этап pipeline, где языковые модели (LLM) анализируют текстовые chunks и извлекают структурированную информацию о сущностях (entities) и их отношениях (relationships). Этот процесс преобразует неструктурированный текст в граф знаний.

## Архитектура

### Главная Функция

**Расположение**: `lightrag/operate.py:2010`

```python
async def extract_entities(
    chunks: dict[str, TextChunkSchema],
    global_config: dict[str, str],
    pipeline_status: dict = None,
    pipeline_status_lock = None,
    llm_response_cache: BaseKVStorage | None = None,
    text_chunks_storage: BaseKVStorage | None = None,
) -> list[tuple[dict, dict]]
```

### Параметры

| Параметр | Тип | Описание |
|----------|-----|----------|
| `chunks` | `dict[str, TextChunkSchema]` | Словарь chunks для обработки |
| `global_config` | `dict` | Конфигурация (LLM func, parameters, prompts) |
| `pipeline_status` | `dict` | Статус обработки для мониторинга |
| `pipeline_status_lock` | `asyncio.Lock` | Блокировка для безопасного обновления статуса |
| `llm_response_cache` | `BaseKVStorage` | Кеш для LLM ответов |
| `text_chunks_storage` | `BaseKVStorage` | Storage для chunks |

### Возвращаемое Значение

```python
list[tuple[dict, dict]]
# Список кортежей (entities_dict, relationships_dict)
# entities_dict: {entity_name: [entity_data, ...]}
# relationships_dict: {(source, target): [relationship_data, ...]}
```

## Трехэтапный Процесс Extraction

```
Input: Text Chunks
    ↓
┌──────────────────────────────────────────┐
│  STAGE 1: INITIAL EXTRACTION             │
│  LLM Agent: Entity Extraction Agent      │
│  • Prompt: entity_extraction_*_prompt    │
│  • Output: entities + relationships      │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│  STAGE 2: GLEANING PASS (Optional)       │
│  LLM Agent: Gleaning Agent               │
│  • Prompt: continue_extraction_*_prompt  │
│  • Output: missed/corrected items        │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│  STAGE 3: PARSING & VALIDATION           │
│  • Parse LLM output                      │
│  • Validate structure                    │
│  • Sanitize data                         │
│  • Filter invalid entries                │
└──────────────┬───────────────────────────┘
               │
               ▼
Output: Structured Entities + Relationships
```

## Stage 1: Initial Extraction

### Prompt Construction

```python
# Контекст для промпта
language = global_config["addon_params"].get(
    "language",
    DEFAULT_SUMMARY_LANGUAGE  # "English"
)
entity_types = global_config["addon_params"].get(
    "entity_types",
    DEFAULT_ENTITY_TYPES  # ["person", "organization", "location", ...]
)

# Базовый контекст
context_base = {
    "tuple_delimiter": PROMPTS["DEFAULT_TUPLE_DELIMITER"],  # "<|#|>"
    "completion_delimiter": PROMPTS["DEFAULT_COMPLETION_DELIMITER"],  # "<|COMPLETE|>"
    "entity_types": ", ".join(entity_types),
    "examples": formatted_examples,
    "language": language,
    "input_text": chunk["content"]
}

# System Prompt
system_prompt = PROMPTS["entity_extraction_system_prompt"].format(
    **context_base
)

# User Prompt
user_prompt = PROMPTS["entity_extraction_user_prompt"].format(
    **context_base
)
```

### System Prompt Структура

**Источник**: `lightrag/prompt.py:10`

Промпт содержит детальные инструкции для LLM:

#### 1. Entity Extraction Rules

```
---Instructions---
1. Entity Extraction & Output:
   - Identification: найти четко определенные сущности
   - Entity Details:
     * entity_name: Title Case нормализация
     * entity_type: категория из предопределенного списка
     * entity_description: comprehensive описание

   Output Format:
   entity<|#|>entity_name<|#|>entity_type<|#|>entity_description
```

**Пример Output**:
```
entity<|#|>Apple Inc<|#|>organization<|#|>Apple Inc is a technology company...
entity<|#|>Tim Cook<|#|>person<|#|>Tim Cook is the CEO of Apple Inc...
```

#### 2. Relationship Extraction Rules

```
2. Relationship Extraction & Output:
   - Identification: прямые, явные отношения между сущностями
   - N-ary Decomposition: разложение N-арных отношений на бинарные
   - Relationship Details:
     * source_entity: исходная сущность (Title Case)
     * target_entity: целевая сущность (Title Case)
     * relationship_keywords: high-level keywords (comma separated)
     * relationship_description: описание связи

   Output Format:
   relation<|#|>source_entity<|#|>target_entity<|#|>keywords<|#|>description
```

**Пример Output**:
```
relation<|#|>Tim Cook<|#|>Apple Inc<|#|>leadership, management<|#|>Tim Cook serves as CEO of Apple Inc
```

#### 3. Key Instructions

```
3. Delimiter Usage:
   - <|#|> это atomic marker, НЕ должен быть заполнен контентом

4. Relationship Direction:
   - Все отношения считаются undirected
   - (A → B) эквивалентно (B → A)

5. Output Order:
   - Сначала все entities
   - Затем все relationships
   - Relationships отсортированы по важности

6. Context & Objectivity:
   - Third person perspective
   - Избегать местоимений (I, you, this paper)
   - Явно называть субъекты

7. Language:
   - Весь output на языке {language}
   - Proper nouns сохраняются в оригинале

8. Completion Signal:
   - <|COMPLETE|> в конце extraction
```

### LLM Call

```python
# Вызов LLM с кешированием
final_result, timestamp = await use_llm_func_with_cache(
    user_prompt,
    use_llm_func,
    system_prompt=system_prompt,
    llm_response_cache=llm_response_cache,
    cache_type="extract",
    chunk_id=chunk_key,
    cache_keys_collector=cache_keys_collector
)
```

**Cache Key Format**:
```python
cache_key = f"extract:{chunk_id}:{md5(system_prompt + user_prompt)}"
```

### Example LLM Output

```
entity<|#|>World Athletics Championship<|#|>event<|#|>The World Athletics Championship is a global sports competition featuring top athletes in track and field.
entity<|#|>Tokyo<|#|>location<|#|>Tokyo is the host city of the World Athletics Championship.
entity<|#|>Noah Carter<|#|>person<|#|>Noah Carter is a sprinter who set a new record in the 100m sprint at the World Athletics Championship.
entity<|#|>100m Sprint Record<|#|>category<|#|>The 100m sprint record is a benchmark in athletics, recently broken by Noah Carter.
entity<|#|>Carbon-Fiber Spikes<|#|>equipment<|#|>Carbon-fiber spikes are advanced sprinting shoes that provide enhanced speed and traction.
relation<|#|>World Athletics Championship<|#|>Tokyo<|#|>event location, international competition<|#|>The World Athletics Championship is being hosted in Tokyo.
relation<|#|>Noah Carter<|#|>100m Sprint Record<|#|>athlete achievement, record-breaking<|#|>Noah Carter set a new 100m sprint record at the championship.
relation<|#|>Noah Carter<|#|>Carbon-Fiber Spikes<|#|>athletic equipment, performance boost<|#|>Noah Carter used carbon-fiber spikes to enhance performance during the race.
<|COMPLETE|>
```

## Stage 2: Gleaning Pass

### Концепция

**Gleaning** - это процесс "дожима" информации, где LLM делает второй проход для выявления:
- Пропущенных сущностей
- Пропущенных отношений
- Некорректно форматированных записей
- Неполных описаний

### Условие Активации

```python
entity_extract_max_gleaning = global_config["entity_extract_max_gleaning"]

if entity_extract_max_gleaning > 0:
    # Выполняем gleaning pass
```

**Default**: `entity_extract_max_gleaning = 1` (один gleaning pass)

### History Context

```python
# Создание истории диалога для gleaning
history = pack_user_ass_to_openai_messages(
    entity_extraction_user_prompt,
    final_result  # Результат initial extraction
)

# History format:
# [
#     {"role": "user", "content": entity_extraction_user_prompt},
#     {"role": "assistant", "content": final_result}
# ]
```

### Gleaning Prompt

**Источник**: `PROMPTS["entity_continue_extraction_user_prompt"]`

```
---Task---
Based on the last extraction task, identify and extract any **missed or
incorrectly formatted** entities and relationships from the input text.

---Instructions---
1. Focus on Corrections/Additions:
   - Do NOT re-output correctly extracted items
   - If missed: extract and output now
   - If truncated/incorrect: re-output corrected version

2. Output Format:
   - Same format as initial extraction
   - Only NEW or CORRECTED items

3. Completion Signal:
   - Output <|COMPLETE|> when done
```

### LLM Call with History

```python
glean_result, timestamp = await use_llm_func_with_cache(
    entity_continue_extraction_user_prompt,
    use_llm_func,
    system_prompt=entity_extraction_system_prompt,
    llm_response_cache=llm_response_cache,
    history_messages=history,  # Передаем историю
    cache_type="extract",
    chunk_id=chunk_key,
    cache_keys_collector=cache_keys_collector
)
```

### Merging Results

```python
# Парсинг gleaning результатов
glean_nodes, glean_edges = await _process_extraction_result(
    glean_result, chunk_key, timestamp, file_path, ...
)

# Слияние с initial extraction
for entity_name, glean_entities in glean_nodes.items():
    if entity_name in maybe_nodes:
        # Сравнение длины описаний
        original_desc_len = len(
            maybe_nodes[entity_name][0].get("description", "") or ""
        )
        glean_desc_len = len(
            glean_entities[0].get("description", "") or ""
        )

        # Выбор лучшего описания
        if glean_desc_len > original_desc_len:
            maybe_nodes[entity_name] = list(glean_entities)
        # Иначе оставляем original
    else:
        # Новая сущность из gleaning
        maybe_nodes[entity_name] = list(glean_entities)

# Аналогично для relationships
```

**Критерий Выбора**: **длина описания** (более длинное = более информативное)

## Stage 3: Parsing & Validation

### Функция Парсинга

**Расположение**: `lightrag/operate.py` в `_process_extraction_result()`

```python
async def _process_extraction_result(
    result: str,
    chunk_key: str,
    timestamp: int,
    file_path: str,
    tuple_delimiter: str = "<|#|>",
    completion_delimiter: str = "<|COMPLETE|>",
) -> tuple[dict, dict]:
    """
    Парсинг и валидация LLM output

    Returns:
        (entities_dict, relationships_dict)
    """
```

### Parsing Algorithm

```
Input: LLM raw output
    ↓
[1. Удаление Completion Delimiter]
    ↓
    result = result.replace(completion_delimiter, "")
    ↓
[2. Разделение на Строки]
    ↓
    lines = result.strip().split("\n")
    ↓
[3. Обработка Каждой Строки]
    ↓
    for line in lines:
        # Пропуск пустых строк
        if not line.strip():
            continue

        # Разделение по delimiter
        fields = line.split(tuple_delimiter)

        # Проверка типа записи
        if fields[0] == "entity":
            # [3.1] Entity Parsing
        elif fields[0] == "relation":
            # [3.2] Relationship Parsing
        else:
            # Неизвестный тип, пропускаем
            logger.warning(f"Unknown record type: {fields[0]}")
    ↓
Output: (entities_dict, relationships_dict)
```

### Entity Parsing

```python
# Expected format:
# entity<|#|>entity_name<|#|>entity_type<|#|>entity_description

if len(fields) < 4:
    logger.warning(f"Invalid entity format: {line}")
    continue

entity_name = fields[1].strip()
entity_type = fields[2].strip()
entity_description = fields[3].strip()

# Валидация
if not entity_name or not entity_type:
    logger.warning(f"Empty entity name or type: {line}")
    continue

# Нормализация entity_name (Title Case)
entity_name = normalize_entity_name(entity_name)

# Санитизация текста
entity_description = sanitize_text_for_encoding(entity_description)

# Создание entity data
entity_data = {
    "entity_name": entity_name,
    "entity_type": entity_type,
    "description": entity_description,
    "source_id": chunk_key,
    "file_path": file_path,
    "created_at": timestamp
}

# Добавление в словарь
if entity_name not in entities_dict:
    entities_dict[entity_name] = []
entities_dict[entity_name].append(entity_data)
```

### Relationship Parsing

```python
# Expected format:
# relation<|#|>source<|#|>target<|#|>keywords<|#|>description

if len(fields) < 5:
    logger.warning(f"Invalid relation format: {line}")
    continue

source_entity = fields[1].strip()
target_entity = fields[2].strip()
keywords = fields[3].strip()
description = fields[4].strip()

# Валидация
if not source_entity or not target_entity:
    logger.warning(f"Empty source or target: {line}")
    continue

# Нормализация имен (Title Case)
source_entity = normalize_entity_name(source_entity)
target_entity = normalize_entity_name(target_entity)

# Санитизация
keywords = sanitize_text_for_encoding(keywords)
description = sanitize_text_for_encoding(description)

# Создание relationship key (sorted для undirected graph)
edge_key = tuple(sorted([source_entity, target_entity]))

# Создание relationship data
relation_data = {
    "source_entity": source_entity,
    "target_entity": target_entity,
    "keywords": keywords,
    "description": description,
    "source_id": chunk_key,
    "file_path": file_path,
    "created_at": timestamp
}

# Добавление в словарь
if edge_key not in relationships_dict:
    relationships_dict[edge_key] = []
relationships_dict[edge_key].append(relation_data)
```

### Entity Name Normalization

```python
def normalize_entity_name(name: str) -> str:
    """
    Нормализация имени сущности для консистентности

    Rules:
    1. Strip whitespace
    2. Title Case (capitalize first letter of each word)
    3. Handle abbreviations (keep uppercase)
    """
    name = name.strip()

    # Проверка на abbreviation (все заглавные)
    if name.isupper() and len(name) <= 5:
        return name  # Сохраняем как есть (например, "NASA", "FBI")

    # Title Case для остальных
    return name.title()

# Примеры:
# "apple inc" → "Apple Inc"
# "JOHN SMITH" → "John Smith"
# "NASA" → "NASA" (сохраняется)
# "  tokyo  " → "Tokyo"
```

## Параллельная Обработка Chunks

### Async Processing

```python
# Обработка chunks параллельно
async def _process_single_content(
    chunk_key_dp: tuple[str, TextChunkSchema]
) -> tuple[dict, dict]:
    """Обработка одного chunk"""
    # ... extraction logic ...
    return (maybe_nodes, maybe_edges)

# Создание tasks для всех chunks
tasks = [
    _process_single_content(chunk_item)
    for chunk_item in ordered_chunks
]

# Параллельное выполнение
chunk_results = await asyncio.gather(*tasks)

# chunk_results: [
#     (entities_dict_1, relationships_dict_1),
#     (entities_dict_2, relationships_dict_2),
#     ...
# ]
```

### Semaphore Control

```python
# Ограничение количества параллельных LLM calls
llm_semaphore = asyncio.Semaphore(
    global_config["llm_model_max_async"]  # Default: 16
)

async def _process_with_limit(chunk):
    async with llm_semaphore:
        return await _process_single_content(chunk)

tasks = [_process_with_limit(chunk) for chunk in chunks]
results = await asyncio.gather(*tasks)
```

### Priority-Based Execution

```python
# Chunks обрабатываются с приоритетами
# Priority 0-7 для extraction (по порядку chunk_order_index)

chunk_priority = min(
    chunk_data["chunk_order_index"],
    7  # Maximum priority 7
)

# Chunks в начале документа имеют больший приоритет
# Предполагается что важные сущности упоминаются раньше
```

## Кеширование LLM Responses

### Cache Function

**Расположение**: `lightrag/utils.py:1594`

```python
async def use_llm_func_with_cache(
    prompt: str,
    llm_func: callable,
    system_prompt: str | None = None,
    llm_response_cache: BaseKVStorage | None = None,
    history_messages: list | None = None,
    cache_type: str = "extract",
    chunk_id: str | None = None,
    cache_keys_collector: list | None = None,
) -> tuple[str, int]:
    """
    LLM вызов с кешированием

    Returns:
        (response_text, timestamp)
    """
```

### Cache Key Generation

```python
# Создание уникального cache key
cache_content = system_prompt + prompt
if history_messages:
    cache_content += json.dumps(history_messages)

cache_key = compute_mdhash_id(cache_content)

# Добавление типа и chunk_id
if cache_type and chunk_id:
    cache_key = f"{cache_type}:{chunk_id}:{cache_key}"

# Пример:
# "extract:chunk-abc123:d41d8cd98f00b204e9800998ecf8427e"
```

### Cache Lookup

```python
# Проверка кеша
if llm_response_cache is not None:
    cached_data = await llm_response_cache.get_by_id(cache_key)
    if cached_data:
        logger.debug(f"Cache HIT for {cache_key}")
        return (
            cached_data["response"],
            cached_data["timestamp"]
        )

# Cache MISS - вызываем LLM
logger.debug(f"Cache MISS for {cache_key}")
response = await llm_func(
    prompt,
    system_prompt=system_prompt,
    history_messages=history_messages
)

# Сохраняем в кеш
timestamp = int(time.time())
await llm_response_cache.upsert({
    cache_key: {
        "response": response,
        "timestamp": timestamp,
        "cache_type": cache_type,
        "chunk_id": chunk_id
    }
})

return (response, timestamp)
```

### Cache Benefits

- ✅ **Избегание дублирующих LLM calls** для одинаковых chunks
- ✅ **Ускорение re-processing** при повторной индексации
- ✅ **Снижение стоимости** API calls
- ✅ **Детерминированность** результатов

## Output Structure

### Entities Dictionary

```python
{
    "Apple Inc": [
        {
            "entity_name": "Apple Inc",
            "entity_type": "organization",
            "description": "Apple Inc is a technology company...",
            "source_id": "chunk-abc123",
            "file_path": "doc1.pdf",
            "created_at": 1705012345
        },
        {
            "entity_name": "Apple Inc",
            "entity_type": "organization",
            "description": "Apple Inc designs and manufactures...",
            "source_id": "chunk-def456",
            "file_path": "doc1.pdf",
            "created_at": 1705012346
        }
    ],
    "Tim Cook": [
        {
            "entity_name": "Tim Cook",
            "entity_type": "person",
            "description": "Tim Cook is the CEO of Apple Inc...",
            "source_id": "chunk-abc123",
            "file_path": "doc1.pdf",
            "created_at": 1705012345
        }
    ]
}
```

### Relationships Dictionary

```python
{
    ("Apple Inc", "Tim Cook"): [
        {
            "source_entity": "Tim Cook",
            "target_entity": "Apple Inc",
            "keywords": "leadership, management",
            "description": "Tim Cook serves as CEO of Apple Inc",
            "source_id": "chunk-abc123",
            "file_path": "doc1.pdf",
            "created_at": 1705012345
        }
    ],
    ("Apple Inc", "iPhone"): [
        {
            "source_entity": "Apple Inc",
            "target_entity": "iPhone",
            "keywords": "product, manufacturing",
            "description": "Apple Inc designs and manufactures iPhone",
            "source_id": "chunk-def456",
            "file_path": "doc1.pdf",
            "created_at": 1705012346
        }
    ]
}
```

**Примечание**: Edge keys всегда sorted для undirected graph

## Error Handling

### LLM Call Failures

```python
try:
    result, timestamp = await use_llm_func_with_cache(...)
except Exception as e:
    # Prefix с chunk info для debugging
    error_msg = f"[Chunk {chunk_key}] LLM call failed: {e}"
    logger.error(error_msg)

    # Возвращаем пустые результаты для этого chunk
    return ({}, {})
```

### Parsing Failures

```python
try:
    entities, relationships = await _process_extraction_result(...)
except Exception as e:
    logger.error(f"Failed to parse extraction result: {e}")
    logger.debug(f"Raw result: {result}")

    # Продолжаем с частичными результатами
    return ({}, {})
```

### Invalid Format Handling

```python
# Логирование invalid records без прерывания процесса
if len(fields) < expected_fields:
    logger.warning(
        f"[Chunk {chunk_key}] Invalid format: {line}\n"
        f"Expected {expected_fields} fields, got {len(fields)}"
    )
    continue  # Пропускаем эту запись, продолжаем parsing
```

## Monitoring & Metrics

### Progress Tracking

```python
processed_chunks = 0
total_chunks = len(ordered_chunks)

async def _process_with_tracking(chunk):
    nonlocal processed_chunks

    result = await _process_single_content(chunk)

    processed_chunks += 1
    progress = (processed_chunks / total_chunks) * 100

    # Обновление pipeline status
    async with pipeline_status_lock:
        pipeline_status["latest_message"] = (
            f"Entity extraction: {processed_chunks}/{total_chunks} "
            f"chunks ({progress:.1f}%)"
        )

    return result
```

### Extraction Statistics

```python
# Подсчет извлеченных entities и relationships
total_entities = sum(len(entities) for entities in all_entities.values())
total_relationships = sum(
    len(rels) for rels in all_relationships.values()
)

logger.info(
    f"Extracted {total_entities} entity mentions "
    f"and {total_relationships} relationship mentions"
)
```

## Следующий Этап

После extraction сущностей и отношений система переходит к **[Graph Construction](04-graph-construction.md)** - процессу слияния извлеченных данных в единый граф знаний.
