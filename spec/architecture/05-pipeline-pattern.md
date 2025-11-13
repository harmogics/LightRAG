# Pipeline Pattern

## Overview

**Pattern Type**: Behavioral  
**Category**: Data Flow Pattern  
**Purpose**: Последовательная обработка данных через серию трансформаций

---

## Problem

Document processing и query execution требуют множества последовательных шагов:
- **Indexing**: Chunk → Extract → Glean → Parse → Merge → Vectorize → Store
- **Query**: Extract Keywords → Resolve → Assemble → Generate → Format

**Challenges**:
- Управление сложными data flows
- Обработка ошибок на каждом этапе
- Мониторинг прогресса
- Checkpoint/resume capability

---

## Solution: Pipeline Pattern

### Indexing Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│              Document Indexing Pipeline                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Input: Document(s)                                          │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 1: Enqueue                                 │        │
│  │ • Validate documents                             │        │
│  │ • Generate doc IDs                               │        │
│  │ • Store in full_docs storage                     │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 2: Chunking (T1)                           │        │
│  │ • Split document by tokens                       │        │
│  │ • Create overlap regions                         │        │
│  │ • Generate chunk IDs                             │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 3: Entity Extraction (T2)                  │        │
│  │ • Parallel processing per chunk                  │        │
│  │ • LLM call: Extract entities + relations         │        │
│  │ • Output: Raw entities list                      │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 4: Gleaning (T3) [Optional]                │        │
│  │ • Multi-turn LLM for missed entities             │        │
│  │ • Merge with initial extraction                  │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 5: Parsing (T4)                            │        │
│  │ • Validate JSON format                           │        │
│  │ • Convert to Python objects                      │        │
│  │ • Type checking                                  │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 6: Graph Construction & Merging (T5)       │        │
│  │ • Aggregate descriptions per entity              │        │
│  │ • Map-reduce summarization                       │        │
│  │ • Build graph edges                              │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 7: Vectorization (T6)                      │        │
│  │ • Embed entity descriptions                      │        │
│  │ • Embed chunk content                            │        │
│  │ • Store in vector DB                             │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 8: Storage Commit                          │        │
│  │ • Save all storages                              │        │
│  │ • Update status                                  │        │
│  │ • Clean up temp data                             │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  Output: Knowledge Graph + Vector Index                      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Query Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                 Query Execution Pipeline                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Input: User Query + QueryParam                              │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 1: Mode Selection                          │        │
│  │ • Naive / Local / Global / Hybrid                │        │
│  │ • Determine pipeline variant                     │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 2: Keyword Extraction (T7)                 │        │
│  │ • LLM call: Extract high/low level keywords      │        │
│  │ • JSON output parsing                            │        │
│  │ [Skipped in Naive mode]                          │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 3: Entity Resolution (T8)                  │        │
│  │ • Vector search in entity_vdb                    │        │
│  │ • Top-K entity candidates                        │        │
│  │ [Skipped in Naive mode]                          │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 4: Context Assembly (T9)                   │        │
│  │ • Graph traversal (if Local/Global)              │        │
│  │ • Chunk retrieval from vector DB                 │        │
│  │ • Context formatting (JSON)                      │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 5: Answer Generation (T10)                 │        │
│  │ • LLM call with context                          │        │
│  │ • Grounded generation                            │        │
│  │ • Citation extraction                            │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ STAGE 6: Answer Formatting (T11)                 │        │
│  │ • Structure answer                               │        │
│  │ • Add metadata                                   │        │
│  │ • Format references                              │        │
│  └─────────────────────────────────────────────────┘        │
│     ↓                                                         │
│  Output: QueryResult (answer + context + references)         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Pipeline Orchestration

```python
# From lightrag/lightrag.py and operate.py

async def process_document_pipeline(
    document: str,
    doc_id: str,
    split_by_character: str | None = None,
    split_by_character_only: bool = False
) -> None:
    """
    Full document processing pipeline
    """

    # STAGE 1: Chunking
    chunks = chunking_func(
        tokenizer=tokenizer,
        content=document,
        split_by_character=split_by_character,
        split_by_character_only=split_by_character_only,
        overlap_token_size=chunk_overlap_token_size,
        max_token_size=chunk_token_size
    )

    # Generate chunk IDs
    chunk_data = {
        compute_mdhash_id(chunk["content"], prefix="chunk-"): {
            **chunk,
            "full_doc_id": doc_id
        }
        for chunk in chunks
    }

    # STAGE 2-5: Entity Extraction + Gleaning + Parsing + Merging
    entities, relations = await extract_entities(
        chunk_data,
        global_config,
        llm_response_cache,
        text_chunks_storage
    )

    # STAGE 6: Graph Construction
    await merge_nodes_and_edges(
        entities,
        relations,
        knowledge_graph_inst,
        entities_vdb,
        relationships_vdb,
        text_chunks_storage,
        global_config
    )

    # STAGE 7: Vectorization (happens in merge_nodes_and_edges)

    # STAGE 8: Commit
    await knowledge_graph_inst.index_done_callback()
    await entities_vdb.index_done_callback()
    await text_chunks_storage.index_done_callback()
```

### Query Pipeline

```python
async def query_pipeline(
    query: str,
    param: QueryParam
) -> QueryResult:
    """
    Query execution pipeline with mode variants
    """

    # STAGE 1: Mode selection (already in param.mode)

    if param.mode == "naive":
        # Naive pipeline: chunks only
        return await naive_query(query, param, ...)

    else:
        # Local/Global/Hybrid pipeline

        # STAGE 2: Keywords
        keywords = await extract_keywords(query, global_config)

        # STAGE 3: Entity resolution
        entities = await resolve_entities(
            keywords["high_level_keywords"],
            entity_vdb,
            top_k=param.top_k
        )

        # STAGE 4: Context assembly
        context = await assemble_context(
            entities,
            knowledge_graph_inst,
            chunks_vdb,
            param
        )

        # STAGE 5: Answer generation
        answer = await generate_answer(
            query,
            context,
            llm_func,
            param
        )

        # STAGE 6: Formatting
        result = format_query_result(answer, context, param)

        return result
```

---

## Pipeline Variants

### Naive Mode Pipeline

```python
# Simplified pipeline
Query
  → Chunk Vector Search (T8 variant)
  → Context Assembly (chunks only, T9 variant)
  → Answer Generation (T10 with P7)
  → Format (T11)
```

**Characteristics**:
- Fastest (2-5s)
- No keyword extraction
- No entity resolution
- Chunk-based only

### Local Mode Pipeline

```python
# Entity-focused pipeline
Query
  → Keywords (T7)
  → Entity Resolution (T8)
  → 1-2 hop graph traversal (T9)
  → Answer Generation (T10 with P6)
  → Format (T11)
```

**Characteristics**:
- Medium speed (4-9s)
- Entity neighborhood
- Relationship-aware

### Global Mode Pipeline

```python
# Comprehensive pipeline
Query
  → Keywords (T7)
  → Entity Resolution (T8)
  → 3+ hop traversal + communities (T9)
  → Answer Generation (T10 with P6)
  → Format (T11)
```

**Characteristics**:
- Slower (7-15s)
- Global graph coverage
- Multi-hop reasoning

---

## Pipeline Features

### 1. Checkpoint Support

```python
# Indexing с checkpoints
async def apipeline_process_enqueue_documents(
    self,
    split_by_character: str | None = None,
    split_by_character_only: bool = False
):
    """Process enqueued documents with checkpoint"""

    while True:
        # Get pending documents
        pending = await self.full_docs.filter_keys([
            k for k, v in self.full_docs.items()
            if v.get("status") == "pending"
        ])

        if not pending:
            break

        for doc_id in pending:
            try:
                # Process document through pipeline
                await self._process_single_document(doc_id, ...)

                # Update status: pending → done
                await self.full_docs.upsert({
                    doc_id: {"status": "done"}
                })

            except Exception as e:
                # Update status: pending → failed
                await self.full_docs.upsert({
                    doc_id: {"status": "failed", "error": str(e)}
                })
```

### 2. Parallel Stage Processing

```python
# Parallel chunk processing in extraction stage
async def extract_entities(chunks, ...):
    """Process chunks in parallel"""

    semaphore = asyncio.Semaphore(max_async)

    async def process_with_semaphore(chunk):
        async with semaphore:
            return await extract_single_chunk(chunk)

    # All chunks processed in parallel (limited by semaphore)
    tasks = [process_with_semaphore(c) for c in chunks]
    results = await asyncio.gather(*tasks)

    return merge_results(results)
```

### 3. Error Handling per Stage

```python
# Graceful error handling
async def process_stage(data, stage_name):
    try:
        result = await stage_func(data)
        logger.info(f"Stage {stage_name} completed")
        return result

    except LLMError as e:
        logger.error(f"Stage {stage_name} LLM error: {e}")
        # Fallback or retry
        return await fallback_func(data)

    except Exception as e:
        logger.error(f"Stage {stage_name} failed: {e}")
        raise PipelineError(f"{stage_name} failed") from e
```

### 4. Progress Tracking

```python
# Pipeline status tracking
pipeline_status = {
    "latest_message": "",
    "history_messages": [],
    "processed_chunks": 0,
    "total_chunks": 100,
    "current_stage": "extraction"
}

# Update during pipeline
async with pipeline_status_lock:
    pipeline_status["processed_chunks"] += 1
    pipeline_status["latest_message"] = f"Processed {pipeline_status['processed_chunks']}/{pipeline_status['total_chunks']}"
```

---

## Benefits

### 1. Modularity

Each stage is independent and testable:
```python
# Test individual stages
@pytest.mark.asyncio
async def test_chunking_stage():
    chunks = chunking_stage(document)
    assert len(chunks) > 0

@pytest.mark.asyncio
async def test_extraction_stage():
    entities = await extraction_stage(chunks)
    assert len(entities) > 0
```

### 2. Extensibility

Easy to add new stages:
```python
# Add custom enrichment stage
async def enrichment_stage(entities):
    """Custom entity enrichment"""
    enriched = []
    for entity in entities:
        entity["custom_score"] = compute_score(entity)
        enriched.append(entity)
    return enriched

# Insert in pipeline
entities = await extraction_stage(chunks)
entities = await enrichment_stage(entities)  # NEW STAGE
await merge_stage(entities)
```

### 3. Observable

Clear visibility into data flow:
```python
# Log intermediate results
logger.debug(f"After chunking: {len(chunks)} chunks")
logger.debug(f"After extraction: {len(entities)} entities")
logger.debug(f"After merging: {len(unique_entities)} unique entities")
```

---

## Related Patterns

- **Map-Reduce Pattern**: Used in extraction stage (parallel processing)
- **Strategy Pattern**: Different modes = different pipeline configurations
- **Chain of Responsibility**: Each stage handles specific transformation

---

## See Also

- [Map-Reduce Pattern](10-map-reduce-pattern.md)
- [Async Pattern](07-async-pattern.md)
- [Agent Pattern](06-agent-pattern.md)

