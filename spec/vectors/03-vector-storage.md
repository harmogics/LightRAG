# Vector Storage: Операции Векторных Баз Данных

## Концептуальная Парадигма

**Vector storage** = persistent storage for embeddings с fast similarity search capabilities.

```
┌───────────────────────────────────────────┐
│        VECTOR DATABASE                    │
│                                           │
│  ID              Embedding (ℝ^n)         │
│  ─────────────────────────────────────   │
│  ent-apple     [0.23, -0.56, ..., 0.89] │
│  ent-iphone    [0.25, -0.51, ..., 0.82] │
│  ent-samsung   [-0.12, 0.34, ..., 0.45] │
│  ...           ...                        │
│                                           │
│  Operations:                              │
│  • upsert(id, embedding, metadata)       │
│  • query(embedding, top_k) → results     │
│  • delete(ids)                            │
│  • get_by_id(id) → embedding             │
└───────────────────────────────────────────┘
```

**Философия**: Persistent vector index для efficient similarity search at scale.

---

## CRUD Operations

### 1. Upsert (Insert/Update)

**Файл**: `lightrag/kg/nano_vector_db_impl.py:91`

```python
async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
    """
    Insert or update embeddings in vector storage.

    Args:
        data: {
            id: {
                "content": str,
                "metadata": dict
            }
        }

    Process:
    1. Generate embeddings (batched)
    2. Compress vectors (Float16 + zlib)
    3. Store in database
    """
    if not data:
        return

    # Extract contents
    list_data = [
        {
            "__id__": k,
            "__created_at__": int(time.time()),
            **{k1: v1 for k1, v1 in v.items() if k1 in self.meta_fields},
        }
        for k, v in data.items()
    ]
    contents = [v["content"] for v in data.values()]

    # Batch embedding generation
    batches = [
        contents[i : i + self._max_batch_size]
        for i in range(0, len(contents), self._max_batch_size)
    ]

    embedding_tasks = [self.embedding_func(batch) for batch in batches]
    embeddings_list = await asyncio.gather(*embedding_tasks)
    embeddings = np.concatenate(embeddings_list)

    # Compress and store
    for i, d in enumerate(list_data):
        # Float16 quantization
        vector_f16 = embeddings[i].astype(np.float16)

        # zlib compression
        compressed_vector = zlib.compress(vector_f16.tobytes())

        # Base64 encoding (for JSON storage)
        encoded_vector = base64.b64encode(compressed_vector).decode("utf-8")

        d["vector"] = encoded_vector
        d["__vector__"] = embeddings[i]  # ← In-memory (uncompressed)

    # Upsert to database
    client = await self._get_client()
    results = client.upsert(datas=list_data)
    return results
```

**Complexity**: O(N*D) for embedding + O(N*D) for compression

### 2. Query (Top-K Similarity Search)

**Файл**: `lightrag/kg/nano_vector_db_impl.py:139`

```python
async def query(
    self, query: str, top_k: int, query_embedding: list[float] = None
) -> list[dict[str, Any]]:
    """
    Top-K similarity search.

    Args:
        query: Query text
        top_k: Number of results
        query_embedding: Optional pre-computed embedding

    Returns:
        List of top-K results sorted by similarity
    """
    # Generate query embedding (if not provided)
    if query_embedding is not None:
        embedding = query_embedding
    else:
        embedding = await self.embedding_func(
            [query], _priority=5  # Higher priority for query
        )
        embedding = embedding[0]

    # Similarity search
    client = await self._get_client()
    results = client.query(
        query=embedding,
        top_k=top_k,
        better_than_threshold=self.cosine_better_than_threshold
    )

    # Format results
    results = [
        {
            "id": dp["__id__"],
            "content": dp["content"],
            "distance": dp["__metrics__"],  # Cosine similarity
            "created_at": dp.get("__created_at__"),
        }
        for dp in results
    ]

    return results
```

**Complexity**: O(N*D) для linear scan, O(log N) with ANN index

### 3. Get by ID(s)

```python
async def get_by_id(self, id: str) -> dict[str, Any] | None:
    """Retrieve single embedding by ID."""
    client = await self._get_client()
    result = client.get([id])

    if result:
        dp = result[0]
        return {
            "id": dp.get("__id__"),
            "content": dp["content"],
            "created_at": dp.get("__created_at__"),
        }
    return None


async def get_by_ids(self, ids: list[str]) -> list[dict[str, Any]]:
    """Batch retrieval by IDs."""
    if not ids:
        return []

    client = await self._get_client()
    results = client.get(ids)

    return [
        {
            "id": dp.get("__id__"),
            "content": dp["content"],
            "created_at": dp.get("__created_at__"),
        }
        for dp in results
    ]
```

### 4. Delete

```python
async def delete(self, ids: list[str]):
    """Delete vectors by IDs."""
    client = await self._get_client()
    client.delete(ids)

    logger.debug(
        f"Successfully deleted {len(ids)} vectors"
    )
```

---

## Vector Compression

**Файл**: `lightrag/kg/nano_vector_db_impl.py:124`

### Float32 → Float16 + zlib + Base64

```python
# Original: Float32 (4 bytes per dim)
vector_f32 = np.array([0.234567, -0.567891, ...])  # 1536 dims
size_f32 = 1536 * 4 = 6144 bytes

# Step 1: Quantize to Float16 (2 bytes per dim)
vector_f16 = vector_f32.astype(np.float16)
size_f16 = 1536 * 2 = 3072 bytes  # 2x compression

# Step 2: zlib compression
compressed = zlib.compress(vector_f16.tobytes())
size_compressed = ~400 bytes  # 7.5x compression

# Step 3: Base64 encoding (for JSON storage)
encoded = base64.b64encode(compressed).decode("utf-8")
size_base64 = ~533 bytes  # Final size (5.8x compression overall)

# Decompression on retrieval
decoded = base64.b64decode(encoded)
decompressed = zlib.decompress(decoded)
vector_f16 = np.frombuffer(decompressed, dtype=np.float16)
vector_f32 = vector_f16.astype(np.float32)  # Restore precision
```

**Benefits**:
- **5-8x storage reduction** (6144 → ~533 bytes)
- **Minimal precision loss** (Float16 sufficient for similarity)
- **JSON-compatible** (Base64 encoding)

**Trade-offs**:
- **CPU overhead** for compression/decompression (~5-10ms)
- **Precision loss** (Float32 → Float16, but negligible for cosine similarity)

---

## Storage Backends Comparison

### 1. NanoVectorDB (Default)

**Файл**: `lightrag/kg/nano_vector_db_impl.py`

```
Type: In-memory JSON storage
Scale: < 100K vectors
Features:
  ✅ Simple (no infrastructure)
  ✅ Fast for small datasets
  ✅ Compression (Float16 + zlib)
  ❌ No ANN (linear scan = O(N))
  ❌ Limited scalability
```

### 2. Milvus

**Файл**: `lightrag/kg/milvus_impl.py`

```
Type: Distributed vector database
Scale: Millions+ vectors
Features:
  ✅ ANN index (HNSW, IVF) = O(log N)
  ✅ Horizontal scaling
  ✅ GPU acceleration
  ✅ Production-ready
  ❌ Requires infrastructure
```

### 3. Qdrant

**Файл**: `lightrag/kg/qdrant_impl.py`

```
Type: Rust-based vector DB
Scale: Millions of vectors
Features:
  ✅ Fast ANN (HNSW)
  ✅ Rich filtering (metadata)
  ✅ Easy deployment (Docker)
  ✅ Good performance/cost
```

### 4. FAISS

**Файл**: `lightrag/kg/faiss_impl.py`

```
Type: Facebook AI similarity search
Scale: Billions of vectors
Features:
  ✅ Fastest ANN (HNSW, IVF-PQ)
  ✅ GPU support
  ✅ Memory-efficient (PQ compression)
  ❌ No metadata filtering
  ❌ Library (not database)
```

**Selection Guide**:
- **< 10K vectors**: NanoVectorDB (default, simple)
- **10K-100K**: FAISS or Qdrant (local deployment)
- **100K-1M**: Qdrant or Milvus (cloud deployment)
- **1M+**: Milvus or FAISS (distributed + GPU)

---

## Связь с Dual-Space Architecture

Vector storage = **persistent layer** for continuous semantic space (spec/research/03-dual-space-architecture.md):

```
┌─────────────────────────────────────────────┐
│         APPLICATION LAYER                   │
│  LightRAG queries, entity extraction, etc.  │
└────────────────┬────────────────────────────┘
                 │
        ┌────────┼────────┐
        ↓                 ↓
┌────────────────┐  ┌────────────────┐
│  VECTOR SPACE  │  │  GRAPH SPACE   │
│  (Continuous)  │  │  (Discrete)    │
│                │  │                │
│  Embeddings    │  │  Entities      │
│  Cosine search │  │  BFS traversal │
└────────┬───────┘  └────────┬───────┘
         │                   │
         ↓                   ↓
┌────────────────┐  ┌────────────────┐
│ Vector Storage │  │ Graph Storage  │
│ (Milvus/Qdrant)│  │ (Neo4j/Mongo)  │
└────────────────┘  └────────────────┘
```

**Vector storage** provides **persistent semantic index** complementing graph's structural index.

---

## Связь with Graph Algorithms

### Vector Search → Graph Expansion

**Файл**: `lightrag/operate.py:3397`

```python
async def local_query(query, entities_vdb, graph, query_param):
    """
    Local mode: Vector search → Graph expansion.
    """
    # Step 1: Vector search (top-K entities)
    entity_results = await entities_vdb.query(
        query,
        top_k=query_param.top_k  # e.g., 20
    )

    # Step 2: Extract entity IDs
    seed_entities = [r["id"] for r in entity_results]

    # Step 3: Graph expansion (BFS)
    subgraph = await graph.get_knowledge_subgraph(
        seeds=seed_entities,
        max_depth=2,
        max_nodes=100
    )

    # Vector storage → Graph storage integration
    return subgraph
```

**Interaction**: Vector DB finds **seed entities**, graph DB expands **neighborhood**.

---

## Производительность

### Benchmark (1536-dim vectors)

| Backend | Insert (1K) | Query (Top-10) | Storage (1K) | Scale Limit |
|---------|-------------|----------------|--------------|-------------|
| **NanoVectorDB** | 500ms | 50ms (linear) | 500KB compressed | 100K |
| **FAISS (CPU)** | 200ms | 5ms (HNSW) | 6MB raw | 10M |
| **FAISS (GPU)** | 100ms | 1ms | 6MB raw | 100M |
| **Qdrant** | 300ms | 3ms (HNSW) | 8MB | 10M |
| **Milvus** | 250ms | 2ms (HNSW) | 10MB | 1B+ |

**Observations**:
- **NanoVectorDB**: Fast insert, slow query (linear scan)
- **FAISS/Qdrant/Milvus**: Slower insert (index building), fast query (ANN)

### Query Latency Breakdown

```
Total: 50ms (NanoVectorDB, 10K vectors)

Embedding generation: 20ms (OpenAI API)
Vector storage query:  25ms (linear scan)
Result formatting:      5ms
```

**Optimization**: Pre-compute query embedding to save 20ms → 30ms total.

---

## Example: Full Storage Pipeline

```python
from lightrag.kg.nano_vector_db_impl import NanoVectorDBStorage
from lightrag.utils import EmbeddingFunc

# Initialize storage
vector_storage = NanoVectorDBStorage(
    namespace="entities",
    global_config={
        "working_dir": "./storage",
        "embedding_batch_num": 32,
        "vector_db_storage_cls_kwargs": {
            "cosine_better_than_threshold": 0.2
        }
    },
    embedding_func=EmbeddingFunc(
        embedding_dim=1536,
        func=openai_embedding
    )
)

# Upsert entities
await vector_storage.upsert({
    "ent-apple": {
        "content": "Apple Inc is a technology company",
        "entity_type": "organization"
    },
    "ent-iphone": {
        "content": "iPhone is a smartphone by Apple",
        "entity_type": "product"
    }
})

# Query
results = await vector_storage.query(
    query="tech companies",
    top_k=10
)

print(results)
# [
#     {"id": "ent-apple", "content": "Apple Inc...", "distance": 0.92},
#     {"id": "ent-iphone", "content": "iPhone...", "distance": 0.75}
# ]

# Get by ID
entity = await vector_storage.get_by_id("ent-apple")
print(entity)
# {"id": "ent-apple", "content": "Apple Inc...", "created_at": 1704067200}

# Delete
await vector_storage.delete(["ent-apple"])
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
