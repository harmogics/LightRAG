# Async & Networking: Асинхронность и Сетевые Операции

## asyncio (stdlib)

**Purpose**: Core async framework

```python
import asyncio

# Parallel LLM calls
tasks = [llm_func(chunk) for chunk in chunks]
results = await asyncio.gather(*tasks)

# Semaphore for rate limiting
semaphore = asyncio.Semaphore(16)
async with semaphore:
    response = await llm_func(prompt)
```

## aiohttp

**Package**: `aiohttp`
**Purpose**: Async HTTP client/server

```python
import aiohttp

async with aiohttp.ClientSession() as session:
    async with session.post(
        "https://api.openai.com/v1/chat/completions",
        headers={"Authorization": f"Bearer {api_key}"},
        json={"model": "gpt-4", "messages": messages}
    ) as response:
        data = await response.json()
```

## httpx

**Package**: `httpx`
**Purpose**: Modern async HTTP client

```python
import httpx

async with httpx.AsyncClient() as client:
    response = await client.post(
        url,
        headers=headers,
        json=payload,
        timeout=30.0
    )
    return response.json()
```

## tenacity

**Package**: `tenacity`
**Purpose**: Retry logic with exponential backoff

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60)
)
async def llm_call_with_retry(prompt):
    return await llm_func(prompt)
```

---

**Version**: 1.0
