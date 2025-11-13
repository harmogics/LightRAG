# Cache Pattern

## Overview

**Pattern Type**: Data Pattern  
**Purpose**: LLM response caching для cost reduction и speed improvement

---

## Implementation

### Cache Key Generation

```python
def compute_cache_key(
    user_prompt: str,
    system_prompt: str,
    model: str,
    temperature: float
) -> str:
    """Generate deterministic cache key"""

    # Combine all parameters that affect output
    cache_input = f"{user_prompt}|{system_prompt}|{model}|{temperature}"

    # MD5 hash for compact key
    return md5(cache_input.encode()).hexdigest()
```

### Cache Lookup

```python
async def use_llm_func_with_cache(
    user_prompt: str,
    use_llm_func: callable,
    system_prompt: str = None,
    llm_response_cache: BaseKVStorage = None,
    **kwargs
) -> str:
    """LLM call with caching"""

    if llm_response_cache is None:
        # No cache, direct call
        return await use_llm_func(user_prompt, system_prompt, **kwargs)

    # Generate cache key
    cache_key = compute_cache_key(
        user_prompt,
        system_prompt,
        kwargs.get("model", "default"),
        kwargs.get("temperature", 0.0)
    )

    # Check cache
    cached = await llm_response_cache.get_by_id(cache_key)

    if cached:
        # Cache hit!
        return cached["response"]

    # Cache miss: call LLM
    response = await use_llm_func(user_prompt, system_prompt, **kwargs)

    # Store in cache
    await llm_response_cache.upsert({
        cache_key: {
            "response": response,
            "timestamp": time.time()
        }
    })

    return response
```

---

## Cache Types

### 1. LLM Response Cache

```python
# Stores LLM responses
cache_entry = {
    "cache_key": "abc123...",
    "response": "entity<|#|>Alice<|#|>person<|#|>...",
    "timestamp": 1699876543.21,
    "model": "gpt-4o-mini",
    "prompt_hash": "def456..."
}
```

### 2. Embedding Cache

```python
# Stores text embeddings
embedding_cache = {
    "text_hash": "xyz789...",
    "embedding": [0.123, -0.456, ...],  # 768-dim vector
    "model": "text-embedding-ada-002"
}
```

---

## Cache Backends

```python
cache_backends = {
    "JsonKVStorage": "Local JSON files (simple)",
    "RedisKVStorage": "Redis (fast, distributed)",
    "MongoKVStorage": "MongoDB (persistent, queryable)",
    "PostgresKVStorage": "PostgreSQL (SQL queries)"
}
```

---

## Benefits

### Cost Savings

```python
# Without cache
100_chunks * $0.01_per_call = $1.00

# With 40% cache hit rate
60_chunks * $0.01 = $0.60
# Savings: $0.40 (40%)
```

### Speed Improvement

```python
# Without cache
llm_latency = 3_seconds
total = 100 * 3 = 300_seconds

# With 40% cache hit rate
cache_hits = 40 * 0.001 = 0.04_seconds
cache_misses = 60 * 3 = 180_seconds
total = 180.04_seconds
# Speedup: 1.67x
```

---

## See Also

- [Storage Abstraction](01-storage-abstraction.md)

