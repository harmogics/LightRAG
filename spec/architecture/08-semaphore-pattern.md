# Semaphore Pattern

## Overview

**Pattern Type**: Concurrency Control  
**Purpose**: Limit concurrent operations (rate limiting)

---

## Problem

```python
# Without semaphore: All 1000 tasks execute immediately
tasks = [llm_call(chunk) for chunk in chunks]  # 1000 chunks
results = await asyncio.gather(*tasks)

# Problem:
# - API rate limit exceeded (OpenAI: 3500 RPM)
# - Memory exhaustion
# - Network congestion
```

---

## Solution

```python
# With semaphore: Only N tasks execute concurrently
max_concurrent = 16
semaphore = asyncio.Semaphore(max_concurrent)

async def process_with_limit(chunk):
    async with semaphore:
        # Only 16 tasks can be here simultaneously
        return await llm_call(chunk)

tasks = [process_with_limit(c) for c in chunks]
results = await asyncio.gather(*tasks)

# Result:
# - Tasks queued: 1000
# - Tasks executing: 16 (at any moment)
# - No rate limit errors
```

---

## Implementation

```python
# From lightrag/operate.py

# Get max async from config
chunk_max_async = global_config.get("llm_model_max_async", 4)
semaphore = asyncio.Semaphore(chunk_max_async)

async def _process_with_semaphore(chunk):
    async with semaphore:
        try:
            return await _process_single_content(chunk)
        except Exception as e:
            logger.error(f"Chunk processing failed: {e}")
            raise

# All chunks submitted, but throttled by semaphore
tasks = [_process_with_semaphore(c) for c in ordered_chunks]
results = await asyncio.gather(*tasks, return_exceptions=True)
```

---

## Configuration

```python
# Adjust based on API limits and performance needs
semaphore_configs = {
    "conservative": 4,   # Safe, slow
    "balanced": 16,      # Recommended
    "aggressive": 32,    # Fast, may hit limits
    "extreme": 64        # Very fast, likely errors
}
```

---

## Benefits

- **Rate limit compliance**: Stay within API quotas
- **Resource control**: Prevent memory/network exhaustion
- **Graceful degradation**: Controlled failure

---

## See Also

- [Async Pattern](07-async-pattern.md)
- [Pipeline Pattern](05-pipeline-pattern.md)

