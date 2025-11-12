# Graph Construction: Построение Графа Знаний

## Обзор

Graph Construction Layer отвечает за слияние извлеченных из chunks сущностей и отношений в единый, согласованный граф знаний. Этот процесс включает дедупликацию сущностей, агрегацию описаний с помощью LLM, и синхронизацию между Graph DB и Vector DB.

## Архитектура

### Главная Функция

**Расположение**: `lightrag/operate.py:1579`

```python
async def merge_nodes_and_edges(
    chunk_results: list[tuple[dict, dict]],
    knowledge_graph_inst: BaseGraphStorage,
    entity_vdb: BaseVectorStorage,
    relationships_vdb: BaseVectorStorage,
    global_config: dict[str, str],
    full_entities_storage: BaseKVStorage = None,
    full_relations_storage: BaseKVStorage = None,
    doc_id: str = None,
    pipeline_status: dict = None,
    pipeline_status_lock = None,
    llm_response_cache: BaseKVStorage | None = None,
    current_file_number: int = 0,
    total_files: int = 0,
    file_path: str = "unknown_source",
) -> None
```

### Параметры

| Параметр | Тип | Описание |
|----------|-----|----------|
| `chunk_results` | `list[tuple[dict, dict]]` | Результаты extraction: (entities, relationships) |
| `knowledge_graph_inst` | `BaseGraphStorage` | Graph database storage |
| `entity_vdb` | `BaseVectorStorage` | Vector DB для entities |
| `relationships_vdb` | `BaseVectorStorage` | Vector DB для relationships |
| `global_config` | `dict` | Конфигурация (LLM, tokenizer, limits) |
| `full_entities_storage` | `BaseKVStorage` | K-V storage для entities |
| `full_relations_storage` | `BaseKVStorage` | K-V storage для relationships |
| `llm_response_cache` | `BaseKVStorage` | Кеш для LLM summary calls |

## Двухфазный Процесс Merging

```
Input: Extracted Entities + Relationships
    ↓
┌─────────────────────────────────────────────┐
│  PHASE 1: ENTITY PROCESSING                 │
│  ┌───────────────────────────────────────┐ │
│  │  For Each Entity (Parallel):          │ │
│  │  1. Acquire Keyed Lock                │ │
│  │  2. Check if Entity Exists            │ │
│  │  3. Merge Descriptions (LLM Agent)    │ │
│  │  4. Update Graph DB                   │ │
│  │  5. Update Vector DB                  │ │
│  │  6. Release Lock                      │ │
│  └───────────────────────────────────────┘ │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  PHASE 2: RELATIONSHIP PROCESSING           │
│  ┌───────────────────────────────────────┐ │
│  │  For Each Relationship (Parallel):    │ │
│  │  1. Acquire Keyed Locks (both nodes)  │ │
│  │  2. Ensure Both Entities Exist        │ │
│  │  3. Check if Edge Exists              │ │
│  │  4. Merge Descriptions (LLM Agent)    │ │
│  │  5. Update Graph DB                   │ │
│  │  6. Update Vector DB                  │ │
│  │  7. Release Locks                     │ │
│  └───────────────────────────────────────┘ │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
           Updated Knowledge Graph
```

## Phase 1: Entity Processing

### Step 1: Collection & Aggregation

```python
# Сбор всех entities из chunk results
all_nodes = defaultdict(list)

for maybe_nodes, maybe_edges in chunk_results:
    for entity_name, entities in maybe_nodes.items():
        all_nodes[entity_name].extend(entities)

# Результат:
# {
#     "Apple Inc": [entity_data_1, entity_data_2, entity_data_3],
#     "Tim Cook": [entity_data_1],
#     ...
# }
```

### Step 2: Parallel Processing with Keyed Locks

```python
# Semaphore для ограничения параллелизма
graph_max_async = global_config.get("llm_model_max_async", 4) * 2
semaphore = asyncio.Semaphore(graph_max_async)  # Default: 8

async def _locked_process_entity_name(entity_name, entities):
    async with semaphore:
        # Namespace для locks
        workspace = global_config.get("workspace", "")
        namespace = f"{workspace}:GraphDB" if workspace else "GraphDB"

        # Keyed lock для конкретной сущности
        async with get_storage_keyed_lock(
            [entity_name],
            namespace=namespace,
            enable_logging=False
        ):
            # === CRITICAL SECTION ===
            # Только один concurrent process может обрабатывать
            # эту сущность одновременно

            return await _merge_nodes_then_upsert(
                entity_name,
                entities,
                knowledge_graph_inst,
                global_config,
                pipeline_status,
                pipeline_status_lock,
                llm_response_cache
            )

# Создание tasks для всех entities
tasks = [
    _locked_process_entity_name(entity_name, entities)
    for entity_name, entities in all_nodes.items()
]

# Параллельное выполнение
processed_entities = await asyncio.gather(*tasks)
```

**Зачем Keyed Locks?**
- Предотвращение race conditions при одновременном обновлении одной entity
- Гарантия консистентности данных в Graph DB
- Корректное слияние descriptions из разных sources

### Step 3: Entity Merging Logic

**Функция**: `_merge_nodes_then_upsert()` в `operate.py`

```python
async def _merge_nodes_then_upsert(
    entity_name: str,
    entities: list[dict],
    knowledge_graph_inst: BaseGraphStorage,
    global_config: dict,
    pipeline_status: dict,
    pipeline_status_lock,
    llm_response_cache: BaseKVStorage,
) -> dict:
    """
    Слияние множественных упоминаний entity в одну запись

    Workflow:
    1. Check if entity exists in graph
    2. If exists: merge descriptions
    3. If new: create new entity
    4. Upsert to graph DB
    """
```

#### Algorithm:

```
Input: entity_name, list[entity_data]
    ↓
[1. Проверка Существования Entity]
    ↓
    existing_node = await knowledge_graph_inst.get_node(entity_name)
    ↓
[2. Извлечение Descriptions]
    ↓
    new_descriptions = [
        entity["description"]
        for entity in entities
        if entity.get("description")
    ]
    ↓
[3. Слияние с Существующим Description (если есть)]
    ↓
    if existing_node and existing_node.get("description"):
        all_descriptions = [existing_node["description"]] + new_descriptions
    else:
        all_descriptions = new_descriptions
    ↓
[4. LLM-Based Description Summarization]
    ↓
    merged_description, llm_used = await _handle_entity_relation_summary(
        description_type="Entity",
        entity_or_relation_name=entity_name,
        description_list=all_descriptions,
        seperator="\n\n",
        global_config=global_config,
        llm_response_cache=llm_response_cache
    )
    ↓
[5. Создание/Обновление Entity Data]
    ↓
    entity_data = {
        "entity_name": entity_name,
        "entity_type": entities[0]["entity_type"],  # Берем из первого
        "description": merged_description,
        "source_id": ",".join(set(e["source_id"] for e in entities)),
        "file_path": entities[0]["file_path"],
        "created_at": entities[0]["created_at"]
    }
    ↓
[6. Upsert to Graph DB]
    ↓
    await knowledge_graph_inst.upsert_node(
        entity_name,
        node_data=entity_data
    )
    ↓
Output: entity_data
```

### Step 4: Vector DB Update

```python
if entity_vdb is not None and entity_data:
    # Подготовка данных для vector DB
    entity_id = compute_mdhash_id(entity_name, prefix="ent-")

    data_for_vdb = {
        entity_id: {
            "entity_name": entity_data["entity_name"],
            "entity_type": entity_data["entity_type"],
            "content": (
                f"{entity_data['entity_name']}\n"
                f"{entity_data['description']}"
            ),
            "source_id": entity_data["source_id"],
            "file_path": entity_data.get("file_path", "unknown_source")
        }
    }

    # Upsert с retry logic
    await safe_vdb_operation_with_exception(
        operation=lambda: entity_vdb.upsert(data_for_vdb),
        operation_name="entity_upsert",
        entity_name=entity_name,
        max_retries=3,
        retry_delay=0.1
    )
```

**Content Format для Embedding**:
```
{entity_name}
{description}

Пример:
Apple Inc
Apple Inc is a technology company that designs, develops, and sells consumer electronics, computer software, and online services. The company is known for its innovative products such as iPhone, iPad, and Mac computers.
```

## Phase 2: Relationship Processing

### Step 1: Collection & Aggregation

```python
# Сбор всех relationships из chunk results
all_edges = defaultdict(list)

for maybe_nodes, maybe_edges in chunk_results:
    for edge_key, edges in maybe_edges.items():
        # Нормализация edge key (sorted для undirected graph)
        sorted_edge_key = tuple(sorted(edge_key))
        all_edges[sorted_edge_key].extend(edges)

# Результат:
# {
#     ("Apple Inc", "Tim Cook"): [relation_data_1, relation_data_2],
#     ("Apple Inc", "iPhone"): [relation_data_1],
#     ...
# }
```

### Step 2: Parallel Processing with Keyed Locks

```python
async def _locked_process_edge(edge_key, edges):
    async with semaphore:
        source_entity, target_entity = edge_key

        # Keyed lock для обеих сущностей
        async with get_storage_keyed_lock(
            [source_entity, target_entity],
            namespace=namespace,
            enable_logging=False
        ):
            # === CRITICAL SECTION ===
            # Гарантия что обе сущности не изменяются
            # во время обработки этого relationship

            return await _merge_edges_then_upsert(
                source_entity,
                target_entity,
                edges,
                knowledge_graph_inst,
                global_config,
                pipeline_status,
                pipeline_status_lock,
                llm_response_cache
            )

# Создание tasks для всех relationships
tasks = [
    _locked_process_edge(edge_key, edges)
    for edge_key, edges in all_edges.items()
]

# Параллельное выполнение
processed_relationships = await asyncio.gather(*tasks)
```

### Step 3: Relationship Merging Logic

**Функция**: `_merge_edges_then_upsert()` в `operate.py`

```python
async def _merge_edges_then_upsert(
    source_entity: str,
    target_entity: str,
    edges: list[dict],
    knowledge_graph_inst: BaseGraphStorage,
    global_config: dict,
    pipeline_status: dict,
    pipeline_status_lock,
    llm_response_cache: BaseKVStorage,
) -> dict:
    """
    Слияние множественных упоминаний relationship в одну запись
    """
```

#### Algorithm:

```
Input: source_entity, target_entity, list[relation_data]
    ↓
[1. Ensure Both Entities Exist]
    ↓
    source_node = await knowledge_graph_inst.get_node(source_entity)
    if not source_node:
        # Создаем недостающую сущность
        await knowledge_graph_inst.upsert_node(
            source_entity,
            node_data={
                "entity_name": source_entity,
                "entity_type": "inferred",
                "description": f"Entity inferred from relationship",
                "source_id": edges[0]["source_id"]
            }
        )

    target_node = await knowledge_graph_inst.get_node(target_entity)
    if not target_node:
        # Создаем недостающую сущность
        await knowledge_graph_inst.upsert_node(...)
    ↓
[2. Проверка Существования Edge]
    ↓
    existing_edge = await knowledge_graph_inst.get_edge(
        source_entity,
        target_entity
    )
    ↓
[3. Извлечение Descriptions]
    ↓
    new_descriptions = [
        edge["description"]
        for edge in edges
        if edge.get("description")
    ]
    ↓
[4. Слияние Keywords]
    ↓
    all_keywords = []
    for edge in edges:
        keywords = edge.get("keywords", "")
        all_keywords.extend(k.strip() for k in keywords.split(","))

    # Дедупликация и сортировка
    unique_keywords = sorted(set(all_keywords))
    merged_keywords = ", ".join(unique_keywords)
    ↓
[5. LLM-Based Description Summarization]
    ↓
    if existing_edge and existing_edge.get("description"):
        all_descriptions = [existing_edge["description"]] + new_descriptions
    else:
        all_descriptions = new_descriptions

    merged_description, llm_used = await _handle_entity_relation_summary(
        description_type="Relationship",
        entity_or_relation_name=f"{source_entity} -> {target_entity}",
        description_list=all_descriptions,
        seperator="\n\n",
        global_config=global_config,
        llm_response_cache=llm_response_cache
    )
    ↓
[6. Создание/Обновление Edge Data]
    ↓
    edge_data = {
        "source_entity": source_entity,
        "target_entity": target_entity,
        "keywords": merged_keywords,
        "description": merged_description,
        "source_id": ",".join(set(e["source_id"] for e in edges)),
        "file_path": edges[0]["file_path"],
        "created_at": edges[0]["created_at"],
        "weight": calculate_edge_weight(edges)  # Опционально
    }
    ↓
[7. Upsert to Graph DB]
    ↓
    await knowledge_graph_inst.upsert_edge(
        source_entity,
        target_entity,
        edge_data=edge_data
    )
    ↓
Output: edge_data
```

### Step 4: Vector DB Update

```python
if relationships_vdb is not None and edge_data:
    # Подготовка данных для vector DB
    edge_id = compute_mdhash_id(
        f"{source_entity}-{target_entity}",
        prefix="rel-"
    )

    data_for_vdb = {
        edge_id: {
            "source_entity": edge_data["source_entity"],
            "target_entity": edge_data["target_entity"],
            "keywords": edge_data["keywords"],
            "content": (
                f"{edge_data['source_entity']} -> {edge_data['target_entity']}\n"
                f"Keywords: {edge_data['keywords']}\n"
                f"{edge_data['description']}"
            ),
            "source_id": edge_data["source_id"],
            "file_path": edge_data.get("file_path", "unknown_source")
        }
    }

    # Upsert с retry logic
    await safe_vdb_operation_with_exception(
        operation=lambda: relationships_vdb.upsert(data_for_vdb),
        operation_name="relationship_upsert",
        entity_name=f"{source_entity}-{target_entity}",
        max_retries=3,
        retry_delay=0.1
    )
```

**Content Format для Embedding**:
```
{source_entity} -> {target_entity}
Keywords: {keywords}
{description}

Пример:
Tim Cook -> Apple Inc
Keywords: leadership, management, CEO
Tim Cook serves as the Chief Executive Officer of Apple Inc, leading the company's strategic direction and operations since 2011.
```

## LLM Agent: Description Summarization

### Функция

**Расположение**: `lightrag/operate.py:121`

```python
async def _handle_entity_relation_summary(
    description_type: str,  # "Entity" или "Relationship"
    entity_or_relation_name: str,
    description_list: list[str],
    seperator: str,
    global_config: dict,
    llm_response_cache: BaseKVStorage | None = None,
) -> tuple[str, bool]:
    """
    Слияние множественных descriptions в одно summary

    Strategy: Map-Reduce approach для больших списков

    Returns:
        (merged_description, llm_was_used)
    """
```

### Decision Logic

```
Input: list of descriptions
    ↓
[Check 1: Empty List]
    ↓
    if not description_list:
        return ("", False)
    ↓
[Check 2: Single Description]
    ↓
    if len(description_list) == 1:
        return (description_list[0], False)
    ↓
[Check 3: Token Count]
    ↓
    combined_text = seperator.join(description_list)
    total_tokens = len(tokenizer.encode(combined_text))

    if total_tokens < summary_context_size AND
       len(description_list) < force_llm_summary_on_merge:
        # Просто объединяем без LLM
        return (combined_text, False)
    ↓
[Check 4: Fits in One LLM Call]
    ↓
    if total_tokens < summary_max_tokens:
        # Одиночный LLM call для summary
        return await _single_summary_call(...)
    ↓
[Map-Reduce Strategy]
    ↓
    # Разбиение на chunks
    # Summary каждого chunk
    # Рекурсивный вызов для summaries
    return await _map_reduce_summary(...)
```

### Configuration Parameters

```python
# Из global_config
summary_context_size = global_config["summary_context_size"]
# Default: 4096 tokens
# Описание: max tokens для prompt + descriptions

summary_max_tokens = global_config["summary_max_tokens"]
# Default: 500 tokens
# Описание: max tokens в output summary

force_llm_summary_on_merge = global_config["force_llm_summary_on_merge"]
# Default: 3
# Описание: минимум descriptions для LLM summarization
```

### Single Summary Call

```python
async def _single_summary_call(
    description_list: list[str],
    entity_or_relation_name: str,
    description_type: str
) -> str:
    """Одиночный LLM call для summarization"""

    # Форматирование descriptions как JSON lines
    descriptions_json = "\n".join(
        json.dumps({"description": desc}, ensure_ascii=False)
        for desc in description_list
    )

    # Промпт
    system_prompt = PROMPTS["summarize_entity_descriptions"].format(
        description_type=description_type,
        description_name=entity_or_relation_name,
        language=global_config["addon_params"].get("language", "English"),
        summary_length=summary_max_tokens
    )

    user_prompt = f"""---Description List---
{descriptions_json}

---Task---
Synthesize these descriptions into a single, comprehensive summary."""

    # LLM call с кешированием
    summary, timestamp = await use_llm_func_with_cache(
        user_prompt,
        use_llm_func,
        system_prompt=system_prompt,
        llm_response_cache=llm_response_cache,
        cache_type="summary",
        chunk_id=entity_or_relation_name
    )

    return summary
```

### Map-Reduce Strategy

```python
async def _map_reduce_summary(
    description_list: list[str],
    entity_or_relation_name: str
) -> str:
    """
    Map-Reduce стратегия для больших списков

    1. MAP: Разбить descriptions на chunks по token limit
    2. MAP: Summary каждого chunk
    3. REDUCE: Рекурсивно объединить summaries
    """

    # === MAP PHASE ===
    chunks = []
    current_chunk = []
    current_tokens = 0

    for desc in description_list:
        desc_tokens = len(tokenizer.encode(desc))

        if current_tokens + desc_tokens > summary_context_size:
            # Завершаем текущий chunk
            chunks.append(current_chunk)
            current_chunk = [desc]
            current_tokens = desc_tokens
        else:
            current_chunk.append(desc)
            current_tokens += desc_tokens

    if current_chunk:
        chunks.append(current_chunk)

    # Summary каждого chunk параллельно
    chunk_summaries = await asyncio.gather(*[
        _single_summary_call(chunk, entity_or_relation_name, description_type)
        for chunk in chunks
    ])

    # === REDUCE PHASE ===
    # Рекурсивный вызов для объединения summaries
    if len(chunk_summaries) == 1:
        return chunk_summaries[0]
    else:
        return await _handle_entity_relation_summary(
            description_type,
            entity_or_relation_name,
            chunk_summaries,  # Рекурсия!
            seperator,
            global_config,
            llm_response_cache
        )
```

### Summary Prompt

**Источник**: `lightrag/prompt.py:174`

```
---Role---
You are a Knowledge Graph Specialist, proficient in data curation and synthesis.

---Task---
Your task is to synthesize a list of descriptions of a given entity or relation
into a single, comprehensive, and cohesive summary.

---Instructions---
1. Input Format: JSON objects, one per line
2. Output Format: Plain text, multiple paragraphs
3. Comprehensiveness: Integrate ALL key information
4. Context: Third-person perspective, explicit naming
5. Conflict Handling:
   - If multiple distinct entities share same name: separate summaries
   - If historical discrepancies: reconcile or present both viewpoints
6. Length Constraint: Must not exceed {summary_length} tokens
7. Language: Output in {language}

---Input---
{description_type} Name: {description_name}

Description List:
{descriptions_json}

---Output---
```

### Example Summarization

**Input Descriptions**:
```json
{"description": "Apple Inc is a technology company."}
{"description": "Apple Inc designs and manufactures consumer electronics."}
{"description": "Apple Inc is known for iPhone, iPad, and Mac computers."}
{"description": "Apple Inc was founded by Steve Jobs, Steve Wozniak, and Ronald Wayne in 1976."}
```

**LLM Summary Output**:
```
Apple Inc is a technology company that designs, develops, and manufactures
consumer electronics, computer software, and online services. Founded in 1976
by Steve Jobs, Steve Wozniak, and Ronald Wayne, the company has become a
global leader in innovation. Apple Inc is particularly known for its flagship
products including iPhone, iPad, and Mac computers, which have revolutionized
their respective markets and established new standards for user experience
and design.
```

## Storage Finalization

### Function

**Расположение**: `lightrag/lightrag.py` в `_insert_done()`

```python
async def _insert_done(self):
    """Финализация после успешной обработки документа"""

    # Persist all storage backends
    tasks = []

    if self.full_entities_storage is not None:
        tasks.append(self.full_entities_storage.persist())

    if self.full_relations_storage is not None:
        tasks.append(self.full_relations_storage.persist())

    if self.knowledge_graph_inst is not None:
        tasks.append(self.knowledge_graph_inst.persist())

    if self.entity_vdb is not None:
        tasks.append(self.entity_vdb.persist())

    if self.relationships_vdb is not None:
        tasks.append(self.relationships_vdb.persist())

    if self.chunks_vdb is not None:
        tasks.append(self.chunks_vdb.persist())

    # Параллельная финализация всех storage backends
    await asyncio.gather(*tasks)

    logger.info("All storage backends persisted successfully")
```

## Consistency Guarantees

### 1. Keyed Locks

```python
# Гарантии:
# ✅ Только один process обрабатывает entity одновременно
# ✅ Только один process обрабатывает edge одновременно
# ✅ Оба endpoints edge locked во время обработки relationship

async with get_storage_keyed_lock([entity_name], namespace="GraphDB"):
    # Атомарная операция:
    # 1. Read existing node
    # 2. Merge descriptions
    # 3. Update node
    # Никто другой не может изменить node в это время
```

### 2. Two-Phase Commit

```python
# Phase 1: Entities
# - Все entities обработаны
# - Graph DB и Vector DB синхронизированы

# Phase 2: Relationships
# - Все relationships обработаны
# - Гарантия что оба endpoints существуют
# - Graph DB и Vector DB синхронизированы
```

### 3. Retry Logic

```python
async def safe_vdb_operation_with_exception(
    operation: callable,
    operation_name: str,
    entity_name: str,
    max_retries: int = 3,
    retry_delay: float = 0.1
):
    """
    Безопасная операция с Vector DB с retry logic

    Стратегия: Exponential backoff
    """
    for attempt in range(max_retries):
        try:
            await operation()
            return  # Success
        except Exception as e:
            if attempt == max_retries - 1:
                # Последняя попытка - raise exception
                raise Exception(
                    f"{operation_name} failed for {entity_name} "
                    f"after {max_retries} attempts: {e}"
                )

            # Exponential backoff
            await asyncio.sleep(retry_delay * (2 ** attempt))
```

## Performance Optimizations

### 1. Concurrent Processing

```python
# Entities обрабатываются параллельно
# Ограничение: graph_max_async (default: 8)

semaphore = asyncio.Semaphore(graph_max_async)

tasks = [
    _process_entity(entity)
    for entity in all_entities
]

results = await asyncio.gather(*tasks)
```

### 2. Batch Storage Operations

```python
# Группировка updates для batch upsert
batch_entities = {}

for entity_data in processed_entities:
    entity_id = compute_mdhash_id(entity_data["entity_name"])
    batch_entities[entity_id] = entity_data

# Одна операция вместо N операций
await entity_vdb.upsert(batch_entities)
```

### 3. Lazy Description Merging

```python
# Проверка необходимости LLM summarization
if len(description_list) < force_llm_summary_on_merge:
    # Простое объединение без LLM
    return "\n\n".join(description_list), False

# LLM вызов только когда необходимо
return await _llm_summary(...), True
```

## Monitoring & Debugging

### Progress Logging

```python
log_message = (
    f"Phase 1: Processing {total_entities_count} entities "
    f"from {doc_id} (async: {graph_max_async})"
)
logger.info(log_message)

# Обновление pipeline status
async with pipeline_status_lock:
    pipeline_status["latest_message"] = log_message
    pipeline_status["history_messages"].append(log_message)
```

### Completion Statistics

```python
log_message = (
    f"Completed merging: "
    f"{len(processed_entities)} entities, "
    f"{len(all_added_entities)} extra entities, "
    f"{len(processed_edges)} relations"
)
logger.info(log_message)
```

### Error Tracking

```python
try:
    entity_data = await _merge_nodes_then_upsert(...)
except Exception as e:
    error_msg = f"Critical error in entity processing for `{entity_name}`: {e}"
    logger.error(error_msg)

    # Обновление pipeline status
    async with pipeline_status_lock:
        pipeline_status["latest_message"] = error_msg
        pipeline_status["history_messages"].append(error_msg)

    # Re-raise для прерывания процесса
    raise
```

## Следующий Этап

После построения графа данные сохраняются в **[Storage Architecture](05-storage-architecture.md)** - системе персистентного хранения с поддержкой различных backends.
