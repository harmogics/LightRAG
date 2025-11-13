# Async/Await Pattern

## Overview

**Pattern Type**: Concurrency  
**Purpose**: Non-blocking I/O для LLM calls, DB operations, network requests

---

## Why Async in LightRAG?

### I/O-Bound Operations

```python
io_operations = {
    "LLM API calls": "1-10 seconds per call",
    "Vector DB queries": "100-500ms",
    "Graph DB operations": "50-200ms",
    "File I/O": "10-100ms",
    "Embedding generation": "10-100ms"
}

# Without async: Sequential execution
total_time_sync = sum(all_operations)  # 50+ seconds

# With async: Parallel execution
total_time_async = max(all_operations)  # 10 seconds (limited by slowest)
```

### High Concurrency Needs

```python
# Document processing
document_chunks = 100  # Typical document

# Sequential (sync)
for chunk in chunks:
    entities = extract_entities(chunk)  # 3s per chunk
# Total: 100 * 3s = 300s = 5 minutes

# Parallel (async)
tasks = [extract_entities(chunk) for chunk in chunks]
results = await asyncio.gather(*tasks)
# Total: ~10s (with proper semaphore limiting)
```

---

## Implementation

### Async Function Signature

```python
# All I/O operations are async
async def extract_entities(
    chunks: dict[str, TextChunkSchema],
    global_config: dict,
    llm_response_cache: BaseKVStorage,
    text_chunks_storage: BaseKVStorage
) -> list:
    """Async entity extraction"""

    # Async operations
    entity_list = []

    for chunk_id, chunk_data in chunks.items():
        # Async LLM call
        response = await use_llm_func_with_cache(
            user_prompt,
            system_prompt=system_prompt,
            llm_response_cache=llm_response_cache
        )

        # Parse and collect
        entities, relations = parse_output(response)
        entity_list.extend(entities)

    return entity_list
```

### Concurrent Processing with Gather

```python
# From lightrag/operate.py

async def _process_single_content(chunk_key_dp):
    """Process one chunk"""
    chunk_key, chunk_dp = chunk_key_dp
    content = chunk_dp["content"]

    # LLM call for extraction
    response = await use_llm_func(...)

    return parse(response)

# Process all chunks concurrently
tasks = [
    _process_single_content(chunk)
    for chunk in ordered_chunks
]

# Wait for all to complete
results = await asyncio.gather(*tasks)
```

---

## Concurrency Control

### Semaphore Pattern

```python
# Limit concurrent LLM calls
max_async = global_config.get("llm_model_max_async", 4)
semaphore = asyncio.Semaphore(max_async)

async def _process_with_semaphore(chunk):
    async with semaphore:
        # Only max_async tasks execute simultaneously
        return await _process_single_content(chunk)

# All chunks queued, but only N execute at once
tasks = [_process_with_semaphore(c) for c in chunks]
results = await asyncio.gather(*tasks)
```

**Why Semaphore?**:
- Prevent API rate limits (OpenAI: 3500 RPM)
- Control resource usage (memory, CPU)
- Graceful degradation

---

## Benefits

### 1. Throughput

```python
# Benchmark: 100 chunks
throughput_comparison = {
    "sync_sequential": {
        "time": "300s",
        "chunks_per_second": 0.33
    },
    "async_parallel_unlimited": {
        "time": "3s",  # All at once, but may fail (rate limits)
        "chunks_per_second": 33.3
    },
    "async_parallel_semaphore_4": {
        "time": "75s",  # 4 at a time
        "chunks_per_second": 1.33,
        "speedup": "4x"
    },
    "async_parallel_semaphore_16": {
        "time": "19s",  # 16 at a time
        "chunks_per_second": 5.26,
        "speedup": "16x"
    }
}
```

### 2. Resource Efficiency

```python
# Sync: Blocking
def sync_process():
    for i in range(100):
        time.sleep(1)  # Blocking! CPU idle
    # Total: 100s of idle time

# Async: Non-blocking
async def async_process():
    tasks = [asyncio.sleep(1) for i in range(100)]
    await asyncio.gather(*tasks)
    # Total: 1s, CPU can do other work
```

### 3. Responsiveness

```python
# API server remains responsive
@app.post("/insert")
async def insert_document(doc: Document):
    # Non-blocking insertion
    track_id = await rag.ainsert(doc.content)

    # Can handle other requests while processing
    return {"track_id": track_id}
```

---

## Async Storage Operations

```python
# All storage operations are async
class BaseVectorStorage(ABC):
    @abstractmethod
    async def query(self, query: str, top_k: int) -> list[dict]:
        """Async query"""

    @abstractmethod
    async def upsert(self, data: dict) -> None:
        """Async upsert"""

# Usage
results = await vector_db.query("search term", top_k=10)
await vector_db.upsert({"id1": {"content": "data"}})
```

---

## Event Loop Management

```python
# From lightrag/utils.py

def always_get_an_event_loop() -> asyncio.AbstractEventLoop:
    """
    Get or create event loop

    Handles:
    - Existing loop in async context
    - New loop in sync context
    - Loop closed scenarios
    """
    try:
        # Try to get current loop
        loop = asyncio.get_event_loop()
        if loop.is_closed():
            raise RuntimeError("Event loop is closed")
        return loop
    except RuntimeError:
        # Create new loop
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        return loop

# Usage: Sync wrapper for async functions
def insert(self, doc: str):
    """Sync wrapper"""
    loop = always_get_an_event_loop()
    return loop.run_until_complete(self.ainsert(doc))
```

---

## Best Practices

### 1. Always Await

```python
# ✅ Good
result = await async_function()

# ❌ Bad: Creates coroutine but doesn't execute
result = async_function()  # Returns coroutine object, not result!
```

### 2. Use Gather for Multiple Tasks

```python
# ✅ Good: Concurrent
results = await asyncio.gather(
    task1(),
    task2(),
    task3()
)

# ❌ Bad: Sequential
result1 = await task1()
result2 = await task2()
result3 = await task3()
```

### 3. Semaphore for Rate Limiting

```python
# ✅ Good: Controlled concurrency
semaphore = asyncio.Semaphore(10)

async with semaphore:
    result = await expensive_operation()

# ❌ Bad: Unlimited concurrency (may crash)
results = await asyncio.gather(*[expensive_operation() for _ in range(1000)])
```

---

## Related Patterns

- **Semaphore Pattern**: Rate limiting
- **Pipeline Pattern**: Async stages
- **Map-Reduce**: Parallel map phase

---

## See Also

- [Semaphore Pattern](08-semaphore-pattern.md)
- [Pipeline Pattern](05-pipeline-pattern.md)

