# Transform Chains: Цепочки Преобразований

## Обзор

Transform Chains описывают последовательности и зависимости между семантическими преобразованиями в LightRAG. Понимание chains критично для оптимизации pipeline и troubleshooting.

## Indexing Chain

### Complete Flow

```
┌───────────────────────────────────────────────────────────────────┐
│                    INDEXING TRANSFORM CHAIN                       │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Document (Raw)                                                   │
│      │ 100% info                                                  │
│      ▼                                                            │
│  [T1: CHUNKING] ────────────────────────────────────────┐        │
│      │ 98% preserved                                     │        │
│      ▼                                                   │        │
│  Text Chunks (1-1000)                                    │        │
│      │                                                   │        │
│      ├─────► [Parallel Processing] ─────────────────────┤        │
│      │                                                   │        │
│      ▼ (per chunk)                                       │        │
│  [T2: ENTITY EXTRACTION] ──┐                             │        │
│      │ 60% entities         │                            │        │
│      ▼                      │                            │        │
│  Raw Entities               │                            │        │
│      │                      │                            │        │
│      ▼ (conditional)        │                            │        │
│  [T3: GLEANING] ──────────┤                             │        │
│      │ +15% coverage       │                            │        │
│      ▼                     │                            │        │
│  Refined Entities          │                            │        │
│      │                     │                            │        │
│      ▼                     │                            │        │
│  [T4: PARSING] ──────────┤                             │        │
│      │ 95% structured     │                            │        │
│      ▼                    │                            │        │
│  Structured Entities      │                            │        │
│      │                    │                            │        │
│      └─────► Aggregate ───┴────────────────────────────┘        │
│                 │                                                 │
│                 ▼                                                 │
│  All Entities + Relationships                                     │
│      │                                                            │
│      ▼ (per entity/relation)                                     │
│  [T5: DESCRIPTION MERGING] ────────────────────────────┐        │
│      │ 90% fidelity                                    │        │
│      ▼                                                  │        │
│  Merged Descriptions                                    │        │
│      │                                                  │        │
│      ├────► [Parallel] ────────────────────────────────┤        │
│      │                                                  │        │
│      ▼                                                  │        │
│  [T6: VECTORIZATION] ─────────────────────────────────┤        │
│      │ 100% semantic                                   │        │
│      ▼                                                  │        │
│  Knowledge Graph + Vector Index                                  │
└───────────────────────────────────────────────────────────────────┘
```

### Dependencies Matrix

| Transform | Depends On | Enables | Can Parallelize |
|-----------|-----------|---------|-----------------|
| **T1** | None | T2 | No (sequential) |
| **T2** | T1 | T3, T4 | Yes (per chunk) |
| **T3** | T2 | T4 | Yes (per chunk) |
| **T4** | T2, T3 | T5 | Yes (per chunk) |
| **T5** | T4 | T6 | Yes (per entity) |
| **T6** | T5 | Query | Yes (batch) |

### Parallel Execution

```python
# T2-T4: Per-chunk parallel processing
async def process_chunks_parallel(chunks):
    tasks = []
    for chunk in chunks:
        task = asyncio.create_task(
            process_single_chunk(chunk)  # T2 → T3 → T4
        )
        tasks.append(task)

    results = await asyncio.gather(*tasks)
    return aggregate_results(results)

# T5: Per-entity parallel merging
async def merge_entities_parallel(entities):
    tasks = []
    for entity_name, descriptions in entities.items():
        task = asyncio.create_task(
            merge_descriptions(entity_name, descriptions)  # T5
        )
        tasks.append(task)

    return await asyncio.gather(*tasks)
```

### Information Flow Metrics

| Stage | Input Info | Output Info | Net Change |
|-------|-----------|-------------|------------|
| **Document** | 100% | - | Baseline |
| **After T1** | 100% | 98% | -2% (boundaries) |
| **After T2** | 98% | 60% | -38% (extraction) |
| **After T3** | 60% | 75% | +15% (gleaning) |
| **After T4** | 75% | 72% | -3% (validation) |
| **After T5** | 72% | 65% | -7% (compression) |
| **After T6** | 65% | 65% | 0% (encoding) |

---

## Query Chains

### Naive Mode Chain

```
┌─────────────────────────────────────────────────────────┐
│              NAIVE MODE CHAIN (Minimal)                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  User Query                                             │
│      │                                                  │
│      ▼                                                  │
│  [Vector Search: Chunks] ────────── Direct             │
│      │ 50-100ms                                        │
│      ▼                                                  │
│  Top-K Chunks                                           │
│      │                                                  │
│      ▼                                                  │
│  [T10: ANSWER GENERATION]                              │
│      │ 2-5s                                            │
│      ▼                                                  │
│  Raw Answer                                             │
│      │                                                  │
│      ▼                                                  │
│  [T11: FORMATTING]                                      │
│      │ 10-50ms                                         │
│      ▼                                                  │
│  Structured Answer                                      │
│                                                         │
│  Total: ~2-5s                                           │
└─────────────────────────────────────────────────────────┘
```

### Local Mode Chain

```
┌──────────────────────────────────────────────────────────────┐
│              LOCAL MODE CHAIN (Entity-Centric)               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  User Query                                                  │
│      │                                                       │
│      ▼                                                       │
│  [T7: KEYWORD EXTRACTION]                                   │
│      │ 1-3s                                                 │
│      ▼                                                       │
│  Keywords (high-level)                                       │
│      │                                                       │
│      ▼                                                       │
│  [T8: ENTITY RESOLUTION]                                    │
│      │ 50-150ms                                            │
│      ▼                                                       │
│  Seed Entities (5-10)                                        │
│      │                                                       │
│      ├──► Graph Traversal (1-2 hops) ──┐                   │
│      │    100-300ms                     │                   │
│      │                                  │                   │
│      └──► Chunk Retrieval ─────────────┤                   │
│           50-100ms                      │                   │
│                                         │                   │
│                                         ▼                   │
│  [T9: CONTEXT ASSEMBLY]                                     │
│      │ 100-500ms                                           │
│      ▼                                                       │
│  Structured Context (entities + relations + chunks)         │
│      │                                                       │
│      ▼                                                       │
│  [T10: ANSWER GENERATION]                                   │
│      │ 2-5s                                                 │
│      ▼                                                       │
│  Raw Answer                                                  │
│      │                                                       │
│      ▼                                                       │
│  [T11: FORMATTING]                                           │
│      │ 10-50ms                                             │
│      ▼                                                       │
│  Structured Answer + Explainability                         │
│                                                              │
│  Total: ~4-9s                                                │
└──────────────────────────────────────────────────────────────┘
```

### Global Mode Chain

```
┌──────────────────────────────────────────────────────────────────┐
│              GLOBAL MODE CHAIN (Comprehensive)                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  User Query                                                      │
│      │                                                           │
│      ▼                                                           │
│  [T7: KEYWORD EXTRACTION] ───── Hierarchical                    │
│      │ 1-3s                     (high + low level)              │
│      ▼                                                           │
│  Keywords (high + low level)                                     │
│      │                                                           │
│      ▼                                                           │
│  [T8: ENTITY RESOLUTION] ───── Extended                         │
│      │ 100-200ms               (more candidates)                │
│      ▼                                                           │
│  Seed Entities (10-15)                                           │
│      │                                                           │
│      ├──► Graph Traversal (3+ hops) ──┐                        │
│      │    500-1000ms                   │                        │
│      │    Max nodes: 200               │                        │
│      │                                 │                        │
│      ├──► Community Detection ─────────┤                        │
│      │    200-500ms                    │                        │
│      │                                 │                        │
│      └──► Comprehensive Chunks ────────┤                        │
│           100-300ms                    │                        │
│                                        │                        │
│                                        ▼                        │
│  [T9: CONTEXT ASSEMBLY] ───── Rich                              │
│      │ 300-800ms                                                │
│      ▼                                                           │
│  Large Structured Context                                        │
│  (50-100 entities, 100-200 relations, 20-30 chunks)            │
│      │                                                           │
│      ▼                                                           │
│  [T10: ANSWER GENERATION] ───── Comprehensive                   │
│      │ 4-10s                                                    │
│      ▼                                                           │
│  Detailed Raw Answer                                             │
│      │                                                           │
│      ▼                                                           │
│  [T11: FORMATTING] ───── Rich Metadata                          │
│      │ 20-80ms                                                  │
│      ▼                                                           │
│  Comprehensive Structured Answer                                 │
│  + Full Explainability                                          │
│  + Multiple Reasoning Paths                                     │
│                                                                  │
│  Total: ~7-15s                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## Chain Comparison Table

| Aspect | Naive Chain | Local Chain | Global Chain |
|--------|------------|-------------|--------------|
| **Transforms** | 3 (Search, T10, T11) | 5 (T7-T11) | 5 (T7-T11 extended) |
| **LLM Calls** | 1 | 2 | 3-5 |
| **Graph Ops** | 0 | 1-2 hops | 3+ hops + communities |
| **Latency** | 2-5s | 4-9s | 7-15s |
| **Context Size** | 10-20KB | 20-50KB | 50-100KB |
| **Entities** | 0 | 5-15 | 50-100 |
| **Precision** | 0.65-0.75 | 0.75-0.85 | 0.80-0.90 |
| **Use Case** | Simple facts | Entity queries | Complex reasoning |

## Critical Paths

### Indexing Critical Path

```
Document → T1 → T2 → T4 → T5 → T6 → Storage

Critical: T2 (slowest), T5 (blocks finalization)
Bottleneck: LLM calls in T2 and T5
Optimization: Parallelize per-chunk and per-entity
```

### Query Critical Path (Local Mode)

```
Query → T7 → T8 → T9 → T10 → T11 → Answer

Critical: T7 (first LLM), T10 (second LLM)
Bottleneck: Sequential LLM calls
Optimization: Cache T7 results, optimize prompts
```

## Optimization Strategies

### 1. Caching

```python
# Cache at transform boundaries
cache_points = {
    "T1": "chunk_cache",      # Cache chunking results
    "T2": "extraction_cache",  # Cache LLM extractions
    "T5": "summary_cache",     # Cache summaries
    "T6": "embedding_cache",   # Cache embeddings
    "T7": "keyword_cache",     # Cache keywords
    "T10": "answer_cache"      # Cache answers
}
```

### 2. Parallelization

```python
# Maximize parallel execution
parallel_opportunities = {
    "Indexing": [
        "T2-T4 per chunk",
        "T5 per entity",
        "T6 batch embeddings"
    ],
    "Query": [
        "T8 multiple keywords",
        "T9 subgraph + chunks"
    ]
}
```

### 3. Early Termination

```python
# Stop chain early if sufficient confidence
if mode == "naive" and confidence > 0.8:
    return answer  # Skip expensive graph operations
```

---

**Next**: [05-comparison-tables.md](05-comparison-tables.md) - Comprehensive comparison tables
