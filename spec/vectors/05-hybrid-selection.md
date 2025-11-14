# Hybrid Selection: Гибридный Отбор Vector + Graph

## Концептуальная Парадигма

**Hybrid selection** объединяет **vector space** (semantic similarity) + **graph space** (structural importance) для optimal chunk selection.

```
Vector Signal (Continuous)     Graph Signal (Discrete)
     │                              │
     │ Cosine similarity            │ Degree, PageRank
     ↓                              ↓
┌─────────────┐              ┌─────────────┐
│ Top chunks  │              │ Hub entity  │
│ by vector   │              │ chunks      │
└──────┬──────┘              └──────┬──────┘
       │                            │
       └──────────┬─────────────────┘
                  ↓
          ┌──────────────┐
          │ Hybrid Score │
          │ α*vector +   │
          │ β*graph      │
          └──────────────┘
```

**Философия**: Vector finds **semantically relevant**, graph finds **structurally important** — combine for best results.

---

## Algorithm 1: pick_by_vector_similarity

### Описание

Select chunks from entity-related chunks using **cosine similarity** to query.

### Реализация

**Файл**: `lightrag/utils.py:2096`

```python
async def pick_by_vector_similarity(
    query: str,
    text_chunks_storage: "BaseKVStorage",
    chunks_vdb: "BaseVectorStorage",
    num_of_chunks: int,
    entity_info: list[dict[str, Any]],
    embedding_func: callable,
    query_embedding=None,
) -> list[str]:
    """
    Vector similarity-based chunk selection.

    Process:
    1. Collect all chunk IDs from entities
    2. Compute query embedding (if not provided)
    3. Retrieve chunk embeddings from vector DB
    4. Calculate cosine similarity for each chunk
    5. Sort by similarity (descending)
    6. Select top-N chunks

    Args:
        query: User query
        chunks_vdb: Vector storage for chunks
        num_of_chunks: Number to select
        entity_info: List of entities with their chunks
        query_embedding: Optional pre-computed query embedding

    Returns:
        List of top-N chunk IDs by similarity
    """
    if not entity_info or num_of_chunks <= 0:
        return []

    # Step 1: Collect chunk IDs from entities
    all_chunk_ids = set()
    for entity in entity_info:
        chunk_ids = entity.get("sorted_chunks", [])
        all_chunk_ids.update(chunk_ids)

    if not all_chunk_ids:
        return []

    all_chunk_ids = list(all_chunk_ids)

    # Step 2: Get query embedding
    if query_embedding is None:
        query_embedding = await embedding_func([query])
        query_embedding = query_embedding[0]

    # Step 3: Retrieve chunk embeddings
    chunk_vectors = await chunks_vdb.get_vectors_by_ids(all_chunk_ids)

    if not chunk_vectors:
        return []

    # Step 4: Calculate cosine similarities
    similarities = []
    for chunk_id in all_chunk_ids:
        if chunk_id in chunk_vectors:
            chunk_embedding = chunk_vectors[chunk_id]
            similarity = cosine_similarity(query_embedding, chunk_embedding)
            similarities.append((chunk_id, similarity))

    # Step 5: Sort by similarity (highest first)
    similarities.sort(key=lambda x: x[1], reverse=True)

    # Step 6: Select top-N
    selected_chunks = [chunk_id for chunk_id, _ in similarities[:num_of_chunks]]

    logger.debug(
        f"Selected {len(selected_chunks)} chunks from {len(all_chunk_ids)} candidates by vector similarity"
    )

    return selected_chunks
```

**Use Case**: Local/hybrid query modes where entities found via vector search, then chunks selected by similarity.

---

## Algorithm 2: pick_by_weighted_polling

### Описание

Select chunks based on **graph-based weighting** (entity frequency, chunk occurrence).

### Реализация

**Файл**: `lightrag/utils.py:2016`

```python
def pick_by_weighted_polling(
    entities_with_chunks: list[dict[str, Any]],
    max_related_chunks: int,
    min_related_chunks: int = 1,
) -> list[str]:
    """
    Weighted polling chunk selection based on entity importance.

    Algorithm:
    1. Build chunk occurrence map (how many entities mention each chunk)
    2. Apply linear gradient weights (entities ranked by similarity)
    3. Round-robin selection with weighted polling
    4. Deduplication

    Metaphor: "Democratic voting" where important entities get more votes.

    Args:
        entities_with_chunks: Entities with their chunk lists
        max_related_chunks: Maximum chunks to select
        min_related_chunks: Minimum chunks per entity

    Returns:
        List of selected chunk IDs
    """
    if not entities_with_chunks:
        return []

    # Step 1: Linear gradient weights (first entity = highest weight)
    num_entities = len(entities_with_chunks)
    weights = [num_entities - i for i in range(num_entities)]

    # Step 2: Weighted round-robin selection
    selected_chunks = []
    chunk_seen = set()

    # Each entity contributes proportionally to its weight
    for entity, weight in zip(entities_with_chunks, weights):
        chunks = entity.get("sorted_chunks", [])

        # Number of chunks to take from this entity (weighted)
        num_to_take = max(
            min_related_chunks,
            int(weight / sum(weights) * max_related_chunks)
        )

        for chunk_id in chunks[:num_to_take]:
            if chunk_id not in chunk_seen:
                selected_chunks.append(chunk_id)
                chunk_seen.add(chunk_id)

            if len(selected_chunks) >= max_related_chunks:
                break

        if len(selected_chunks) >= max_related_chunks:
            break

    logger.info(
        f"Selected {len(selected_chunks)} chunks by weighted polling from {num_entities} entities"
    )

    return selected_chunks
```

**Use Case**: When vector similarity unavailable (e.g., no rerank, no chunk embeddings) — fallback to graph-based selection.

---

## Hybrid Scoring Formula

### Combined Score

```python
def hybrid_chunk_score(
    chunk_id: str,
    query_embedding: np.ndarray,
    chunk_embedding: np.ndarray,
    entity_degree: int,
    entity_pagerank: float,
    alpha: float = 0.6,  # Vector weight
    beta: float = 0.3,   # Graph degree weight
    gamma: float = 0.1,  # PageRank weight
) -> float:
    """
    Hybrid scoring: Vector + Graph signals.

    Score = α * cosine_similarity +
            β * (degree / max_degree) +
            γ * pagerank

    Args:
        alpha: Weight for vector similarity (0-1)
        beta: Weight for graph degree (0-1)
        gamma: Weight for PageRank (0-1)
        (Note: α + β + γ should = 1.0)

    Returns:
        Combined score (0-1 range)
    """
    # Vector signal
    vector_score = cosine_similarity(query_embedding, chunk_embedding)

    # Graph signals (normalized)
    degree_score = min(entity_degree / 100, 1.0)  # Normalize to [0,1]
    pagerank_score = entity_pagerank  # Already [0,1]

    # Combined score
    combined = (
        alpha * vector_score +
        beta * degree_score +
        gamma * pagerank_score
    )

    return combined
```

---

## Связь with Query Modes

### Local Mode

```python
# Vector search entities → Graph expansion → Vector chunk selection
entities = await entities_vdb.query(query, top_k=20)  # ← VECTOR
subgraph = await graph.bfs(entities, depth=2)          # ← GRAPH
chunks = await pick_by_vector_similarity(              # ← VECTOR
    query, entity_chunks, chunks_vdb
)
```

### Hybrid Mode

```python
# Vector + Graph combined scoring
vector_score = cosine_similarity(query_emb, chunk_emb)  # ← VECTOR
degree_score = entity_degree / max_degree               # ← GRAPH
combined = 0.7 * vector_score + 0.3 * degree_score

# Sort by combined score
chunks.sort(key=lambda c: combined_score(c), reverse=True)
```

---

## Comparison: Vector vs Weighted vs Hybrid

| Method | Criterion | Pros | Cons |
|--------|-----------|------|------|
| **pick_by_vector_similarity** | Cosine similarity | ✅ Semantic relevance<br>✅ Query-specific | ❌ Requires embeddings<br>❌ Slower (embedding retrieval) |
| **pick_by_weighted_polling** | Entity rank + frequency | ✅ Fast (no embeddings)<br>✅ Graph structure | ❌ No semantic matching<br>❌ May miss relevant chunks |
| **Hybrid Scoring** | Vector + Graph combined | ✅ Best of both<br>✅ Balanced | ❌ More complex<br>❌ Requires tuning weights |

**Recommendation**:
- **Vector similarity**: Default (best precision)
- **Weighted polling**: Fallback when embeddings unavailable
- **Hybrid**: Critical queries requiring both semantic + structural signals

---

## Example: Full Hybrid Selection

```python
# Query: "Apple products"
query_embedding = await embedding_func([query])

# Step 1: Vector search entities
entities = await entities_vdb.query(query, top_k=20)
# [Apple Inc (0.95), iPhone (0.89), iPad (0.82), ...]

# Step 2: Collect entity chunks
entity_chunks = []
for entity in entities:
    chunks = await get_entity_chunks(entity["id"])
    entity_chunks.append({
        "entity_id": entity["id"],
        "sorted_chunks": chunks,
        "similarity": entity["distance"],  # Vector score
        "degree": await graph.node_degree(entity["id"]),  # Graph score
    })

# Step 3: Hybrid chunk selection
selected_chunks = []

for entity in entity_chunks[:10]:  # Top-10 entities
    chunk_ids = entity["sorted_chunks"][:5]  # Top-5 chunks per entity

    for chunk_id in chunk_ids:
        # Get chunk embedding
        chunk_vectors = await chunks_vdb.get_vectors_by_ids([chunk_id])
        chunk_emb = chunk_vectors[chunk_id]

        # Hybrid score
        vector_score = cosine_similarity(query_embedding[0], chunk_emb)
        graph_score = entity["degree"] / 100

        combined_score = 0.7 * vector_score + 0.3 * graph_score

        selected_chunks.append({
            "chunk_id": chunk_id,
            "entity": entity["entity_id"],
            "vector_score": vector_score,
            "graph_score": graph_score,
            "combined_score": combined_score
        })

# Step 4: Sort by combined score
selected_chunks.sort(key=lambda x: x["combined_score"], reverse=True)

# Step 5: Take top-30
final_chunks = selected_chunks[:30]

print(f"Selected {len(final_chunks)} chunks")
for chunk in final_chunks[:5]:
    print(f"  Entity: {chunk['entity']}, Combined: {chunk['combined_score']:.3f}")
    print(f"    (Vector: {chunk['vector_score']:.3f}, Graph: {chunk['graph_score']:.3f})")

# Output:
# Selected 30 chunks
#   Entity: Apple Inc, Combined: 0.885
#     (Vector: 0.92, Graph: 0.85)
#   Entity: iPhone, Combined: 0.831
#     (Vector: 0.87, Graph: 0.79)
#   ...
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
