# Map-Reduce Pattern

## Overview

**Pattern Type**: Behavioral/Semantic  
**Purpose**: Parallel processing with aggregation

---

## Use Cases in LightRAG

### 1. Parallel Chunk Processing (Map)

```python
# Map: Process each chunk independently
async def process_chunk(chunk):
    return await extract_entities(chunk)

tasks = [process_chunk(c) for c in chunks]
results = await asyncio.gather(*tasks)  # Parallel execution

# Reduce: Merge results
all_entities = []
for chunk_entities in results:
    all_entities.extend(chunk_entities)
```

### 2. Description Summarization (Map-Reduce)

```python
# Problem: Entity has 100+ descriptions
descriptions = [...]  # 100 descriptions

# Map: Summarize chunks
chunks = split_into_groups(descriptions, size=10)
partial_summaries = [
    await summarize(chunk) for chunk in chunks
]  # 10 summaries

# Reduce: Summarize summaries
final_summary = await summarize(partial_summaries)
```

---

## Implementation

```python
async def map_reduce_summarization(
    entity_name: str,
    descriptions: list[str],
    max_tokens: int
) -> str:
    """Recursive map-reduce for large description lists"""

    total_tokens = count_tokens(descriptions)

    if total_tokens <= max_tokens:
        # Base case: direct summarization
        return await llm_summarize(entity_name, descriptions)

    # Map phase: split and summarize chunks
    chunks = split_by_token_limit(descriptions, max_tokens)

    summaries = await asyncio.gather(*[
        llm_summarize(entity_name, chunk)
        for chunk in chunks
    ])

    # Reduce phase: recursively summarize
    return await map_reduce_summarization(
        entity_name,
        summaries,
        max_tokens
    )
```

---

## Benefits

- **Scalability**: Process 1000+ chunks in parallel
- **Efficiency**: Optimal resource utilization
- **Modularity**: Map and reduce are independent

---

## See Also

- [Async Pattern](07-async-pattern.md)
- [Pipeline Pattern](05-pipeline-pattern.md)

