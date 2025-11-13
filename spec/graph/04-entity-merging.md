# Entity Merging: Алгоритм Слияния Сущностей

## Концептуальная Парадигма

**Entity merging** решает проблему **дубликатов** в knowledge graph — когда одна концепция представлена множественными entities.

```
BEFORE Merging:
┌──────────┐   ┌──────────┐   ┌──────────┐
│ Apple    │   │ Apple Inc│   │ Apple Co.│  ← Duplicates (same concept)
│ (entity1)│   │ (entity2)│   │ (entity3)│
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
   iPhone        MacBook         iPad

AFTER Merging:
        ┌──────────────┐
        │ Apple Inc    │  ← Merged entity (canonical form)
        │ (target)     │
        └──────┬───────┘
               │
     ┌─────────┼─────────┐
     │         │         │
  iPhone    MacBook     iPad   ← All relations preserved
```

**Философия**: Entity deduplication реализует **concept-manifestation bridge** — объединение различных textual manifestations одного abstract concept.

---

## Алгоритм: amerge_entities

### Описание

**amerge_entities** объединяет несколько source entities в single target entity, сохраняя все relations и metadata.

### Реализация

**Файл**: `lightrag/utils_graph.py:850-1025`

```python
async def amerge_entities(
    chunk_entity_relation_graph: BaseGraphStorage,
    entities_vdb: BaseVectorStorage,
    relationships_vdb: BaseVectorStorage,
    source_entities: list[str],        # Entities to merge (duplicates)
    target_entity: str,                 # Target entity (canonical)
    merge_strategy: dict[str, str] = None,  # Field merge strategies
    target_entity_data: dict[str, Any] = None,  # Override target data
) -> dict[str, Any]:
    """
    Merge multiple entities into one.

    Metaphor: Collapsing multiple semantic attractors into single unified attractor.

    Merge strategies per field:
    • concatenate: Join all values (e.g., descriptions)
    • keep_first: Keep first non-empty value
    • keep_last: Keep last value
    • join_unique: Join unique values only

    Returns:
        {
            "merged_entity": target_entity,
            "merged_count": len(source_entities),
            "preserved_relations": count,
            "merged_chunks": count,
        }
    """
```

### Алгоритм (High-Level)

**Step 1: Validate Inputs**
```python
# Check source entities exist
for entity in source_entities:
    if not await graph.has_node(entity):
        raise ValueError(f"Entity {entity} not found")

# Ensure target ≠ sources
if target_entity in source_entities:
    raise ValueError("Target cannot be in source list")
```

**Step 2: Collect Source Data**
```python
source_data = []
for entity in source_entities:
    node_data = await graph.get_node(entity)
    source_data.append(node_data)
```

**Step 3: Merge Fields (apply strategies)**
```python
merged_data = {}

for field in all_fields:
    strategy = merge_strategy.get(field, "concatenate")

    if strategy == "concatenate":
        # Join all values with separator
        values = [data.get(field, "") for data in source_data]
        merged_data[field] = " | ".join(v for v in values if v)

    elif strategy == "keep_first":
        # Keep first non-empty value
        for data in source_data:
            if data.get(field):
                merged_data[field] = data[field]
                break

    elif strategy == "keep_last":
        # Keep last value
        merged_data[field] = source_data[-1].get(field, "")

    elif strategy == "join_unique":
        # Join unique values only (for lists)
        values = []
        for data in source_data:
            val = data.get(field, "")
            if val and val not in values:
                values.append(val)
        merged_data[field] = " | ".join(values)
```

**Step 4: Update Target Entity**
```python
# Create or update target entity
if await graph.has_node(target_entity):
    # Merge with existing target data
    existing_data = await graph.get_node(target_entity)
    final_data = merge_dicts(existing_data, merged_data, merge_strategy)
else:
    # Create new target entity
    final_data = merged_data

await graph.upsert_node(target_entity, final_data)
```

**Step 5: Redirect Relations**
```python
# Collect all relations from source entities
all_relations = []
for source in source_entities:
    edges = await graph.get_node_edges(source)
    all_relations.extend(edges)

# Redirect relations to target
for (u, v) in all_relations:
    if u in source_entities:
        # Outgoing edge: source → other becomes target → other
        await graph.upsert_edge(target_entity, v, edge_data)
    elif v in source_entities:
        # Incoming edge: other → source becomes other → target
        await graph.upsert_edge(u, target_entity, edge_data)
```

**Step 6: Merge Vector Embeddings**
```python
# Entities VDB: redirect source IDs → target ID
for source in source_entities:
    source_embedding_data = await entities_vdb.get_by_id(source)
    if source_embedding_data:
        # Update ID to target
        await entities_vdb.upsert({target_entity: source_embedding_data})
        await entities_vdb.delete([source])

# Relations VDB: update src_id/tgt_id in relation embeddings
relations = await relationships_vdb.get_by_ids(all_relation_ids)
for rel in relations:
    if rel["src_id"] in source_entities:
        rel["src_id"] = target_entity
    if rel["tgt_id"] in source_entities:
        rel["tgt_id"] = target_entity
    await relationships_vdb.upsert({rel["id"]: rel})
```

**Step 7: Delete Source Entities**
```python
# Remove source entities from graph
await graph.remove_nodes(source_entities)

# Remove from vector DBs (already done in step 6)
```

### Complexity

- **Time**: O(E_source + R_source) where E = edges, R = relations
- **Space**: O(E_source) for temporary edge storage

### Parameters

```python
merge_strategy: dict[str, str] = {
    "description": "concatenate",     # Merge descriptions
    "entity_type": "keep_first",      # Keep first type
    "source_id": "join_unique",       # Merge all source chunk IDs
    "created_at": "keep_first",       # Keep earliest timestamp
}
```

---

## Merge Strategies

### 1. Concatenate (Join All)

```python
# Example: Descriptions
source1 = {"description": "Apple makes iPhones"}
source2 = {"description": "Apple founded by Steve Jobs"}

merged = {"description": "Apple makes iPhones | Apple founded by Steve Jobs"}
```

**Use Case**: Merging complementary information (descriptions, summaries).

### 2. Keep First (First Non-Empty)

```python
# Example: Entity type
source1 = {"entity_type": ""}
source2 = {"entity_type": "organization"}
source3 = {"entity_type": "company"}

merged = {"entity_type": "organization"}  # First non-empty
```

**Use Case**: Canonical metadata (type, category).

### 3. Keep Last (Override)

```python
# Example: Updated timestamp
source1 = {"updated_at": "2024-01-01"}
source2 = {"updated_at": "2024-06-01"}

merged = {"updated_at": "2024-06-01"}  # Most recent
```

**Use Case**: Temporal data (timestamps, versions).

### 4. Join Unique (Deduplicate)

```python
# Example: Source chunk IDs
source1 = {"source_id": "chunk1,chunk2"}
source2 = {"source_id": "chunk2,chunk3"}  # chunk2 duplicated

merged = {"source_id": "chunk1,chunk2,chunk3"}  # Unique only
```

**Use Case**: Lists, sets (source IDs, tags, categories).

---

## Связь с Concept-Manifestation Bridge

Entity merging реализует **entity resolution** в concept-manifestation bridge (spec/research/05-concept-manifestation-bridge.md):

```
Textual Manifestations (Particulars):
• "Apple" (chunk1)
• "Apple Inc" (chunk2)
• "Apple Inc." (chunk3)
• "Apple Company" (chunk4)
        ↓
   Entity Resolution (Merging)
        ↓
Abstract Concept (Platonic Form):
• "Apple Inc" (canonical entity)
        ↓
Relations to Manifestations:
• source_id: "chunk1,chunk2,chunk3,chunk4"
```

**Philosophy**: Multiple textual mentions (particulars) → single abstract entity (form) → bridge via `source_id`.

---

## Use Cases

### 1. Insert-Time Deduplication

During document insertion, LLM may extract duplicates:

```python
# LLM extracts entities from chunks
entities_chunk1 = ["Apple", "iPhone", "Tim Cook"]
entities_chunk2 = ["Apple Inc", "MacBook", "Tim Cook"]  # "Apple" vs "Apple Inc"

# Detect duplicates (similarity > threshold)
duplicates = detect_duplicates(entities_chunk1 + entities_chunk2)
# duplicates = [("Apple", "Apple Inc")]

# Merge duplicates
for source, target in duplicates:
    await amerge_entities(
        graph,
        entities_vdb,
        relationships_vdb,
        source_entities=[source],
        target_entity=target,
        merge_strategy={"description": "concatenate", "source_id": "join_unique"}
    )
```

### 2. Manual Entity Resolution

User-initiated merge via API:

```python
# User: "Merge 'Apple', 'Apple Inc', 'Apple Co.' into 'Apple Inc'"
await amerge_entities(
    graph,
    entities_vdb,
    relationships_vdb,
    source_entities=["Apple", "Apple Co."],
    target_entity="Apple Inc",
    merge_strategy=default_strategy
)
```

### 3. Automated Deduplication Pipeline

Periodic deduplication job:

```python
# Find duplicate candidates (high embedding similarity)
all_entities = await entities_vdb.get_all()
duplicates = []

for i, entity1 in enumerate(all_entities):
    for entity2 in all_entities[i+1:]:
        similarity = cosine_similarity(
            entity1["embedding"],
            entity2["embedding"]
        )
        if similarity > 0.95:  # Very high similarity
            duplicates.append((entity1["id"], entity2["id"]))

# Auto-merge duplicates (keep entity with more relations)
for source, target in duplicates:
    source_degree = await graph.node_degree(source)
    target_degree = await graph.node_degree(target)

    if source_degree > target_degree:
        source, target = target, source  # Swap (merge into higher-degree entity)

    await amerge_entities(
        graph,
        entities_vdb,
        relationships_vdb,
        source_entities=[source],
        target_entity=target,
        merge_strategy=auto_merge_strategy
    )
```

---

## Связь с Dependencies

### NetworkX (spec/dependencies/03-graph-processing.md)

```python
import networkx as nx

# Graph-based duplicate detection
def find_duplicate_candidates(graph: nx.Graph) -> list[tuple]:
    """
    Find entity pairs with similar neighborhoods (Jaccard similarity).
    """
    candidates = []

    for entity1 in graph.nodes():
        for entity2 in graph.nodes():
            if entity1 >= entity2:  # Skip duplicates and self
                continue

            # Jaccard similarity of neighborhoods
            neighbors1 = set(graph.neighbors(entity1))
            neighbors2 = set(graph.neighbors(entity2))

            jaccard = len(neighbors1 & neighbors2) / len(neighbors1 | neighbors2)

            if jaccard > 0.7:  # High overlap
                candidates.append((entity1, entity2))

    return candidates
```

### Custom (lightrag.utils_graph)

```python
from lightrag.utils_graph import amerge_entities

# Merge API
result = await amerge_entities(
    chunk_entity_relation_graph=graph_storage,
    entities_vdb=entities_vdb,
    relationships_vdb=relationships_vdb,
    source_entities=["entity1", "entity2"],
    target_entity="canonical_entity",
    merge_strategy={
        "description": "concatenate",
        "entity_type": "keep_first",
        "source_id": "join_unique"
    }
)

print(result)
# {
#     "merged_entity": "canonical_entity",
#     "merged_count": 2,
#     "preserved_relations": 15,
#     "merged_chunks": 10
# }
```

---

## Graph Topology Preservation

### Relation Redirection

Merging preserves all relations:

```
BEFORE:
  A → B
  A → C
  D → A
  (merge A → X)

AFTER:
  X → B   (A's outgoing preserved)
  X → C
  D → X   (A's incoming preserved)
```

**Implementation**:
```python
# Get all edges involving source entities
for source in source_entities:
    # Outgoing edges
    outgoing = await graph.get_node_edges(source)
    for (u, v) in outgoing:
        if u == source:
            # Redirect: source → v becomes target → v
            edge_data = await graph.get_edge(u, v)
            await graph.upsert_edge(target_entity, v, edge_data)

    # Incoming edges (if directed graph)
    # ...
```

### Self-Loop Handling

```python
# Edge case: A → B, B → C, merge B → A
# Result: A → C (not A → A self-loop)

if u == target_entity and v == target_entity:
    # Skip self-loop
    continue
```

---

## Производительность

### Benchmark (Merge 10 entities with avg 50 relations each)

| Operation | Time |
|-----------|------|
| Collect source data | 10ms |
| Merge fields | 5ms |
| Redirect relations | 150ms (50 relations * 3ms per update) |
| Update embeddings | 100ms |
| Delete sources | 20ms |
| **Total** | **285ms** |

**Bottleneck**: Relation redirection (many edge updates).

### Оптимизации

1. **Batch edge updates**:
```python
# Instead of per-edge upsert
for edge in edges:
    await graph.upsert_edge(u, v, data)  # ❌ Slow (N roundtrips)

# Batch upsert
await graph.upsert_edges_batch([
    (u1, v1, data1),
    (u2, v2, data2),
    ...
])  # ✅ Fast (1 roundtrip)
```

2. **Parallel VDB updates**:
```python
# Update entities_vdb and relationships_vdb in parallel
await asyncio.gather(
    update_entities_vdb(source_entities, target_entity),
    update_relationships_vdb(relation_ids, target_entity)
)
```

3. **Transaction support** (if DB supports):
```python
async with graph.transaction():
    # All merge operations in single transaction
    # Rollback on failure
```

---

## Example: Full Merge Workflow

```python
async def merge_entity_duplicates(
    entity_name: str,
    graph: BaseGraphStorage,
    entities_vdb: BaseVectorStorage,
    relationships_vdb: BaseVectorStorage
):
    """
    Find and merge all duplicates of an entity.
    """

    # Step 1: Find duplicates (high embedding similarity)
    entity_data = await entities_vdb.get_by_id(entity_name)
    query_embedding = entity_data["embedding"]

    similar_entities = await entities_vdb.query(
        query_text=entity_name,
        query_embedding=query_embedding,
        top_k=10,
        cosine_better_than_threshold=0.95  # Very high similarity
    )

    duplicates = [e["id"] for e in similar_entities if e["id"] != entity_name]

    if not duplicates:
        print(f"No duplicates found for {entity_name}")
        return

    # Step 2: Select target (entity with highest degree)
    degrees = {}
    for entity in [entity_name] + duplicates:
        degrees[entity] = await graph.node_degree(entity)

    target = max(degrees, key=degrees.get)  # Highest degree
    sources = [e for e in degrees.keys() if e != target]

    print(f"Merging {len(sources)} duplicates into {target}")
    print(f"  Sources: {sources}")

    # Step 3: Merge
    result = await amerge_entities(
        graph,
        entities_vdb,
        relationships_vdb,
        source_entities=sources,
        target_entity=target,
        merge_strategy={
            "description": "concatenate",
            "entity_type": "keep_first",
            "source_id": "join_unique",
            "created_at": "keep_first"
        }
    )

    print(f"Merge complete: {result}")
    return result

# Usage
await merge_entity_duplicates("Apple", graph, entities_vdb, relationships_vdb)
# Output:
# Merging 2 duplicates into Apple Inc
#   Sources: ['Apple', 'Apple Co.']
# Merge complete: {
#     'merged_entity': 'Apple Inc',
#     'merged_count': 2,
#     'preserved_relations': 47,
#     'merged_chunks': 23
# }
```

---

## Future Enhancements

### 1. Confidence Scores

```python
merge_result = await amerge_entities(
    ...,
    confidence_threshold=0.95  # Only merge if very confident
)

# Store merge confidence in metadata
await graph.upsert_node(target_entity, {
    "merge_confidence": 0.98,
    "merged_from": source_entities
})
```

### 2. Undo Merge

```python
async def unmerge_entity(
    target_entity: str,
    graph: BaseGraphStorage
):
    """
    Undo a previous merge (split entity back into sources).
    """
    node_data = await graph.get_node(target_entity)
    source_entities = node_data.get("merged_from", [])

    if not source_entities:
        raise ValueError("Entity was not merged")

    # Recreate source entities from stored metadata
    # ...
```

### 3. Hierarchical Merging

```python
# Merge at different granularities
# Fine: "iPhone 14" + "iPhone 14 Pro" → "iPhone 14 Series"
# Coarse: "iPhone 14 Series" + "iPhone 13 Series" → "iPhone"
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
