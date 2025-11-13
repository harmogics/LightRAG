# Vector & Embedding Processing: Векторная Обработка и Embeddings

## Обзор

Библиотеки для работы с векторными представлениями (embeddings) и операциями в continuous semantic space. Эти зависимости обеспечивают **проекцию в семантическое пространство** и **similarity search** для entity resolution.

## Core Libraries

### tiktoken

**Версия**: Latest
**Лицензия**: MIT
**Сайт**: https://github.com/openai/tiktoken

#### Назначение

Fast BPE tokenizer от OpenAI (written in Rust). Используется для:
- **Токенизация** текста перед chunking
- **Подсчет токенов** для LLM context limits
- **Chunk size control** при разбиении документов

#### Использование в LightRAG

**Файл**: `lightrag/utils.py`

```python
import tiktoken

class TiktokenTokenizer:
    """
    Wrapper for tiktoken tokenizer.

    Models:
    • cl100k_base (GPT-4, GPT-3.5-turbo, text-embedding-ada-002)
    • o200k_base (GPT-4o)
    """

    def __init__(self, model_name="gpt-4o-mini"):
        self.encoding = tiktoken.encoding_for_model(model_name)

    def encode(self, text: str) -> list[int]:
        """Text → Token IDs"""
        return self.encoding.encode(text)

    def decode(self, tokens: list[int]) -> str:
        """Token IDs → Text"""
        return self.encoding.decode(tokens)

    def count_tokens(self, text: str) -> int:
        """Fast token counting"""
        return len(self.encode(text))
```

**Роль в chunking**:
```python
def chunking_by_token_size(
    tokenizer,
    content: str,
    max_token_size=1024,
    overlap_token_size=128
):
    """
    Chunk text by token count (not character count).

    Process:
    1. Encode entire text → token IDs
    2. Split tokens into overlapping windows
    3. Decode each window back to text

    Benefits:
    • Accurate LLM context size control
    • Token-aware overlap (semantic continuity)
    """
    tokens = tokenizer.encode(content)

    chunks = []
    for start in range(0, len(tokens), max_token_size - overlap_token_size):
        chunk_tokens = tokens[start:start + max_token_size]
        chunk_text = tokenizer.decode(chunk_tokens)
        chunks.append({
            "content": chunk_text,
            "tokens": len(chunk_tokens),
            "chunk_order_index": len(chunks)
        })

    return chunks
```

**Performance**:
- **Speed**: 100-1000x faster than pure Python tokenizers
- **Language**: Rust implementation
- **Memory**: Efficient byte-pair encoding

---

### numpy

**Версия**: Latest
**Лицензия**: BSD
**Сайт**: https://numpy.org

#### Назначение

Фундаментальная библиотека для numerical computing. Используется для:
- **Vector operations** на embeddings
- **Cosine similarity** calculation
- **Matrix operations** для batch processing

#### Использование в LightRAG

**Файл**: `lightrag/utils.py`, storage implementations

```python
import numpy as np

def cosine_similarity(vec1: np.ndarray, vec2: np.ndarray) -> float:
    """
    Compute cosine similarity between two vectors.

    Formula: cos(θ) = (A · B) / (||A|| × ||B||)
    """
    dot_product = np.dot(vec1, vec2)
    norm_product = np.linalg.norm(vec1) * np.linalg.norm(vec2)

    if norm_product == 0:
        return 0.0

    return dot_product / norm_product

def batch_cosine_similarity(query_vec: np.ndarray, doc_vecs: np.ndarray) -> np.ndarray:
    """
    Vectorized cosine similarity for batch processing.

    Input:
    • query_vec: (D,) - query embedding
    • doc_vecs: (N, D) - N document embeddings

    Output:
    • similarities: (N,) - similarity scores
    """
    # Normalize vectors
    query_norm = query_vec / np.linalg.norm(query_vec)
    doc_norms = doc_vecs / np.linalg.norm(doc_vecs, axis=1, keepdims=True)

    # Matrix multiplication for batch similarity
    similarities = np.dot(doc_norms, query_norm)

    return similarities
```

**Embedding Operations**:
```python
# Entity embeddings storage
entity_embeddings = {
    "Apple Inc": np.array([0.234, -0.567, 0.123, ...], dtype=np.float32),
    "iPhone": np.array([0.256, -0.512, 0.145, ...], dtype=np.float32),
    ...
}

# Query embedding
query_embedding = np.array([0.245, -0.534, 0.134, ...], dtype=np.float32)

# Find top-k most similar entities
similarities = {
    entity: cosine_similarity(query_embedding, emb)
    for entity, emb in entity_embeddings.items()
}

top_k = sorted(similarities.items(), key=lambda x: x[1], reverse=True)[:10]
```

**Vector Space Operations**:
```python
# L2 normalization
embedding = embedding / np.linalg.norm(embedding)

# Dimensionality check
assert embedding.shape == (768,)  # OpenAI ada-002
assert embedding.shape == (1536,) # OpenAI text-embedding-3

# Batch processing
batch_embeddings = np.stack([emb1, emb2, emb3, ...])  # (N, D)
mean_embedding = np.mean(batch_embeddings, axis=0)    # (D,)
```

---

### nano-vectordb

**Версия**: Latest
**Лицензия**: MIT
**Сайт**: https://github.com/gusye1234/nano-vectordb

#### Назначение

Lightweight pure-Python vector database. **Default vector storage** в LightRAG для малых/средних deployments.

#### Использование в LightRAG

**Файл**: `lightrag/kg/nano_vector_db_impl.py`

```python
from nano_vectordb import NanoVectorDB

class NanoVectorDBStorage(BaseVectorStorage):
    """
    Lightweight vector storage using nano-vectordb.

    Features:
    • Pure Python (no external dependencies)
    • In-memory with persistence (JSON)
    • HNSW-like indexing for fast search
    • Good for < 1M vectors
    """

    def __init__(self, namespace, global_config, embedding_func, ...):
        self.client = NanoVectorDB(
            storage_file=f"{working_dir}/{namespace}.json",
            dim=embedding_dim  # 768, 1536, etc.
        )

    async def upsert(self, data: dict[str, dict]):
        """Insert/update vectors"""
        for id, item in data.items():
            self.client.upsert(
                id=id,
                vector=item["embedding"],
                metadata={"entity_name": item["entity_name"], ...}
            )

    async def query(self, query: str, top_k: int = 10) -> list[dict]:
        """Similarity search"""
        query_embedding = await self.embedding_func([query])
        query_embedding = query_embedding[0]

        results = self.client.search(
            query_vector=query_embedding,
            top_k=top_k,
            threshold=self.cosine_better_than_threshold
        )

        return [
            {
                "id": r.id,
                "score": r.score,
                **r.metadata
            }
            for r in results
        ]
```

**Internal Algorithm** (simplified):
```python
# HNSW-inspired indexing
class NanoVectorDB:
    def __init__(self):
        self.vectors = {}  # id → vector
        self.metadata = {}  # id → metadata
        self.index = {}     # Hierarchical graph structure

    def search(self, query_vector, top_k):
        """
        Approximate nearest neighbor search.

        Algorithm:
        1. Start at entry point in top layer
        2. Greedy search to find nearest in current layer
        3. Move to next layer down
        4. Repeat until bottom layer
        5. Return top-k candidates
        """
        candidates = []
        current = self.entry_point

        for layer in range(self.num_layers, -1, -1):
            current = self._greedy_search(query_vector, current, layer)

        # Final refinement at bottom layer
        candidates = self._search_layer(query_vector, current, layer=0, ef=top_k*2)

        # Compute exact similarities
        results = []
        for candidate_id in candidates[:top_k]:
            similarity = cosine_similarity(query_vector, self.vectors[candidate_id])
            results.append((candidate_id, similarity))

        return sorted(results, key=lambda x: x[1], reverse=True)
```

**Performance Characteristics**:
```
Vector Count | Index Time | Search Time | Memory
-------------|------------|-------------|--------
1K           | 0.1s       | 1-5ms       | 10MB
10K          | 1s         | 5-10ms      | 100MB
100K         | 10s        | 10-20ms     | 1GB
1M           | 100s       | 20-50ms     | 10GB
```

**Advantages**:
- Zero external dependencies (pure Python)
- Easy setup (no server required)
- Persistence via JSON
- Good for development/testing

**Limitations**:
- Not for production at scale (> 1M vectors)
- No distributed search
- In-memory (limited by RAM)

---

## Vector Processing Pipeline

### Embedding Generation

```python
# Step 1: Text → Tokens
text = "Apple Inc manufactures iPhone"
tokens = tokenizer.encode(text)  # tiktoken
# [23758, 4953, 97345, 17433]

# Step 2: Tokens → Embedding (via LLM API)
embedding = await embedding_func([text])  # openai
# [[0.234, -0.567, 0.123, ..., 0.891]]

# Step 3: Normalize (optional)
embedding = np.array(embedding[0])
embedding = embedding / np.linalg.norm(embedding)  # numpy

# Step 4: Store in vector DB
await vector_db.upsert({
    "entity_42": {
        "embedding": embedding.tolist(),
        "entity_name": "Apple Inc",
        ...
    }
})  # nano-vectordb
```

### Similarity Search

```python
# Step 1: Query → Embedding
query = "tech companies producing phones"
query_embedding = await embedding_func([query])
query_embedding = np.array(query_embedding[0])

# Step 2: Vector search
results = await vector_db.query(
    query,
    top_k=10,
    query_embedding=query_embedding  # Pre-computed
)

# results = [
#     {"entity_name": "Apple Inc", "score": 0.92},
#     {"entity_name": "Samsung", "score": 0.87},
#     ...
# ]

# Step 3: Filter by threshold
filtered = [
    r for r in results
    if r["score"] >= cosine_threshold  # e.g., 0.2
]
```

---

## Alternative Vector Databases

While `nano-vectordb` is the **default**, LightRAG supports multiple vector DBs:

### FAISS (Facebook AI Similarity Search)

```python
# pip install faiss-cpu  # or faiss-gpu
import faiss

index = faiss.IndexFlatIP(dimension)  # Inner product (cosine after L2 norm)
index.add(vectors)  # Add vectors
D, I = index.search(query_vector, k=10)  # Search
```

**Use Case**: Large-scale (millions of vectors), GPU acceleration.

### Milvus

```python
# pip install pymilvus
from pymilvus import connections, Collection

connections.connect(host="localhost", port="19530")
collection = Collection("entities")
results = collection.search(query_vector, "embedding", param={}, limit=10)
```

**Use Case**: Distributed, production-grade, billions of vectors.

### Qdrant

```python
# pip install qdrant-client
from qdrant_client import QdrantClient

client = QdrantClient(host="localhost", port=6333)
results = client.search(
    collection_name="entities",
    query_vector=query_embedding,
    limit=10
)
```

**Use Case**: Modern API, easy deployment, good for medium scale.

---

## Optimization Strategies

### Batch Embedding

```python
# Bad: Sequential (slow)
embeddings = []
for text in texts:
    emb = await embedding_func([text])
    embeddings.append(emb[0])

# Good: Batch (fast)
embeddings = await embedding_func(texts)  # Batch API call
# 10x-100x faster
```

### Embedding Caching

```python
embedding_cache = {}

def get_embedding(text):
    cache_key = md5(text.encode()).hexdigest()

    if cache_key in embedding_cache:
        return embedding_cache[cache_key]

    embedding = await embedding_func([text])
    embedding_cache[cache_key] = embedding[0]

    return embedding[0]
```

### Dimensionality Reduction (Optional)

```python
from sklearn.decomposition import PCA

# Reduce 1536-dim → 768-dim (2x memory savings)
pca = PCA(n_components=768)
reduced_embeddings = pca.fit_transform(high_dim_embeddings)

# Trade-off: ~5% accuracy loss, 50% memory savings
```

---

## Related Documentation

- **[Dual-Space Architecture](../research/03-dual-space-architecture.md)** - Vector space vs Graph space
- **[Entity Resolution (T8)](../transform/02-query-transforms.md#t8-entity-resolution)** - Uses vector search
- **[Storage Abstraction](../architecture/01-storage-abstraction.md)** - BaseVectorStorage interface

---

**Версия**: 1.0
**Дата**: 2025-01-13
