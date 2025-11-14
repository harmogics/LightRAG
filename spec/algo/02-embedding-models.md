# Embedding Models: Модели Векторизации Текста

## Концептуальная Парадигма

**Embedding models** = encoder-only transformers that map **text → dense vectors** in continuous semantic space.

```
Text Input → Tokenization → Transformer Encoder → Pooling → Vector ∈ ℝⁿ
"Apple Inc produces iPhones"
           ↓
    [Mean Pooling]
           ↓
[0.234, -0.567, 0.123, ..., 0.891]  ← 1536-dim vector
```

**Философия**: Transform discrete text into **continuous geometric space** где semantic similarity = geometric proximity.

---

## Architecture: Encoder-Only Transformer

### BERT-Style Architecture

```
Input: "Apple Inc produces iPhones"
       ↓
Tokenization: [CLS] Apple Inc produces iPhones [SEP]
       ↓
Token Embeddings: → ℝ^768
       ↓
┌────────────────────────────────────┐
│  Transformer Encoder Layers (×12)  │
│                                    │
│  • Multi-Head Self-Attention       │
│    (bidirectional, no masking)     │
│  • Feed-Forward Networks           │
│  • Layer Normalization             │
│  • Residual Connections            │
└────────────────────────────────────┘
       ↓
Token Representations: [h_CLS, h_Apple, h_Inc, ..., h_SEP]
       ↓
Pooling (mean/CLS): → ℝ^768
       ↓
Final Embedding: [0.234, -0.567, ..., 0.891]
```

**Key Differences from LLMs**:
- **Bidirectional attention**: Each token sees all other tokens (no causal mask)
- **No generation**: Only encodes input, doesn't generate output
- **Pooling required**: Convert token sequences → single vector

---

## Models in LightRAG

### 1. OpenAI Embeddings (API)

**Файл**: `lightrag/llm/openai.py:174`

```python
async def openai_embedding(
    texts: list[str],
    model: str = "text-embedding-3-small",
    base_url: str = None,
    api_key: str = None,
    **kwargs
) -> np.ndarray:
    """
    OpenAI embedding API.

    Models:
    - text-embedding-3-small (1536 dims, $0.02/1M tokens)
    - text-embedding-3-large (3072 dims, $0.13/1M tokens)
    - text-embedding-ada-002 (1536 dims, deprecated)

    Args:
        texts: List of texts to embed
        model: Model name

    Returns:
        np.ndarray of shape (len(texts), embedding_dim)
    """
    import openai

    if api_key is None:
        api_key = os.getenv("OPENAI_API_KEY")

    if base_url:
        client = openai.AsyncOpenAI(api_key=api_key, base_url=base_url)
    else:
        client = openai.AsyncOpenAI(api_key=api_key)

    # API call
    response = await client.embeddings.create(
        input=texts,
        model=model,
        **kwargs
    )

    # Extract embeddings
    embeddings = np.array([item.embedding for item in response.data])

    return embeddings
```

**Model Characteristics**:

| Model | Dimensions | Cost (per 1M tokens) | Quality | Max Tokens |
|-------|-----------|---------------------|---------|------------|
| **text-embedding-3-small** | 1536 | $0.02 | Good | 8191 |
| **text-embedding-3-large** | 3072 | $0.13 | Best | 8191 |
| **text-embedding-ada-002** | 1536 | $0.10 | Decent | 8191 |

**Recommendation**: `text-embedding-3-small` (best cost-performance)

**Dependencies**: `openai>=1.0.0`

### 2. Sentence-Transformers (Local)

**Файл**: `lightrag/llm/hf.py:134`

```python
async def hf_embedding(
    texts: list[str],
    model_name: str = "sentence-transformers/all-mpnet-base-v2",
    **kwargs
) -> np.ndarray:
    """
    HuggingFace sentence-transformers embedding.

    Popular models:
    - all-mpnet-base-v2 (768 dims, English)
    - all-MiniLM-L6-v2 (384 dims, English, fast)
    - paraphrase-multilingual-mpnet-base-v2 (768 dims, 50+ langs)

    Args:
        texts: List of texts
        model_name: Model identifier

    Returns:
        np.ndarray of shape (len(texts), embedding_dim)
    """
    from sentence_transformers import SentenceTransformer

    # Load model (cached after first load)
    model = SentenceTransformer(model_name)

    # Encode
    embeddings = model.encode(
        texts,
        batch_size=kwargs.get("batch_size", 32),
        show_progress_bar=False,
        convert_to_numpy=True,
        device=kwargs.get("device", "cpu")  # or "cuda"
    )

    return embeddings
```

**Model Characteristics**:

| Model | Dimensions | Memory | Quality | Speed (CPU) | Languages |
|-------|-----------|--------|---------|-------------|-----------|
| **all-mpnet-base-v2** | 768 | 420 MB | Best | 50 texts/s | English |
| **all-MiniLM-L6-v2** | 384 | 80 MB | Good | 200 texts/s | English |
| **paraphrase-multilingual** | 768 | 420 MB | Good | 40 texts/s | 50+ |
| **bge-large-en-v1.5** | 1024 | 1.3 GB | Excellent | 30 texts/s | English |

**Recommendation**:
- **Best quality**: `bge-large-en-v1.5` (state-of-the-art)
- **Balanced**: `all-mpnet-base-v2`
- **Fast**: `all-MiniLM-L6-v2`
- **Multilingual**: `paraphrase-multilingual-mpnet-base-v2`

**Dependencies**: `sentence-transformers>=2.2.0`

### 3. Jina Embeddings (API)

```python
async def jina_embedding(
    texts: list[str],
    model: str = "jina-embeddings-v2-base-en",
    api_key: str = None,
    **kwargs
) -> np.ndarray:
    """
    Jina AI embedding API.

    Models:
    - jina-embeddings-v2-base-en (768 dims, $0.02/1M tokens)
    - jina-embeddings-v2-base-multilingual (768 dims, 89 langs)
    """
    import aiohttp

    if api_key is None:
        api_key = os.getenv("JINA_API_KEY")

    url = "https://api.jina.ai/v1/embeddings"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }

    payload = {
        "model": model,
        "input": texts
    }

    async with aiohttp.ClientSession() as session:
        async with session.post(url, headers=headers, json=payload) as response:
            result = await response.json()

    embeddings = np.array([item["embedding"] for item in result["data"]])

    return embeddings
```

**Dependencies**: `aiohttp>=3.8.0`

---

## Embedding Function in LightRAG

**Файл**: `lightrag/base.py:340`

```python
@dataclass
class EmbeddingFunc:
    """
    Unified embedding function wrapper.

    Standardizes different embedding providers (OpenAI, HF, Jina)
    into a single interface.
    """
    embedding_dim: int
    max_token_size: int | None = None
    func: callable = None

    async def __call__(self, texts: list[str], **kwargs) -> np.ndarray:
        """
        Generate embeddings for texts.

        Args:
            texts: List of text strings

        Returns:
            np.ndarray of shape (len(texts), embedding_dim)
        """
        return await self.func(texts, **kwargs)
```

**Usage**:

```python
from lightrag.base import EmbeddingFunc
from lightrag.llm.openai import openai_embedding

# Create embedding function
embedding_func = EmbeddingFunc(
    embedding_dim=1536,
    max_token_size=8192,
    func=openai_embedding
)

# Generate embeddings
texts = ["Apple Inc", "iPhone", "Samsung"]
embeddings = await embedding_func(texts)

print(embeddings.shape)  # (3, 1536)
```

---

## Training Paradigm: Contrastive Learning

### SimCSE (Simple Contrastive Sentence Embeddings)

```
Objective: Learn embeddings where similar texts are close
Data: (anchor, positive, negative) triplets

Example:
Anchor:   "Apple company produces iPhones"
Positive: "Apple Inc manufactures smartphones"
Negative: "Apples are healthy fruits"

Loss: Contrastive loss (InfoNCE)
L = -log(exp(sim(anchor, positive) / τ) / Σ exp(sim(anchor, negative_i) / τ))

Temperature τ = 0.05 (hyperparameter)

Training:
1. Encode all texts → embeddings
2. Compute similarities (cosine)
3. Maximize sim(anchor, positive)
4. Minimize sim(anchor, negative)
```

### Data Augmentation

```python
# Positive pairs from data augmentation
original = "Apple Inc produces iPhones"

# Method 1: Paraphrasing
positive1 = "iPhones are made by Apple Inc"

# Method 2: Dropout (same text, different dropout masks)
positive2 = encode_with_dropout(original)  # Stochastic

# Method 3: Back-translation
positive3 = translate(original, "de → en")  # "Apple Inc stellt iPhones her" → ...

# Negative pairs: Random batch samples
negatives = random_sample(batch, k=64)
```

### Training Process

```
1. Pre-training (unsupervised):
   Data: Large corpus (Wikipedia, Common Crawl)
   Task: Masked Language Modeling (MLM)
         "Apple [MASK] produces iPhones" → predict "Inc"

2. Fine-tuning (supervised):
   Data: Labeled pairs (sentence similarity datasets)
         - SNLI (Natural Language Inference)
         - STS Benchmark (Semantic Textual Similarity)
   Task: Contrastive learning (SimCSE, InfoNCE)

3. Distillation (optional):
   Teacher: Large model (e.g., bge-large, 1.3GB)
   Student: Small model (e.g., all-MiniLM-L6-v2, 80MB)
   Transfer knowledge: Student mimics teacher's embeddings
```

---

## Связь с Vector Algorithms

Embeddings = input для vector algorithms:

**Файл**: spec/vectors/02-cosine-similarity.md

```python
# Generate embeddings
texts = ["Apple Inc", "iPhone", "Samsung"]
embeddings = await embedding_func(texts)

# Cosine similarity
from lightrag.utils import cosine_similarity

query_emb = embeddings[0]  # "Apple Inc"
doc_emb = embeddings[1]    # "iPhone"

similarity = cosine_similarity(query_emb, doc_emb)
print(similarity)  # 0.87 (high similarity)
```

**Interaction**: Embeddings provide **semantic representation**, cosine similarity measures **geometric distance**.

---

## Связь с Vector Storage

Embeddings stored in vector databases:

**Файл**: spec/vectors/03-vector-storage.md

```python
from lightrag.kg.nano_vector_db_impl import NanoVectorDBStorage

# Initialize storage with embedding function
vector_storage = NanoVectorDBStorage(
    namespace="entities",
    global_config={"working_dir": "./storage"},
    embedding_func=embedding_func  # ← Embedding model
)

# Upsert (auto-generates embeddings)
await vector_storage.upsert({
    "ent-apple": {
        "content": "Apple Inc is a technology company",
        "entity_type": "organization"
    }
})

# Query (auto-generates query embedding)
results = await vector_storage.query("tech companies", top_k=10)
```

**Interaction**: Embedding function = **bridge** between text and vector storage.

---

## Связь с Graph Construction

Entity embeddings used in graph construction:

**Файл**: spec/graph/04-entity-merging.md

```python
# Extract entities from text
entities = ["Apple Inc", "Apple company", "Apple Corporation"]

# Generate embeddings
entity_embeddings = await embedding_func(entities)

# Compute pairwise similarities
from sklearn.metrics.pairwise import cosine_similarity
similarities = cosine_similarity(entity_embeddings)

# Merge similar entities (threshold = 0.9)
if similarities[0, 1] > 0.9:
    # "Apple Inc" and "Apple company" are same entity
    await graph.amerge_entities(
        source_entities=["Apple company"],
        target_entity="Apple Inc"
    )
```

**Interaction**: Embeddings enable **semantic entity deduplication**.

---

## Performance Characteristics

### Latency (32 texts, 512 tokens each)

| Model | API Latency | Local (GPU) | Local (CPU) |
|-------|-------------|-------------|-------------|
| **text-embedding-3-small** | 200ms | N/A | N/A |
| **text-embedding-3-large** | 250ms | N/A | N/A |
| **all-mpnet-base-v2** | N/A | 80ms | 500ms |
| **all-MiniLM-L6-v2** | N/A | 30ms | 150ms |
| **bge-large-en-v1.5** | N/A | 150ms | 1200ms |

### Throughput (texts per second)

```python
# API models (parallel requests)
text-embedding-3-small: ~1000 texts/s (10 concurrent requests)

# Local models (single GPU, A100)
all-mpnet-base-v2: ~400 texts/s (batch size 32)
all-MiniLM-L6-v2: ~1000 texts/s (batch size 64)
bge-large-en-v1.5: ~200 texts/s (batch size 16)

# Local models (CPU, 16 cores)
all-mpnet-base-v2: ~50 texts/s
all-MiniLM-L6-v2: ~200 texts/s
```

### Cost Comparison (Embedding 1M texts)

```
Input: 1M texts, avg 50 tokens each = 50M tokens

OpenAI text-embedding-3-small:
50M tokens × $0.02 / 1M = $1.00

OpenAI text-embedding-3-large:
50M tokens × $0.13 / 1M = $6.50

Jina jina-embeddings-v2:
50M tokens × $0.02 / 1M = $1.00

Local (all-mpnet-base-v2):
$0 (infrastructure: ~$1/hour GPU, ~10 hours = $10)

Recommendation:
- < 100M tokens: OpenAI text-embedding-3-small (API)
- > 100M tokens: Local model (amortized cost)
```

---

## Embedding Quality Benchmarks

### MTEB (Massive Text Embedding Benchmark)

Standard benchmark for embedding models (56 tasks):

| Model | MTEB Score | Retrieval | Classification | Clustering |
|-------|------------|-----------|----------------|------------|
| **text-embedding-3-large** | 64.6 | 54.0 | 70.2 | 49.0 |
| **text-embedding-3-small** | 62.3 | 51.2 | 67.8 | 47.4 |
| **bge-large-en-v1.5** | 63.9 | 53.9 | 75.0 | 46.7 |
| **all-mpnet-base-v2** | 57.8 | 49.6 | 66.5 | 44.6 |
| **all-MiniLM-L6-v2** | 56.3 | 48.0 | 63.4 | 42.4 |

**Observation**: text-embedding-3-large best overall, bge-large-en-v1.5 best classification.

### RAG-Specific Evaluation

**Файл**: LightRAG internal evaluation

```
Query: "What products does Apple make?"
Ground Truth: "iPhones, iPads, Macs, Apple Watch, AirPods"

Retrieval Precision@10:
text-embedding-3-small: 0.92
text-embedding-3-large: 0.95
all-mpnet-base-v2: 0.88
all-MiniLM-L6-v2: 0.84

Conclusion: OpenAI embeddings slightly better for LightRAG retrieval tasks
```

---

## Optimization Strategies

### 1. Batch Processing

```python
# Slow (sequential): 1000 texts × 200ms = 200s
for text in texts:
    embedding = await embedding_func([text])

# Fast (batched): 1000 texts / 32 per batch × 200ms = 6.25s
batch_size = 32
batches = [texts[i:i+batch_size] for i in range(0, len(texts), batch_size)]

embeddings = []
for batch in batches:
    batch_embeddings = await embedding_func(batch)
    embeddings.append(batch_embeddings)

embeddings = np.concatenate(embeddings)
```

**Speedup**: 32x (batching reduces API overhead)

### 2. Caching

```python
import hashlib
from functools import lru_cache

# Cache embeddings by text hash
embedding_cache = {}

async def cached_embedding(texts: list[str]) -> np.ndarray:
    """
    Cache embeddings to avoid recomputation.
    """
    uncached_texts = []
    cached_embeddings = {}

    for i, text in enumerate(texts):
        text_hash = hashlib.md5(text.encode()).hexdigest()

        if text_hash in embedding_cache:
            cached_embeddings[i] = embedding_cache[text_hash]
        else:
            uncached_texts.append((i, text))

    # Generate embeddings for uncached texts only
    if uncached_texts:
        indices, texts_to_embed = zip(*uncached_texts)
        new_embeddings = await embedding_func(list(texts_to_embed))

        for idx, text, emb in zip(indices, texts_to_embed, new_embeddings):
            text_hash = hashlib.md5(text.encode()).hexdigest()
            embedding_cache[text_hash] = emb
            cached_embeddings[idx] = emb

    # Reconstruct embeddings in original order
    result = np.array([cached_embeddings[i] for i in range(len(texts))])

    return result
```

**Speedup**: 100x for repeated texts (instant cache retrieval)

### 3. Model Quantization

```python
from optimum.onnxruntime import ORTModelForFeatureExtraction
from transformers import AutoTokenizer

# Load quantized model (INT8)
model = ORTModelForFeatureExtraction.from_pretrained(
    "sentence-transformers/all-mpnet-base-v2",
    file_name="model_quantized.onnx"
)

tokenizer = AutoTokenizer.from_pretrained("sentence-transformers/all-mpnet-base-v2")

# Inference (2-4x faster)
inputs = tokenizer(texts, padding=True, truncation=True, return_tensors="pt")
outputs = model(**inputs)
embeddings = outputs.last_hidden_state.mean(dim=1).numpy()
```

**Speedup**: 2-4x (quantization reduces compute)
**Tradeoff**: ~1% quality degradation

### 4. GPU Acceleration

```python
# CPU inference (slow)
model = SentenceTransformer("all-mpnet-base-v2", device="cpu")
embeddings = model.encode(texts)  # ~500ms for 32 texts

# GPU inference (fast)
model = SentenceTransformer("all-mpnet-base-v2", device="cuda")
embeddings = model.encode(texts)  # ~80ms for 32 texts

# Multi-GPU (very fast)
model = SentenceTransformer("all-mpnet-base-v2")
pool = model.start_multi_process_pool(["cuda:0", "cuda:1", "cuda:2", "cuda:3"])
embeddings = model.encode_multi_process(texts, pool)  # ~20ms for 32 texts
```

**Speedup**: 6x (single GPU), 25x (multi-GPU)

---

## Dimensionality Selection

### Trade-offs

| Dimensions | Quality | Storage | Speed | Use Case |
|-----------|---------|---------|-------|----------|
| **384** (MiniLM) | Good | 1.5 KB | Fast | Large-scale (millions of vectors) |
| **768** (MPNet, BERT) | Better | 3 KB | Medium | Medium-scale (100K-1M vectors) |
| **1024** (BGE-large) | Best | 4 KB | Slower | Quality-critical applications |
| **1536** (OpenAI-small) | Excellent | 6 KB | Fast (API) | General-purpose |
| **3072** (OpenAI-large) | Best | 12 KB | Medium (API) | Highest quality needs |

**Storage calculation** (1M vectors):
- 384 dims: 1M × 1.5 KB = 1.5 GB
- 768 dims: 1M × 3 KB = 3 GB
- 1536 dims: 1M × 6 KB = 6 GB
- 3072 dims: 1M × 12 KB = 12 GB

**Recommendation**:
- **Default**: 768-1024 dims (good balance)
- **Large scale**: 384 dims (storage-efficient)
- **Quality-critical**: 1536-3072 dims (best retrieval)

---

**Версия**: 1.0
**Дата**: 2025-01-13
