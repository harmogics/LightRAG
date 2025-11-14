# Reranking: Переранжирование Результатов Поиска

## Концептуальная Парадигма

**Reranking** = second-stage refinement после initial vector search для improving precision.

```
Stage 1: Vector Search (Fast, Broad)
Query → Top-100 candidates (cosine similarity)

Stage 2: Reranking (Slow, Precise)
Top-100 → Cross-encoder → Top-10 refined results

Metaphor:
Stage 1 = "Быстрый отбор" (retrieve candidates)
Stage 2 = "Точная оценка" (deep semantic matching)
```

**Философия**: Bi-encoder (vector search) finds **candidates**, cross-encoder (reranking) finds **best matches**.

---

## Algorithm: Cross-Encoder Reranking

### Bi-Encoder vs Cross-Encoder

```
Bi-Encoder (Vector Search):
  Query      → Encoder → [0.23, -0.56, ...]
  Document   → Encoder → [0.25, -0.51, ...]
  Similarity = cosine([0.23...], [0.25...])

  ✅ Fast: Encode once, compare many
  ❌ Limited: Independent encodings (no interaction)

Cross-Encoder (Reranking):
  [Query, Document] → Joint Encoder → Relevance Score

  ✅ Precise: Models query-doc interaction
  ❌ Slow: Re-encode every (query, doc) pair
```

**Use Case**: Bi-encoder for retrieval (stage 1), cross-encoder for reranking (stage 2).

---

## Реализация

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
    response_format: str = "standard",  # Jina/Cohere
    request_format: str = "standard",
) -> List[Dict[str, Any]]:
    """
    Generic rerank API for cross-encoder models.

    Args:
        query: Search query
        documents: List of candidate documents (from vector search)
        model: Rerank model name
        base_url: API endpoint
        api_key: Authentication key
        top_n: Number of top results after reranking

    Returns:
        [{"index": int, "relevance_score": float}, ...]
    """
    # Build request payload
    payload = {
        "model": model,
        "query": query,
        "documents": documents,
    }

    if top_n is not None:
        payload["top_n"] = top_n

    # API call
    async with aiohttp.ClientSession() as session:
        async with session.post(base_url, headers=headers, json=payload) as response:
            if response.status != 200:
                raise aiohttp.ClientResponseError(...)

            response_json = await response.json()

            # Extract results
            results = response_json.get("results", [])

            # Standardize format
            return [
                {
                    "index": result["index"],
                    "relevance_score": result["relevance_score"]
                }
                for result in results
            ]
```

### Cohere Reranking

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
    - rerank-english-v3.0
    - rerank-multilingual-v3.0
    """
    return await generic_rerank_api(
        query=query,
        documents=documents,
        model=model,
        base_url=base_url,
        api_key=api_key or os.getenv("COHERE_API_KEY"),
        top_n=top_n,
    )
```

### Jina Reranking

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
    Jina rerank API wrapper.

    Models:
    - jina-reranker-v2-base-multilingual (best multilingual)
    - jina-reranker-v1-base-en (English only)
    """
    return await generic_rerank_api(
        query=query,
        documents=documents,
        model=model,
        base_url=base_url,
        api_key=api_key or os.getenv("JINA_API_KEY"),
        top_n=top_n,
        return_documents=False,  # Jina-specific param
    )
```

---

## Integration with LightRAG

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
    Apply reranking if enabled and rerank model configured.

    Args:
        query: Search query
        retrieved_docs: Results from vector search
        global_config: Contains rerank configuration
        enable_rerank: Enable/disable reranking
        top_n: Number of results after reranking

    Returns:
        Reranked documents (or original if reranking disabled)
    """
    if not enable_rerank:
        logger.debug("Reranking disabled by query parameter")
        return retrieved_docs

    # Check if rerank function configured
    rerank_func = global_config.get("rerank_func")
    if not rerank_func:
        logger.debug("No rerank function configured, skipping")
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

---

## Связь с Semantic Traversal Methods

Reranking улучшает **naive and local modes** (spec/research/04-semantic-traversal-methods.md):

### Naive Mode + Reranking

```python
# Step 1: Vector search (fast, broad)
chunks = await chunks_vdb.query(query, top_k=100)  # ← Bi-encoder

# Step 2: Rerank (slow, precise)
if enable_rerank:
    chunks = await apply_rerank_if_enabled(
        query, chunks, global_config, top_n=20
    )  # ← Cross-encoder

# Result: Top-20 most relevant chunks (refined from 100 candidates)
```

### Local Mode + Reranking

```python
# Step 1: Entity vector search
entities = await entities_vdb.query(query, top_k=50)

# Step 2: Rerank entities
if enable_rerank:
    entities = await rerank(query, entities, top_n=10)

# Step 3: BFS expansion from top-10 reranked entities
subgraph = await graph.bfs(entities[:10], depth=2)
```

---

## Model Comparison

### Cohere

| Model | Languages | Latency | Quality | Cost |
|-------|-----------|---------|---------|------|
| **rerank-v3.5** | English | ~150ms/20 docs | Best | $2/1K reranks |
| **rerank-english-v3.0** | English | ~120ms | Good | $2/1K |
| **rerank-multilingual-v3.0** | 100+ langs | ~180ms | Good | $2/1K |

### Jina

| Model | Languages | Latency | Quality | Cost |
|-------|-----------|---------|---------|------|
| **jina-reranker-v2-base-multilingual** | 89 langs | ~100ms | Good | $0.02/1K reranks |
| **jina-reranker-v1-base-en** | English | ~80ms | Decent | $0.02/1K |

**Selection**:
- **Best quality**: Cohere rerank-v3.5
- **Multilingual**: Jina v2-base-multilingual or Cohere multilingual-v3.0
- **Cost-effective**: Jina (100x cheaper than Cohere)

---

## Производительность

### Benchmark (20 documents reranking)

| Provider | Model | Time | Cost (per 1K reranks) |
|----------|-------|------|----------------------|
| **Cohere** | rerank-v3.5 | 150ms | $2.00 |
| **Jina** | v2-base-multilingual | 100ms | $0.02 |
| **Local** | Cross-encoder (GPU) | 50ms | $0 (infrastructure cost) |

### Impact on Query Latency

```
Without Reranking:
Vector search: 50ms
Total: 50ms

With Reranking (20 docs):
Vector search: 50ms (top-100)
Reranking: 150ms (top-20)
Total: 200ms (4x slower)
```

**Recommendation**:
- Use reranking for **critical queries** (high precision needed)
- Disable for **bulk/background queries** (speed priority)
- Pre-filter candidates to 20-50 docs before reranking (avoid reranking 100+ docs)

---

## Example: Full Reranking Pipeline

```python
# Configuration
global_config = {
    "rerank_func": cohere_rerank,  # or jina_rerank
    "rerank_model": "rerank-v3.5",
}

query = "What products does Apple make?"

# Step 1: Vector search (broad retrieval)
vector_results = await chunks_vdb.query(query, top_k=100)
# 100 candidates retrieved by cosine similarity

# Step 2: Rerank (precise refinement)
reranked_results = await apply_rerank_if_enabled(
    query=query,
    retrieved_docs=vector_results,
    global_config=global_config,
    enable_rerank=True,
    top_n=20  # Refine to top-20
)

# Result: 20 highest-quality chunks
for i, chunk in enumerate(reranked_results[:5]):
    print(f"{i+1}. Score: {chunk['rerank_score']:.3f}")
    print(f"   Content: {chunk['content'][:100]}...")

# Output:
# 1. Score: 0.985
#    Content: Apple Inc produces iPhone, iPad, Mac computers...
# 2. Score: 0.921
#    Content: iPhone is Apple's flagship smartphone product...
# 3. Score: 0.887
#    Content: MacBook laptops are manufactured by Apple...
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
