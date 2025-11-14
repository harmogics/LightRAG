# Embedding Generation: Генерация Векторных Представлений

## Концептуальная Парадигма

**Embedding generation** преобразует discrete text → continuous vector space, реализуя фундаментальную трансформацию:

```
Text (Symbolic)           Embedding (Semantic)
"Apple Inc"      →→→      [0.234, -0.567, 0.123, ..., 0.891]
                           ℝ^768 or ℝ^1536 or ℝ^4096
```

**Философия**: Embedding "encodes meaning" — семантически близкие тексты получают близкие vectors в continuous space.

---

## Алгоритм: Text-to-Vector Transformation

### Описание

**Embedding models** (neural networks) learned to map text → dense vectors such that:
- **Semantic similarity** → geometric proximity (cosine distance)
- **Distributed representation** → meaning spread across all dimensions
- **Fixed dimensionality** → all texts map to same ℝ^n

### Реализация

**Файл**: `lightrag/utils.py:340`

```python
@dataclass
class EmbeddingFunc:
    """
    Wrapper for embedding function.

    Embedding transforms text into continuous vector space:
    text (discrete) → vector (continuous)
    """
    embedding_dim: int           # Dimensionality (e.g., 768, 1536, 4096)
    func: callable               # Async embedding function
    max_token_size: int | None = None

    async def __call__(self, *args, **kwargs) -> np.ndarray:
        """
        Generate embeddings for batch of texts.

        Args:
            texts: List of strings to embed

        Returns:
            np.ndarray of shape (batch_size, embedding_dim)
        """
        return await self.func(*args, **kwargs)
```

### OpenAI Embedding (Example Implementation)

**Файл**: `lightrag/llm/openai.py` (conceptual)

```python
from openai import AsyncOpenAI
import numpy as np

async def openai_embedding(
    texts: list[str],
    model: str = "text-embedding-3-small",  # 1536 dims
    dimensions: int = None,  # Optional dimensionality reduction
    _priority: int = 0,  # Internal priority for queue
) -> np.ndarray:
    """
    Generate embeddings using OpenAI API.

    Metaphor: Neural network "reads" text and produces
    continuous semantic fingerprint in ℝ^n.
    """
    client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))

    # API call
    response = await client.embeddings.create(
        model=model,
        input=texts,
        dimensions=dimensions  # Optional: reduce from 1536 → 768
    )

    # Extract embeddings
    embeddings = np.array([
        data.embedding for data in response.data
    ])

    # Shape: (len(texts), embedding_dim)
    return embeddings
```

### Batch Processing

**Файл**: `lightrag/kg/nano_vector_db_impl.py:112`

```python
async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
    """
    Upsert embeddings with batch processing.
    """
    contents = [v["content"] for v in data.values()]

    # Split into batches (avoid API limits)
    batches = [
        contents[i : i + self._max_batch_size]
        for i in range(0, len(contents), self._max_batch_size)
    ]

    # Parallel embedding generation
    embedding_tasks = [self.embedding_func(batch) for batch in batches]
    embeddings_list = await asyncio.gather(*embedding_tasks)

    # Concatenate batches
    embeddings = np.concatenate(embeddings_list)
    # Shape: (total_items, embedding_dim)

    # Store embeddings
    for i, embedding in enumerate(embeddings):
        data[i]["embedding"] = embedding
```

### Complexity

- **Time**: O(N * M) where N = batch size, M = model inference time
  - OpenAI API: ~200-500ms for batch of 32 texts
  - Local models: ~50-200ms depending on GPU
- **Space**: O(N * D) where D = embedding dimensionality

### Parameters

```python
model: str = "text-embedding-3-small"  # Embedding model
    # Options:
    # - text-embedding-3-small (1536 dims, $0.02/1M tokens)
    # - text-embedding-3-large (3072 dims, $0.13/1M tokens)
    # - text-embedding-ada-002 (1536 dims, legacy)

embedding_dim: int = 1536              # Vector dimensionality
    # Typical sizes:
    # - 384 dims: sentence-transformers/all-MiniLM-L6-v2
    # - 768 dims: BERT-base, sentence-transformers/all-mpnet-base-v2
    # - 1536 dims: OpenAI text-embedding-3-small
    # - 4096 dims: OpenAI text-embedding-3-large

batch_size: int = 32                   # API batch size (OpenAI max = 2048)
```

---

## Связь с Dual-Space Architecture

Embedding generation реализует **projection from symbolic space → vector space** (spec/research/03-dual-space-architecture.md):

```
┌─────────────────────────────────────────┐
│      SYMBOLIC TEXT SPACE                │
│                                         │
│  "Apple Inc"                            │
│  "Steve Jobs founded Apple"             │
│  "iPhone is a product by Apple"         │
└───────────────┬─────────────────────────┘
                │ Embedding Function
                │ (Neural Network)
                ↓
┌─────────────────────────────────────────┐
│      CONTINUOUS VECTOR SPACE (ℝ^n)     │
│                                         │
│  v("Apple Inc") = [0.23, -0.56, ...]   │
│  v("Steve Jobs") = [0.25, -0.51, ...]  │
│  v("iPhone") = [0.26, -0.53, ...]      │
│                                         │
│  Cosine similarity:                     │
│  sim(Apple, iPhone) = 0.87  (high)     │
│  sim(Apple, Random) = 0.12  (low)      │
└─────────────────────────────────────────┘
```

**Key insight**: Semantically related concepts cluster in vector space → enable similarity search.

---

## Связь с Query-as-Key Paradigm

Embedding generation transforms **query into semantic key** (spec/research/01-query-as-semantic-key.md):

```
Query Text
  ↓ [Embedding]
Query Vector (ℝ^n)
  ↓ [Cosine Similarity]
Top-K Entities/Chunks (closest vectors)
  ↓ [BFS/Community Detection]
Subgraph Expansion
  ↓
Context for LLM
```

**Query vector** = **key** для navigating semantic space:
- **Direction** in ℝ^n определяет semantic intent
- **Magnitude** less important (normalized via cosine)
- **Nearest neighbors** = most relevant entities/chunks

---

## Embedding Models Comparison

### OpenAI Models

| Model | Dims | Cost (per 1M tokens) | Performance | Use Case |
|-------|------|---------------------|-------------|----------|
| **text-embedding-3-small** | 1536 | $0.02 | Good | Default (cost-effective) |
| **text-embedding-3-large** | 3072 | $0.13 | Best | High-precision tasks |
| **text-embedding-ada-002** | 1536 | $0.10 | Decent | Legacy (deprecated) |

### Open-Source Models

| Model | Dims | Speed | Use Case |
|-------|------|-------|----------|
| **all-MiniLM-L6-v2** | 384 | Very Fast | Low-resource environments |
| **all-mpnet-base-v2** | 768 | Fast | Balanced speed/quality |
| **bge-large-en-v1.5** | 1024 | Medium | High quality (SOTA) |
| **e5-large-v2** | 1024 | Medium | Multilingual |

### Choosing Embedding Model

```python
# Cost-sensitive (large volume)
embedding_func = openai_embedding(model="text-embedding-3-small", dims=1536)

# Quality-sensitive (critical retrieval)
embedding_func = openai_embedding(model="text-embedding-3-large", dims=3072)

# On-premise (no API)
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-mpnet-base-v2')
embedding_func = model.encode  # Local GPU inference
```

---

## Caching Embeddings

**Embeddings are expensive** → cache to avoid redundant computation.

### Embedding Cache Strategy

```python
# LightRAG automatically caches embeddings in vector DB
async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
    """
    Embeddings stored with content → reuse on retrieval.
    """
    # Generate embeddings once
    embeddings = await self.embedding_func(contents)

    # Store content + embedding
    for i, (id, item) in enumerate(data.items()):
        await vdb.upsert({
            id: {
                "content": item["content"],
                "embedding": embeddings[i],  # ← Cached
                "metadata": item.get("metadata")
            }
        })

# On query: reuse cached embeddings
async def query(self, query: str, top_k: int):
    # Compute query embedding (not cached - query is new)
    query_embedding = await self.embedding_func([query])

    # Compare against cached embeddings (fast)
    results = vdb.query(query_embedding[0], top_k=top_k)
    return results
```

**Benefit**: Only query embeddings computed dynamically; all entity/chunk embeddings pre-computed and cached.

---

## Dimensionality Reduction

OpenAI embeddings support **on-the-fly dimensionality reduction**:

```python
# Full dimensionality (best quality)
embeddings = await openai_embedding(
    texts,
    model="text-embedding-3-large",
    dimensions=3072  # Full size
)

# Reduced dimensionality (faster search, less storage)
embeddings = await openai_embedding(
    texts,
    model="text-embedding-3-large",
    dimensions=1024  # 3x compression
)
```

**Trade-off**:
- **Higher dims** → better semantic precision, slower search, more storage
- **Lower dims** → faster search, less storage, potential precision loss

**Recommendation**: Start with 1536 dims (text-embedding-3-small), reduce to 768-1024 if performance bottleneck.

---

## Связь с Dependencies

### OpenAI SDK (spec/dependencies/01-llm-integration.md)

```python
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = await client.embeddings.create(
    model="text-embedding-3-small",
    input=["Apple Inc", "Samsung", "Microsoft"],
    dimensions=1536
)

embeddings = np.array([data.embedding for data in response.data])
# Shape: (3, 1536)
```

### NumPy (spec/dependencies/05-data-processing.md)

```python
import numpy as np

# Convert to numpy array for vectorized operations
embeddings = np.array(embedding_list)  # Shape: (N, D)

# Normalize embeddings (for cosine similarity)
norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
normalized = embeddings / norms

# Batch cosine similarity
query_embedding = np.array([...])  # Shape: (D,)
similarities = normalized @ query_embedding
# Shape: (N,) - similarity scores for all N embeddings
```

### sentence-transformers (Open-Source Alternative)

```python
from sentence_transformers import SentenceTransformer

# Load model (runs locally, no API)
model = SentenceTransformer('all-mpnet-base-v2')

# Generate embeddings
embeddings = model.encode([
    "Apple Inc produces iPhones",
    "Samsung manufactures Galaxy phones"
])
# Shape: (2, 768)

# Async wrapper for LightRAG
async def local_embedding_func(texts):
    # Run in thread pool (CPU-bound)
    loop = asyncio.get_event_loop()
    embeddings = await loop.run_in_executor(
        None, model.encode, texts
    )
    return np.array(embeddings)
```

---

## Производительность

### Benchmark (OpenAI text-embedding-3-small)

| Batch Size | Tokens | API Time | Cost |
|-----------|--------|----------|------|
| 1 | 10 | 150ms | $0.0000002 |
| 32 | 320 | 250ms | $0.000006 |
| 100 | 1000 | 400ms | $0.00002 |
| 1000 | 10000 | 2s | $0.0002 |

**Observations**:
- **Batching critical** for throughput (32-100 items per batch optimal)
- **Cost negligible** compared to LLM generation ($0.02/1M tokens)
- **Latency dominated** by network roundtrip (~150-200ms base)

### Local Model Benchmark (all-mpnet-base-v2, GPU)

| Batch Size | GPU Time | Speedup vs API |
|-----------|----------|----------------|
| 1 | 15ms | 10x faster |
| 32 | 80ms | 3x faster |
| 100 | 200ms | 2x faster |

**Trade-off**:
- **Local models**: Faster, no cost, privacy, but require GPU infrastructure
- **API models**: Higher quality (typically), no infrastructure, but cost + latency

---

## Example: Full Embedding Pipeline

```python
from lightrag.utils import EmbeddingFunc
import numpy as np

# Initialize embedding function
embedding_func = EmbeddingFunc(
    embedding_dim=1536,
    func=openai_embedding_func  # Async function
)

# Step 1: Extract entities from text
entities = ["Apple Inc", "Samsung", "Microsoft", "Google"]

# Step 2: Generate embeddings (batched)
embeddings = await embedding_func(entities)
# Shape: (4, 1536)

# Step 3: Store in vector DB
await entities_vdb.upsert({
    "ent-apple": {
        "content": "Apple Inc",
        "embedding": embeddings[0],
        "entity_type": "organization"
    },
    "ent-samsung": {
        "content": "Samsung",
        "embedding": embeddings[1],
        "entity_type": "organization"
    },
    # ...
})

# Step 4: Query (later)
query = "tech companies"
query_embedding = await embedding_func([query])  # Shape: (1, 1536)

results = await entities_vdb.query(
    query=query,
    top_k=10,
    query_embedding=query_embedding[0]
)
# Returns top-10 entities by cosine similarity
```

---

## Future Enhancements

### 1. Adaptive Dimensionality

```python
# Choose dims based on query complexity
if query_is_simple(query):
    dims = 768  # Fast search
else:
    dims = 3072  # High precision
```

### 2. Multi-Modal Embeddings

```python
# Combine text + image embeddings (e.g., CLIP)
text_embedding = await text_encoder(text)
image_embedding = await image_encoder(image)

# Joint embedding
combined = α * text_embedding + β * image_embedding
```

### 3. Domain-Specific Fine-Tuning

```python
# Fine-tune embedding model on domain corpus
from sentence_transformers import SentenceTransformer, InputExample
from sentence_transformers import losses, evaluation

model = SentenceTransformer('all-mpnet-base-v2')

# Domain-specific training data
train_examples = [
    InputExample(texts=['Apple Inc', 'iPhone producer'], label=1.0),
    InputExample(texts=['Apple Inc', 'random text'], label=0.0),
    # ...
]

# Fine-tune
model.fit(
    train_objectives=[(train_dataloader, losses.CosineSimilarityLoss(model))],
    epochs=3
)

# Use fine-tuned model
embeddings = model.encode(texts)
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
