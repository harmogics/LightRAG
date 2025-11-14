# Vector Algorithms: Векторные Алгоритмы Семантического Поиска

## Обзор

Документация векторных алгоритмов, применяемых в LightRAG для continuous semantic space navigation, similarity search, и hybrid retrieval.

## Векторное Пространство vs Графовое Пространство

LightRAG функционирует в **dual-space architecture** (spec/research/03-dual-space-architecture.md):

```
┌─────────────────────────────────────────────┐
│         VECTOR SPACE (Continuous)           │
│                                             │
│  ℝ^n embeddings: [0.23, -0.56, 0.12, ...]  │
│  Cosine similarity: 0.87                    │
│  Soft boundaries, fuzzy matching            │
└──────────────┬──────────────────────────────┘
               │ Entity Resolution
               │ (Bridge between spaces)
               ↓
┌─────────────────────────────────────────────┐
│         GRAPH SPACE (Discrete)              │
│                                             │
│  Nodes: {Apple Inc, iPhone, Tim Cook}      │
│  Edges: (Apple, produces, iPhone)          │
│  Hard boundaries, exact matching            │
└─────────────────────────────────────────────┘
```

## Категории Алгоритмов

### 1. Embedding Generation (Генерация Векторов)

**Файл**: [01-embedding-generation.md](01-embedding-generation.md)

Преобразование текста в dense vector representations:
- Embedding models (OpenAI, sentence-transformers, etc.)
- Batch embedding для производительности
- Dimensionality (768, 1536, 4096 dims)
- Caching embeddings

**Роль**: Трансформация discrete text → continuous semantic space.

### 2. Cosine Similarity Search (Семантический Поиск)

**Файл**: [02-cosine-similarity.md](02-cosine-similarity.md)

Core similarity metric в vector space:
- Cosine similarity calculation
- Top-K retrieval
- Threshold filtering
- Distance vs Similarity metrics

**Роль**: Измерение **semantic proximity** между query и candidates.

### 3. Vector Storage Operations (Векторные БД)

**Файл**: [03-vector-storage.md](03-vector-storage.md)

CRUD операции в векторных БД:
- Upsert (insert/update embeddings)
- Query (top-k similarity search)
- Delete (remove vectors)
- Compression (Float16 + zlib + Base64)

**Роль**: Persistent storage для embeddings с fast similarity search.

### 4. Reranking Algorithms (Переранжирование)

**Файл**: [04-reranking.md](04-reranking.md)

Second-stage refinement после vector search:
- Cross-encoder reranking (Cohere, Jina)
- Relevance scoring
- Top-N selection after reranking

**Роль**: Улучшение precision первичного vector search.

### 5. Hybrid Vector-Graph Selection (Гибридный Отбор)

**Файл**: [05-hybrid-selection.md](05-hybrid-selection.md)

Интеграция vector similarity + graph structure:
- pick_by_vector_similarity: chunk selection via cosine
- pick_by_weighted_polling: graph-based chunk selection
- Hybrid scoring: α * vector_sim + β * graph_importance

**Роль**: Объединение **continuous (vector)** + **discrete (graph)** сигналов.

### 6. Embedding Compression (Сжатие Векторов)

**Файл**: [06-compression.md](06-compression.md)

Storage optimization для embeddings:
- Float32 → Float16 precision reduction
- zlib compression
- Base64 encoding for JSON storage
- Decompression on retrieval

**Роль**: Reducing storage footprint (~4x compression).

---

## Связь с Research Concepts

| Алгоритм | Research Concept | Описание |
|----------|------------------|----------|
| **Embedding Generation** | [Dual-Space Architecture](../research/03-dual-space-architecture.md) | Проекция text → vector space |
| **Cosine Similarity** | [Query as Semantic Key](../research/01-query-as-semantic-key.md) | Query vector = key для semantic navigation |
| **Top-K Retrieval** | [Star-Attractor Pattern](../research/02-star-attractor-pattern.md) | Активация top-K attractors по similarity |
| **Hybrid Selection** | [Semantic Traversal Methods](../research/04-semantic-traversal-methods.md) | Vector + graph signals для Local/Global modes |
| **Reranking** | [Concept-Manifestation Bridge](../research/05-concept-manifestation-bridge.md) | Refinement chunks → best manifestations |

## Связь с Graph Algorithms

| Vector Algorithm | Graph Algorithm | Interaction |
|------------------|-----------------|-------------|
| **Top-K Vector Search** | [BFS Traversal](../graph/01-bfs-traversal.md) | Vector search → seed entities → BFS expansion |
| **Cosine Similarity** | [Degree Operations](../graph/05-degree-operations.md) | Hybrid ranking: α*similarity + β*degree |
| **pick_by_vector_similarity** | [Community Detection](../graph/02-community-detection.md) | Select chunks within community via similarity |
| **Reranking** | [Centrality Algorithms](../graph/03-centrality-algorithms.md) | Rerank by PageRank + cosine similarity |

---

## Связь с Dependencies

| Algorithm | Library | Function |
|-----------|---------|----------|
| **Embedding** | [OpenAI SDK](../dependencies/01-llm-integration.md) | `openai.embeddings.create()` |
| **Cosine Similarity** | [NumPy](../dependencies/05-data-processing.md) | `np.dot()`, `np.linalg.norm()` |
| **Vector Storage** | [nano-vectordb](../dependencies/02-vector-embedding.md) | Default in-memory vector DB |
| **Vector Storage** | [Milvus/Qdrant/FAISS](../dependencies/04-storage-backends.md) | Production vector DBs |
| **Reranking** | [Cohere/Jina APIs](../dependencies/01-llm-integration.md) | Cross-encoder reranking |
| **Compression** | [NumPy](../dependencies/05-data-processing.md) | `np.float16`, `zlib.compress()` |

---

## Архитектурный Паттерн: Vector → Graph Integration

```
┌─────────────────────────────────────────────────────┐
│              USER QUERY                             │
└────────────────────┬────────────────────────────────┘
                     │
                     ↓ [Embedding]
              ┌──────────────┐
              │ Query Vector │ ∈ ℝ^n
              └──────┬───────┘
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
     [Entities] [Relations] [Chunks]
     Vector DB  Vector DB   Vector DB
          │          │          │
          ↓ Cosine Similarity (Top-K)
     ┌─────────────────────────────┐
     │ Top-K Candidates            │
     │ • Entities: 20 results      │
     │ • Relations: 10 results     │
     │ • Chunks: 50 results        │
     └─────────┬───────────────────┘
               │
               ↓ [Optional Reranking]
          ┌─────────────┐
          │ Refined     │
          │ Top-N       │
          └──────┬──────┘
                 │
                 ↓ [Hybrid: Vector + Graph]
          ┌──────────────────────────┐
          │ Graph Operations         │
          │ • BFS from top entities  │
          │ • Community detection    │
          │ • Degree filtering       │
          └──────┬───────────────────┘
                 │
                 ↓ [Final Ranking]
          ┌──────────────────────────┐
          │ Combined Score:          │
          │ α*cosine + β*pagerank +  │
          │ γ*degree + δ*rerank      │
          └──────┬───────────────────┘
                 │
                 ↓
            FINAL CONTEXT
                 │
                 ↓ [LLM]
              ANSWER
```

**Философия**:
1. **Vector space** для initial retrieval (soft matching, semantic similarity)
2. **Graph space** для structural refinement (relations, communities)
3. **Hybrid scoring** для optimal precision + recall

---

## Производительность

### Benchmark (10K entities, 768-dim embeddings)

| Operation | Time | Scalability |
|-----------|------|-------------|
| **Embedding (batch 32)** | 200ms | O(batch_size) |
| **Cosine Similarity** | <1ms | O(1) per pair |
| **Top-K Vector Search** | 50ms | O(N) linear scan, O(log N) with HNSW |
| **Reranking (20 docs)** | 150ms | O(N) cross-encoder |
| **Hybrid Selection** | 100ms | O(entities * chunks) |

### Storage Efficiency

| Method | Size per Vector | Compression Ratio |
|--------|----------------|-------------------|
| **Float32 (raw)** | 3072 bytes (768*4) | 1.0x |
| **Float16 (quantized)** | 1536 bytes | 2.0x |
| **Float16 + zlib** | ~400 bytes | 7.5x |
| **Float16 + zlib + Base64** | ~533 bytes (JSON) | 5.8x |

**Recommendation**: Use Float16 + zlib для in-memory/JSON storage (default: nano-vectordb).

---

## Query Modes и Vector Operations

### Naive Mode

```python
# Pure vector search (no graph)
chunks = await chunks_vdb.query(query, top_k=50)
# → Direct semantic matching
```

### Local Mode

```python
# Vector search entities → BFS
entities = await entities_vdb.query(query, top_k=20)  # ← VECTOR
subgraph = await graph.bfs(entities, depth=2)         # ← GRAPH

# Hybrid chunk selection
chunks = await pick_by_vector_similarity(             # ← VECTOR
    query, entity_chunks, chunks_vdb
)
```

### Global Mode

```python
# Vector search relations → Community detection
relations = await relations_vdb.query(query, top_k=10)  # ← VECTOR
communities = detect_communities(subgraph)               # ← GRAPH

# Reranking chunks
chunks = await rerank(query, candidate_chunks)          # ← VECTOR refinement
```

### Hybrid Mode

```python
# Combine all signals
vector_score = cosine_similarity(query_emb, entity_emb)  # ← VECTOR
graph_score = pagerank(entity) * degree(entity)          # ← GRAPH

combined_score = 0.6 * vector_score + 0.4 * graph_score
```

---

## Example: Full Vector Pipeline

```python
# Step 1: Generate embeddings
query = "What products does Apple make?"
query_embedding = await embedding_func([query])  # ℝ^768

# Step 2: Vector search (top-k)
entity_results = await entities_vdb.query(
    query,
    top_k=20,
    query_embedding=query_embedding[0]
)
# Results sorted by cosine similarity (highest first)

# Step 3: Optional reranking
if enable_rerank:
    entity_results = await rerank(
        query=query,
        documents=[r["content"] for r in entity_results],
        top_n=10
    )

# Step 4: Graph expansion
seed_entities = [r["id"] for r in entity_results[:5]]
subgraph = await graph.bfs(seed_entities, depth=2)

# Step 5: Hybrid chunk selection
entity_chunks = extract_chunks_from_entities(seed_entities)
selected_chunks = await pick_by_vector_similarity(
    query=query,
    chunks=entity_chunks,
    chunks_vdb=chunks_vdb,
    num_of_chunks=30,
    query_embedding=query_embedding[0]
)

# Step 6: Final ranking (hybrid score)
for chunk in selected_chunks:
    chunk["vector_score"] = cosine_similarity(
        query_embedding[0],
        chunk["embedding"]
    )
    chunk["graph_score"] = get_entity_pagerank(chunk["entity"])
    chunk["final_score"] = (
        0.6 * chunk["vector_score"] +
        0.4 * chunk["graph_score"]
    )

final_chunks = sorted(
    selected_chunks,
    key=lambda x: x["final_score"],
    reverse=True
)[:20]
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
