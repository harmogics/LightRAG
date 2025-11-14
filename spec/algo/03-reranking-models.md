# Reranking Models: Cross-Encoder переранжирование

## Концептуальная Парадигма

**Cross-encoder reranking models** = transformer models для **precise relevance scoring** between query and document pairs.

```
Bi-Encoder (Stage 1: Retrieval)        Cross-Encoder (Stage 2: Reranking)
Query → [Encoder] → v_q                [Query, Doc] → [Encoder] → Score
Doc   → [Encoder] → v_d
sim = cosine(v_q, v_d)                 Direct relevance score (0-1)

✅ Fast (independent encoding)         ✅ Precise (joint attention)
❌ Limited interaction                 ❌ Slow (pairwise encoding)
```

**Философия**: Bi-encoders для **candidate retrieval** (broad), cross-encoders для **reranking** (precise refinement).

---

## Architecture: Full Transformer with Cross-Attention

### Cross-Encoder vs Bi-Encoder

```
Bi-Encoder Architecture:
─────────────────────────
Query:  "Apple products"        → Encoder → [0.23, -0.56, ...]
Doc:    "iPhone by Apple Inc"  → Encoder → [0.25, -0.51, ...]

Similarity = cosine([0.23...], [0.25...]) = 0.87

Problem: Query and Doc encoded independently (no interaction)


Cross-Encoder Architecture:
───────────────────────────
Input: [CLS] Apple products [SEP] iPhone by Apple Inc [SEP]
           ↓
    Transformer Encoder (bidirectional attention)
           ↓
    Query tokens attend to Doc tokens directly
           ↓
    [CLS] representation → Linear → Sigmoid
           ↓
    Relevance Score: 0.94

Advantage: Models query-document interaction explicitly
```

### Full Architecture

```
Input Pair: (Query, Document)
       ↓
Tokenization:
[CLS] What products does Apple make? [SEP]
      Apple Inc produces iPhones, iPads, Macs [SEP]
       ↓
Token Embeddings + Position Embeddings
       ↓
┌────────────────────────────────────┐
│  Transformer Encoder (×12 layers)  │
│                                    │
│  • Multi-Head Self-Attention       │
│    (query tokens ↔ doc tokens)     │
│  • Feed-Forward Networks           │
│  • Layer Normalization             │
└────────────────────────────────────┘
       ↓
[CLS] token representation: h_CLS ∈ ℝ^768
       ↓
Classification Head: Linear(768 → 1)
       ↓
Sigmoid: → Relevance Score ∈ [0, 1]
```

---

## Models in LightRAG

### 1. Cohere Rerank (API)

**Файл**: `lightrag/rerank.py:142`

```python
async def cohere_rerank(
    query: str,
    documents: List[str],
    top_n: Optional[int] = None,
    api_key: Optional[str] = None,
    model: str = "rerank-v3.5",
    base_url: str = "https://api.cohere.com/v2/rerank",
) -> List[Dict[str, Any]]:
    """
    Cohere rerank API wrapper.

    Models:
    - rerank-v3.5 (latest, best quality)
    - rerank-english-v3.0 (English only)
    - rerank-multilingual-v3.0 (100+ languages)

    Args:
        query: Search query
        documents: List of candidate documents (from vector search)
        top_n: Number of top results to return after reranking

    Returns:
        [{"index": int, "relevance_score": float}, ...]
        Sorted by relevance_score (descending)
    """
    return await generic_rerank_api(
        query=query,
        documents=documents,
        model=model,
        base_url=base_url,
        api_key=api_key or os.getenv("COHERE_API_KEY"),
        top_n=top_n,
        response_format="cohere"
    )
```

**Model Characteristics**:

| Model | Languages | Context Length | Latency (20 docs) | Cost |
|-------|-----------|----------------|-------------------|------|
| **rerank-v3.5** | English | 4096 tokens | 150ms | $2.00 per 1K reranks |
| **rerank-english-v3.0** | English | 512 tokens | 120ms | $2.00 per 1K reranks |
| **rerank-multilingual-v3.0** | 100+ | 512 tokens | 180ms | $2.00 per 1K reranks |

**Recommendation**: `rerank-v3.5` (best quality, longer context)

**Dependencies**: `cohere>=4.0.0` (optional, for SDK usage)

### 2. Jina Reranker (API)

**Файл**: `lightrag/rerank.py:167`

```python
async def jina_rerank(
    query: str,
    documents: List[str],
    top_n: Optional[int] = None,
    api_key: Optional[str] = None,
    model: str = "jina-reranker-v2-base-multilingual",
    base_url: str = "https://api.jina.ai/v1/rerank",
) -> List[Dict[str, Any]]:
    """
    Jina AI rerank API wrapper.

    Models:
    - jina-reranker-v2-base-multilingual (89 languages)
    - jina-reranker-v1-base-en (English only)

    Args:
        query: Search query
        documents: Candidate documents
        top_n: Number of results after reranking

    Returns:
        [{"index": int, "relevance_score": float}, ...]
    """
    return await generic_rerank_api(
        query=query,
        documents=documents,
        model=model,
        base_url=base_url,
        api_key=api_key or os.getenv("JINA_API_KEY"),
        top_n=top_n,
        return_documents=False,  # Jina-specific parameter
        response_format="jina"
    )
```

**Model Characteristics**:

| Model | Languages | Context Length | Latency (20 docs) | Cost |
|-------|-----------|----------------|-------------------|------|
| **jina-reranker-v2-base-multilingual** | 89 | 8192 tokens | 100ms | $0.02 per 1K reranks |
| **jina-reranker-v1-base-en** | English | 512 tokens | 80ms | $0.02 per 1K reranks |

**Recommendation**: `jina-reranker-v2-base-multilingual` (best value, 100x cheaper than Cohere)

**Dependencies**: No SDK required (pure HTTP API)

### 3. Local Cross-Encoders (HuggingFace)

```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer
import torch

async def local_cross_encoder_rerank(
    query: str,
    documents: List[str],
    model_name: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    top_n: Optional[int] = None,
    device: str = "cpu"
) -> List[Dict[str, Any]]:
    """
    Local cross-encoder reranking with HuggingFace.

    Popular models:
    - cross-encoder/ms-marco-MiniLM-L-12-v2 (best balance)
    - cross-encoder/ms-marco-TinyBERT-L-2-v2 (fast)
    - cross-encoder/qnli-electra-base (high quality)

    Args:
        query: Search query
        documents: Candidate documents
        top_n: Number of results
        device: "cpu" or "cuda"

    Returns:
        [{"index": int, "relevance_score": float}, ...]
    """
    # Load model and tokenizer
    model = AutoModelForSequenceClassification.from_pretrained(model_name)
    tokenizer = AutoTokenizer.from_pretrained(model_name)

    model.to(device)
    model.eval()

    # Prepare input pairs
    pairs = [[query, doc] for doc in documents]

    # Tokenize
    inputs = tokenizer(
        pairs,
        padding=True,
        truncation=True,
        max_length=512,
        return_tensors="pt"
    ).to(device)

    # Inference
    with torch.no_grad():
        logits = model(**inputs).logits.squeeze(-1)

    # Convert to scores
    scores = torch.sigmoid(logits).cpu().numpy()

    # Create results
    results = [
        {"index": i, "relevance_score": float(score)}
        for i, score in enumerate(scores)
    ]

    # Sort by score
    results.sort(key=lambda x: x["relevance_score"], reverse=True)

    # Return top_n
    if top_n is not None:
        results = results[:top_n]

    return results
```

**Model Characteristics**:

| Model | Parameters | Memory | Latency (20 docs, GPU) | Quality |
|-------|-----------|--------|------------------------|---------|
| **ms-marco-MiniLM-L-12-v2** | 33M | 130 MB | 50ms | Good |
| **ms-marco-TinyBERT-L-2-v2** | 4M | 16 MB | 15ms | Decent |
| **qnli-electra-base** | 110M | 420 MB | 120ms | Excellent |

**Dependencies**: `transformers>=4.30.0`, `torch>=2.0.0`

---

## Generic Rerank API

**Файл**: `lightrag/rerank.py:30`

```python
async def generic_rerank_api(
    query: str,
    documents: List[str],
    model: str,
    base_url: str,
    api_key: Optional[str],
    top_n: Optional[int] = None,
    return_documents: Optional[bool] = None,
    extra_body: Optional[Dict[str, Any]] = None,
    response_format: str = "standard",  # "standard", "cohere", "jina"
    request_format: str = "standard",
) -> List[Dict[str, Any]]:
    """
    Unified rerank API for multiple providers.

    Standardizes request/response formats across Cohere, Jina, and others.

    Args:
        query: Search query
        documents: List of candidate documents
        model: Model name (provider-specific)
        base_url: API endpoint
        api_key: Authentication key
        top_n: Number of results
        response_format: Output format ("standard", "cohere", "jina")

    Returns:
        [{"index": int, "relevance_score": float}, ...]
    """
    import aiohttp

    # Build request
    payload = {
        "model": model,
        "query": query,
        "documents": documents,
    }

    if top_n is not None:
        payload["top_n"] = top_n

    if return_documents is not None:
        payload["return_documents"] = return_documents

    if extra_body:
        payload.update(extra_body)

    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }

    # API call
    async with aiohttp.ClientSession() as session:
        async with session.post(base_url, headers=headers, json=payload) as response:
            if response.status != 200:
                error_text = await response.text()
                raise aiohttp.ClientResponseError(
                    request_info=response.request_info,
                    history=response.history,
                    status=response.status,
                    message=f"Rerank API error: {error_text}"
                )

            response_json = await response.json()

    # Parse response based on format
    if response_format == "cohere":
        # Cohere format: {"results": [{"index": 0, "relevance_score": 0.95}, ...]}
        results = response_json.get("results", [])

    elif response_format == "jina":
        # Jina format: {"data": [{"index": 0, "score": 0.95}, ...]}
        results = [
            {"index": item["index"], "relevance_score": item["score"]}
            for item in response_json.get("data", [])
        ]

    else:
        # Standard format
        results = response_json.get("results", [])

    return results
```

---

## Integration with LightRAG Query Pipeline

**Файл**: `lightrag/utils.py:2276`

```python
async def apply_rerank_if_enabled(
    query: str,
    retrieved_docs: list[dict],
    global_config: dict,
    enable_rerank: bool = True,
    top_n: int = None,
) -> list[dict]:
    """
    Apply reranking if enabled and configured.

    Process:
    1. Check if reranking enabled
    2. Extract document contents
    3. Call rerank API
    4. Reorder documents by relevance score

    Args:
        query: Search query
        retrieved_docs: Results from vector search (stage 1)
        global_config: Contains rerank configuration
        enable_rerank: Enable/disable reranking
        top_n: Number of results after reranking

    Returns:
        Reranked documents (or original if disabled)
    """
    if not enable_rerank:
        logger.debug("Reranking disabled by query parameter")
        return retrieved_docs

    # Check if rerank function configured
    rerank_func = global_config.get("rerank_func")
    if not rerank_func:
        logger.debug("No rerank function configured, skipping reranking")
        return retrieved_docs

    # Extract document contents
    documents = [doc.get("content", "") for doc in retrieved_docs]

    if not documents:
        return retrieved_docs

    # Call rerank API
    try:
        rerank_results = await rerank_func(
            query=query,
            documents=documents,
            top_n=top_n or len(documents)
        )

        # Reorder documents by relevance score
        reranked_docs = []
        for result in rerank_results:
            idx = result["index"]
            doc = retrieved_docs[idx].copy()
            doc["rerank_score"] = result["relevance_score"]
            reranked_docs.append(doc)

        logger.info(
            f"Reranked {len(retrieved_docs)} docs → {len(reranked_docs)} top results"
        )

        return reranked_docs

    except Exception as e:
        logger.warning(f"Reranking failed: {e}, returning original results")
        return retrieved_docs
```

**Usage in Query Modes**:

```python
# Naive mode with reranking
async def naive_query_with_rerank(
    query: str,
    chunks_vdb: BaseVectorStorage,
    global_config: dict
):
    # Stage 1: Vector search (broad retrieval)
    chunks = await chunks_vdb.query(query, top_k=100)

    # Stage 2: Reranking (precise refinement)
    if global_config.get("enable_rerank"):
        chunks = await apply_rerank_if_enabled(
            query=query,
            retrieved_docs=chunks,
            global_config=global_config,
            top_n=20  # Refine to top-20
        )

    return chunks
```

---

## Training Paradigm: Classification Fine-Tuning

### Training Data

```
Format: (query, document, label)

Examples:
("What products does Apple make?", "Apple Inc produces iPhones", 1)  # Relevant
("What products does Apple make?", "Samsung makes Galaxy phones", 0)  # Not relevant

Dataset: MS MARCO (Microsoft Machine Reading Comprehension)
- 8.8M query-document pairs
- Binary relevance labels (0 or 1)
```

### Training Process

```
1. Pre-training (same as bi-encoder):
   Task: Masked Language Modeling (MLM)
   Data: Wikipedia, Common Crawl

2. Fine-tuning for reranking:
   Input: [CLS] query [SEP] document [SEP]
   Output: Relevance score ∈ [0, 1]

   Loss: Binary Cross-Entropy
   L = -[y * log(ŷ) + (1-y) * log(1-ŷ)]

   where:
   y = ground truth label (0 or 1)
   ŷ = model prediction (sigmoid output)

3. Optimization:
   Optimizer: AdamW
   Learning rate: 2e-5
   Batch size: 16-32
   Epochs: 3-5
```

### Data Augmentation

```python
# Hard negative mining
query = "Apple products"
positive_doc = "Apple Inc produces iPhones"  # Label: 1

# Easy negative (random)
easy_negative = "Banana nutrition facts"  # Label: 0

# Hard negative (similar but irrelevant)
hard_negative = "Orange fruit benefits"  # Label: 0 (confusing: also fruit)

# Training with hard negatives improves discrimination
```

---

## Связь с Vector Algorithms

Reranking refines vector search results:

**Файл**: spec/vectors/04-reranking.md

```
Two-Stage Retrieval:
─────────────────────

Stage 1 (Vector Search):
Query: "Apple products"
  ↓
Bi-encoder embedding: [0.23, -0.56, ...]
  ↓
Top-100 candidates (cosine similarity)

Stage 2 (Reranking):
Query + Each candidate → Cross-encoder
  ↓
[("Apple products", "iPhone by Apple"), ...]  → Relevance scores
  ↓
Top-20 refined results
```

**Interaction**: Vector search = **recall** (find candidates), reranking = **precision** (refine best matches).

---

## Связь с Semantic Traversal Methods

Reranking improves Local and Hybrid modes:

**Файл**: spec/research/04-semantic-traversal-methods.md

### Local Mode + Reranking

```python
# Step 1: Vector search entities
entities = await entities_vdb.query(query, top_k=50)

# Step 2: Rerank entities (optional)
if enable_rerank:
    entity_descriptions = [e["description"] for e in entities]
    reranked_results = await rerank_func(query, entity_descriptions, top_n=10)

    # Keep top-10 reranked entities
    entities = [entities[r["index"]] for r in reranked_results]

# Step 3: Graph expansion from top entities
subgraph = await graph.bfs(entities[:10], depth=2)
```

**Benefit**: More relevant seed entities → better graph expansion.

### Hybrid Mode + Reranking

```python
# Combine vector + graph signals
hybrid_chunks = []

for chunk in candidate_chunks:
    vector_score = chunk["vector_similarity"]  # From bi-encoder
    graph_score = chunk["entity_degree"] / max_degree

    # Hybrid score
    combined_score = 0.7 * vector_score + 0.3 * graph_score
    hybrid_chunks.append((chunk, combined_score))

# Sort by hybrid score
hybrid_chunks.sort(key=lambda x: x[1], reverse=True)
top_chunks = [c for c, _ in hybrid_chunks[:100]]

# Rerank top-100 with cross-encoder
if enable_rerank:
    chunk_contents = [c["content"] for c in top_chunks]
    reranked_results = await rerank_func(query, chunk_contents, top_n=20)

    final_chunks = [top_chunks[r["index"]] for r in reranked_results]
```

**Benefit**: Best of both worlds (hybrid + precise reranking).

---

## Performance Characteristics

### Latency (20 documents)

| Model | Latency | Cost per 1K reranks |
|-------|---------|---------------------|
| **Cohere rerank-v3.5** | 150ms | $2.00 |
| **Jina reranker-v2** | 100ms | $0.02 |
| **Local ms-marco-MiniLM (GPU)** | 50ms | $0 (infra cost) |
| **Local ms-marco-MiniLM (CPU)** | 300ms | $0 |

### Throughput

```
API models (parallel requests):
- Cohere: ~130 reranks/s (10 concurrent, 20 docs each)
- Jina: ~200 reranks/s (10 concurrent)

Local models (single GPU, A100):
- ms-marco-MiniLM-L-12: ~400 reranks/s (batch size 32)
- ms-marco-TinyBERT-L-2: ~1300 reranks/s (batch size 64)
```

### Quality Comparison (NDCG@10)

```
Benchmark: TREC Deep Learning 2020

Bi-encoder only (no reranking):
- all-mpnet-base-v2: 0.52

Bi-encoder + Reranking:
- all-mpnet + Cohere rerank-v3.5: 0.68 (+31%)
- all-mpnet + Jina reranker-v2: 0.64 (+23%)
- all-mpnet + ms-marco-MiniLM: 0.61 (+17%)

Conclusion: Reranking significantly improves retrieval quality
```

---

## Trade-offs and Recommendations

### When to Use Reranking

**✅ Use Reranking**:
- **High-precision needs**: Critical queries (e.g., medical, legal)
- **Complex queries**: Multi-faceted questions requiring nuanced matching
- **Ambiguous queries**: Where bi-encoder struggles (e.g., "Apple" = company vs fruit)
- **Top-K refinement**: Refining 100 candidates → 10-20 best results

**❌ Skip Reranking**:
- **Bulk queries**: Background indexing, batch processing (speed priority)
- **Simple queries**: Single-entity lookups ("What is iPhone?")
- **Low latency requirements**: Real-time autocomplete, instant search
- **Cost-sensitive**: Reranking adds 2-100x cost

### Model Selection

**Best quality**: Cohere rerank-v3.5
- Pros: State-of-the-art quality, long context (4096 tokens)
- Cons: $2/1K reranks (expensive)
- Use case: Critical queries, production systems with budget

**Best value**: Jina reranker-v2-base-multilingual
- Pros: 100x cheaper ($0.02/1K), multilingual, long context (8192 tokens)
- Cons: Slightly lower quality than Cohere
- Use case: Default choice for most applications

**Best control**: Local ms-marco-MiniLM-L-12-v2
- Pros: No API cost, privacy, no rate limits
- Cons: Requires GPU for speed, infrastructure overhead
- Use case: Privacy-sensitive, high-volume reranking

---

## Example: Full Reranking Pipeline

```python
# Configuration
from lightrag.rerank import jina_rerank

global_config = {
    "rerank_func": jina_rerank,
    "enable_rerank": True
}

query = "What products does Apple make?"

# Stage 1: Vector search (broad retrieval)
vector_results = await chunks_vdb.query(query, top_k=100)
# 100 candidates retrieved by cosine similarity

# Stage 2: Rerank (precise refinement)
reranked_results = await apply_rerank_if_enabled(
    query=query,
    retrieved_docs=vector_results,
    global_config=global_config,
    top_n=20  # Refine to top-20
)

# Result: 20 highest-quality chunks
for i, chunk in enumerate(reranked_results[:5]):
    print(f"{i+1}. Rerank Score: {chunk['rerank_score']:.3f}")
    print(f"   Vector Score: {chunk['distance']:.3f}")
    print(f"   Content: {chunk['content'][:100]}...")

# Output:
# 1. Rerank Score: 0.985
#    Vector Score: 0.876
#    Content: Apple Inc produces iPhone, iPad, Mac computers, Apple Watch, and AirPods...
# 2. Rerank Score: 0.921
#    Vector Score: 0.853
#    Content: iPhone is Apple's flagship smartphone product line...
# 3. Rerank Score: 0.887
#    Vector Score: 0.791
#    Content: MacBook laptops are manufactured by Apple Inc...
```

**Observation**: Reranking reorders results — some low vector scores moved higher due to better query-document interaction.

---

**Версия**: 1.0
**Дата**: 2025-01-13
