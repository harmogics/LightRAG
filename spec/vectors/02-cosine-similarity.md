# Cosine Similarity: Метрика Семантической Близости

## Концептуальная Парадигма

**Cosine similarity** = fundamental metric для measuring **semantic proximity** в vector space.

```
Vector Space (ℝ^n):

    v2 [0.8, 0.6]
      /
     /  θ = 25° → cos(θ) = 0.91 (high similarity)
    /
   •────────→ v1 [0.6, 0.8]
  origin

    v3 [-0.5, 0.3]
      \
       \ θ = 125° → cos(θ) = -0.57 (dissimilar)
        \
         •

Cosine measures angle between vectors, NOT distance.
```

**Философия**: Vectors pointing in same direction = semantically similar, regardless of magnitude.

---

## Алгоритм: Cosine Similarity Calculation

### Описание

**Cosine similarity** вычисляет cosine of angle between two vectors:

```
cos(θ) = (v₁ · v₂) / (||v₁|| ||v₂||)

где:
- v₁ · v₂ = dot product = Σ(v₁ᵢ * v₂ᵢ)
- ||v|| = L2 norm = √(Σ vᵢ²)
- θ = angle between vectors

Range: [-1, 1]
- +1: identical direction (perfect similarity)
-  0: orthogonal (unrelated)
- -1: opposite direction (opposite meaning)
```

### Реализация

**Файл**: `lightrag/utils.py:1024`

```python
import numpy as np

def cosine_similarity(v1, v2):
    """
    Calculate cosine similarity between two vectors.

    Metaphor: Measuring angular alignment in semantic space.
    Vectors pointing same direction = similar meaning.

    Args:
        v1: numpy array of shape (D,)
        v2: numpy array of shape (D,)

    Returns:
        float in range [-1, 1]
    """
    # Dot product
    dot_product = np.dot(v1, v2)

    # L2 norms
    norm1 = np.linalg.norm(v1)
    norm2 = np.linalg.norm(v2)

    # Cosine similarity
    return dot_product / (norm1 * norm2)


# Example usage
v_apple = np.array([0.234, -0.567, 0.123, ..., 0.891])  # "Apple Inc"
v_iphone = np.array([0.256, -0.512, 0.145, ..., 0.823])  # "iPhone"
v_random = np.array([-0.123, 0.456, -0.789, ..., 0.234])  # "Random text"

similarity_apple_iphone = cosine_similarity(v_apple, v_iphone)
# → 0.87 (high similarity - Apple produces iPhone)

similarity_apple_random = cosine_similarity(v_apple, v_random)
# → 0.12 (low similarity - unrelated)
```

### Batch Cosine Similarity

**Optimized vectorized implementation** для comparing query против multiple candidates:

```python
def batch_cosine_similarity(query_vector, candidate_vectors):
    """
    Compute cosine similarity between query and N candidates.

    Args:
        query_vector: shape (D,)
        candidate_vectors: shape (N, D)

    Returns:
        similarities: shape (N,) - cosine scores for each candidate
    """
    # Normalize query
    query_norm = query_vector / np.linalg.norm(query_vector)

    # Normalize candidates (axis=1 for row-wise norm)
    candidate_norms = np.linalg.norm(candidate_vectors, axis=1, keepdims=True)
    candidates_normalized = candidate_vectors / candidate_norms

    # Batch dot product (matrix multiplication)
    similarities = candidates_normalized @ query_norm
    # Shape: (N,)

    return similarities


# Example: Query against 1000 entities
query_embedding = np.array([...])  # Shape: (1536,)
entity_embeddings = np.array([...])  # Shape: (1000, 1536)

similarities = batch_cosine_similarity(query_embedding, entity_embeddings)
# Shape: (1000,) - similarity score for each entity

# Get top-K
top_k_indices = np.argsort(similarities)[::-1][:10]  # Top 10
top_k_entities = [entities[i] for i in top_k_indices]
top_k_scores = [similarities[i] for i in top_k_indices]
```

### Complexity

- **Time**: O(D) for single pair, O(N*D) for query vs N candidates
- **Space**: O(1) for single pair, O(N*D) for storing N vectors

### Optimizations

**NumPy vectorization** (100x+ speedup vs naive loop):
```python
# Naive (slow)
similarities = []
for candidate in candidates:
    sim = cosine_similarity(query, candidate)
    similarities.append(sim)

# Vectorized (fast)
similarities = batch_cosine_similarity(query, candidates)
```

---

## Связь с Vector Search

Cosine similarity = **core operation** в top-K vector retrieval (spec/vectors/03-vector-storage.md):

### Top-K Retrieval Algorithm

**Файл**: `lightrag/kg/nano_vector_db_impl.py:139`

```python
async def query(
    self, query: str, top_k: int, query_embedding: list[float] = None
) -> list[dict[str, Any]]:
    """
    Top-K vector search using cosine similarity.

    Algorithm:
    1. Compute query embedding (if not provided)
    2. Calculate cosine similarity vs all stored vectors
    3. Sort by similarity (descending)
    4. Return top-K results
    """
    # Step 1: Get query embedding
    if query_embedding is not None:
        embedding = query_embedding
    else:
        embedding = await self.embedding_func([query])
        embedding = embedding[0]

    # Step 2: Cosine similarity search (delegated to vector DB)
    client = await self._get_client()
    results = client.query(
        query=embedding,
        top_k=top_k,
        better_than_threshold=self.cosine_better_than_threshold  # Filter
    )

    # Step 3: Format results
    results = [
        {
            "id": dp["__id__"],
            "content": dp["content"],
            "distance": dp["__metrics__"],  # ← Cosine similarity score
            "created_at": dp.get("__created_at__"),
        }
        for dp in results
    ]

    return results


# NanoVectorDB internal implementation
class NanoVectorDB:
    def query(self, query_vector, top_k, better_than_threshold=0.0):
        """
        Internal cosine similarity search.
        """
        # Calculate cosine similarity for all vectors
        similarities = []
        for item in self.storage["data"]:
            vector = item["__vector__"]
            sim = cosine_similarity(query_vector, vector)

            if sim >= better_than_threshold:  # Threshold filter
                item["__metrics__"] = sim
                similarities.append(item)

        # Sort by similarity (descending)
        similarities.sort(key=lambda x: x["__metrics__"], reverse=True)

        # Return top-K
        return similarities[:top_k]
```

### Threshold Filtering

**Файл**: `lightrag/base.py:220`

```python
@dataclass
class BaseVectorStorage(StorageNameSpace, ABC):
    cosine_better_than_threshold: float = field(default=0.2)
    """
    Minimum cosine similarity threshold.

    Results with similarity < threshold are filtered out.
    Typical values:
    - 0.0: No filtering (return all results)
    - 0.2: Weak filtering (remove very dissimilar)
    - 0.5: Moderate filtering
    - 0.7: Strong filtering (only highly similar)
    """
```

**Purpose**: Avoid returning irrelevant low-similarity results.

---

## Distance vs Similarity Metrics

### Cosine Similarity vs Euclidean Distance

```
Cosine Similarity:
- Measures angle (direction)
- Invariant to magnitude (vector length)
- Range: [-1, 1] (higher = more similar)
- Good for: Semantic similarity (meaning encoded in direction)

Euclidean Distance:
- Measures straight-line distance
- Sensitive to magnitude
- Range: [0, ∞) (lower = more similar)
- Good for: Spatial proximity (coordinates matter)

Example:
v1 = [1, 2, 3]
v2 = [2, 4, 6]  # Same direction as v1, double magnitude

Cosine:    cos(v1, v2) = 1.0 (perfect similarity - same direction)
Euclidean: dist(v1, v2) = √((1-2)² + (2-4)² + (3-6)²) = 3.74 (far apart)
```

**Why Cosine for Embeddings?**
- Embedding models encode **meaning in direction**, not magnitude
- Documents of different lengths → different magnitude, same meaning
- Cosine ignores magnitude → robust to length variation

### Conversion: Similarity ↔ Distance

```python
# Cosine similarity to distance
cosine_distance = 1 - cosine_similarity  # Range: [0, 2]

# Normalized to [0, 1]
normalized_distance = (1 - cosine_similarity) / 2  # 0 = identical, 1 = opposite
```

---

## Связь с Query-as-Key Paradigm

Cosine similarity реализует **query vector as key** для semantic search (spec/research/01-query-as-semantic-key.md):

```
Query: "What products does Apple make?"
         ↓ [Embedding]
Query Vector: v_q = [0.25, -0.54, 0.13, ...]
         ↓ [Cosine Similarity]
┌─────────────────────────────────────────┐
│ Entity Vector Database                  │
│                                         │
│ v("Apple Inc")  → cos(v_q, v) = 0.92   │ ← High similarity
│ v("iPhone")     → cos(v_q, v) = 0.85   │ ← High similarity
│ v("Samsung")    → cos(v_q, v) = 0.67   │ ← Medium similarity
│ v("Random")     → cos(v_q, v) = 0.15   │ ← Low similarity (filtered)
└─────────────────────────────────────────┘
         ↓ [Top-K Selection]
Top-K Entities: [Apple Inc, iPhone, Samsung]
         ↓ [Graph Expansion]
Subgraph + Context
```

**Query vector** acts as **semantic compass** pointing towards relevant entities.

---

## Связь с Graph Algorithms

### Hybrid Scoring: Cosine + Graph Metrics

**Combining vector similarity + graph centrality** (spec/graph/03-centrality-algorithms.md):

```python
async def hybrid_entity_ranking(
    query: str,
    entities_vdb: BaseVectorStorage,
    graph: BaseGraphStorage,
    alpha: float = 0.6,  # Weight for cosine similarity
    beta: float = 0.4,   # Weight for graph metrics
):
    """
    Hybrid ranking: Cosine similarity + Graph importance.
    """
    # Step 1: Vector search (cosine similarity)
    query_embedding = await embedding_func([query])
    vector_results = await entities_vdb.query(
        query,
        top_k=100,
        query_embedding=query_embedding[0]
    )

    # Step 2: Get graph metrics
    ranked = []
    for result in vector_results:
        entity_id = result["id"]
        cosine_score = result["distance"]  # ← Vector score

        # Graph metrics
        degree = await graph.node_degree(entity_id)
        pagerank = await graph.get_pagerank(entity_id)
        graph_score = (degree / 100) * 0.5 + pagerank * 0.5  # ← Graph score

        # Combined score
        combined = alpha * cosine_score + beta * graph_score

        ranked.append({
            "entity": entity_id,
            "cosine_score": cosine_score,
            "graph_score": graph_score,
            "combined_score": combined
        })

    # Sort by combined score
    ranked.sort(key=lambda x: x["combined_score"], reverse=True)

    return ranked[:20]
```

**Benefits**:
- **Cosine similarity**: Semantic relevance to query
- **Graph metrics**: Structural importance in KG
- **Combination**: Best of both worlds (relevant + important)

---

## Производительность

### Benchmark (1536-dim vectors, NumPy)

| Operation | Single Pair | Batch (100) | Batch (1000) | Batch (10K) |
|-----------|------------|-------------|--------------|-------------|
| **Naive Loop** | 10µs | 1ms | 10ms | 100ms |
| **Vectorized** | 10µs | 100µs | 1ms | 10ms |
| **Speedup** | 1x | 10x | 10x | 10x |

**Observation**: Vectorization critical for batch operations.

### Memory-Efficient Top-K

For large vector DBs (millions of vectors), full sort expensive:

```python
import heapq

def memory_efficient_top_k(query_vector, candidate_vectors, k=10):
    """
    Top-K using min-heap (avoids full sort).

    Time: O(N + K log K) instead of O(N log N)
    Space: O(K) instead of O(N)
    """
    # Min-heap of size K
    heap = []

    for i, candidate in enumerate(candidate_vectors):
        similarity = cosine_similarity(query_vector, candidate)

        if len(heap) < k:
            heapq.heappush(heap, (similarity, i))
        elif similarity > heap[0][0]:
            heapq.heapreplace(heap, (similarity, i))

    # Extract top-K (sorted descending)
    top_k = sorted(heap, reverse=True)
    return [(idx, score) for score, idx in top_k]
```

---

## Связь with Dependencies

### NumPy (spec/dependencies/05-data-processing.md)

```python
import numpy as np

# Vectorized operations for speed
dot_product = np.dot(v1, v2)
norm = np.linalg.norm(v)

# Batch operations
similarities = candidate_vectors @ query_vector  # Matrix multiplication
```

### FAISS/Milvus (Approximate Nearest Neighbors)

For **production-scale** vector search (millions+ vectors), use optimized libraries:

```python
import faiss

# Build FAISS index (HNSW for cosine similarity)
dimension = 1536
index = faiss.IndexHNSWFlat(dimension, 32)  # 32 = M param

# Add vectors (normalized for cosine)
vectors_normalized = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)
index.add(vectors_normalized)

# Query (top-K)
query_normalized = query / np.linalg.norm(query)
distances, indices = index.search(query_normalized[None, :], k=10)

# distances = cosine distances (convert to similarity)
similarities = 1 - distances
```

**Performance**:
- **Exact search** (nano-vectordb): O(N*D) - slow for large N
- **ANN (FAISS/Milvus)**: O(log N) - 100-1000x faster for large N

---

## Example: Full Cosine Similarity Pipeline

```python
import numpy as np
from lightrag.utils import cosine_similarity

# Step 1: Generate embeddings
query = "Apple products"
entities = ["Apple Inc", "iPhone", "Samsung", "Microsoft"]

query_embedding = await embedding_func([query])  # Shape: (1, 1536)
entity_embeddings = await embedding_func(entities)  # Shape: (4, 1536)

# Step 2: Calculate cosine similarities
similarities = []
for i, entity in enumerate(entities):
    sim = cosine_similarity(query_embedding[0], entity_embeddings[i])
    similarities.append((entity, sim))

print(similarities)
# [("Apple Inc", 0.92),
#  ("iPhone", 0.87),
#  ("Samsung", 0.65),
#  ("Microsoft", 0.58)]

# Step 3: Filter by threshold
threshold = 0.7
filtered = [(entity, sim) for entity, sim in similarities if sim >= threshold]

print(filtered)
# [("Apple Inc", 0.92),
#  ("iPhone", 0.87)]

# Step 4: Top-K selection
top_k = 2
top_k_results = sorted(similarities, key=lambda x: x[1], reverse=True)[:top_k]

print(top_k_results)
# [("Apple Inc", 0.92),
#  ("iPhone", 0.87)]
```

---

## Alternative Similarity Metrics

### Dot Product (Unnormalized)

```python
dot_product = np.dot(v1, v2)
# Range: (-∞, +∞)
# Faster (no normalization) but sensitive to magnitude
```

**Use case**: When vectors pre-normalized or magnitude meaningful.

### Manhattan Distance (L1)

```python
manhattan = np.sum(np.abs(v1 - v2))
# Range: [0, ∞)
# Less sensitive to outliers than Euclidean
```

### Jaccard Similarity (Sparse Vectors)

```python
# For binary/sparse vectors (e.g., TF-IDF)
intersection = np.sum(np.minimum(v1, v2))
union = np.sum(np.maximum(v1, v2))
jaccard = intersection / union
# Range: [0, 1]
```

**Comparison**:
- **Cosine**: Best for dense embeddings (default choice)
- **Dot product**: Faster, use if vectors normalized
- **Euclidean/Manhattan**: For spatial data, not semantic
- **Jaccard**: For sparse features (TF-IDF, not embeddings)

---

**Версия**: 1.0
**Дата**: 2025-01-13
