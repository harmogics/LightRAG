# LLM Integration: Интеграция Языковых Моделей

## Обзор

Библиотеки для взаимодействия с большими языковыми моделями (LLMs). Эти зависимости обеспечивают **ядро семантической обработки** в LightRAG - entity extraction, summarization, keywords extraction, answer generation.

## Core LLM Libraries

### openai

**Версия**: Latest (динамически устанавливается через pipmaster)
**Лицензия**: Apache 2.0
**Сайт**: https://github.com/openai/openai-python

#### Назначение

Официальный Python клиент для OpenAI API. Используется для:
- **GPT-4 / GPT-3.5-turbo**: Text generation, entity extraction, summarization
- **text-embedding-ada-002**: Semantic embeddings
- **text-embedding-3-small/large**: Modern embeddings

#### Использование в LightRAG

**Файл**: `lightrag/llm/openai.py`

```python
from openai import AsyncOpenAI, APIConnectionError, RateLimitError

class OpenAIWrapper:
    """
    Async wrapper for OpenAI API.

    Features:
    • Async chat completions
    • Async embeddings
    • Retry logic (via tenacity)
    • Streaming support
    """

    def __init__(self, api_key, model="gpt-4o-mini", ...):
        self.client = AsyncOpenAI(api_key=api_key)
        self.model = model

    async def generate(self, prompt, system_prompt=None, **kwargs):
        """
        Generate response from GPT model.

        Used for:
        • Entity extraction (P1+P2)
        • Gleaning (P3)
        • Summarization (P4)
        • Keywords (P5)
        • Answer generation (P6)
        """
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": prompt}
            ],
            temperature=kwargs.get("temperature", 0.0),
            ...
        )
        return response.choices[0].message.content

    async def embed(self, texts: list[str]):
        """
        Generate embeddings for texts.

        Used for:
        • Entity embeddings (for vector search)
        • Chunk embeddings (for similarity search)
        • Query embeddings (for entity resolution)
        """
        response = await self.client.embeddings.create(
            model="text-embedding-ada-002",
            input=texts
        )
        return [item.embedding for item in response.data]
```

**Роль в семантической обработке**:
```
Text → [OpenAI GPT] → Entities + Relations  (Extraction)
Multiple Descriptions → [OpenAI GPT] → Summary  (Consolidation)
Query → [OpenAI GPT] → Keywords  (Intent Decomposition)
Context → [OpenAI GPT] → Answer  (Synthesis)

Text → [OpenAI Embedding] → Vector  (Projection to semantic space)
```

---

### anthropic

**Версия**: Latest
**Лицензия**: MIT
**Сайт**: https://github.com/anthropics/anthropic-sdk-python

#### Назначение

Официальный Python клиент для Anthropic Claude API. Альтернатива OpenAI с лучшим reasoning для сложных задач.

#### Использование в LightRAG

**Файл**: `lightrag/llm/anthropic.py`

```python
from anthropic import AsyncAnthropic

async def claude_model_complete(
    prompt,
    system_prompt=None,
    model="claude-3-5-sonnet-20241022",
    **kwargs
):
    """
    Claude API wrapper.

    Models:
    • claude-3-5-sonnet: Best reasoning
    • claude-3-haiku: Fast, cost-effective
    • claude-3-opus: Maximum capability
    """
    client = AsyncAnthropic(api_key=api_key)

    response = await client.messages.create(
        model=model,
        max_tokens=kwargs.get("max_tokens", 4096),
        temperature=kwargs.get("temperature", 0.0),
        system=system_prompt,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

**Преимущества Claude**:
- Больший context window (200K tokens vs 128K)
- Лучший reasoning для entity extraction
- Более точный JSON output

---

### tenacity

**Версия**: Latest
**Лицензия**: Apache 2.0
**Сайт**: https://github.com/jd/tenacity

#### Назначение

Библиотека для retry logic с exponential backoff. Критична для надежности LLM вызовов.

#### Использование в LightRAG

**Файл**: `lightrag/llm/openai.py`, `lightrag/llm/anthropic.py`

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60),
    retry=retry_if_exception_type((
        APIConnectionError,
        RateLimitError,
        APITimeoutError,
    ))
)
async def llm_call_with_retry(prompt, **kwargs):
    """
    LLM call with automatic retry.

    Retry Strategy:
    • Max 3 attempts
    • Exponential backoff: 4s, 8s, 16s, ...
    • Only retry on transient errors
    """
    response = await llm_func(prompt, **kwargs)
    return response
```

**Retry Strategy**:
```
Attempt 1 → Fails (RateLimitError)
    ↓ Wait 4 seconds
Attempt 2 → Fails (APIConnectionError)
    ↓ Wait 8 seconds
Attempt 3 → Success
    ↓
Return response
```

**Обрабатываемые ошибки**:
- `APIConnectionError` - Network issues
- `RateLimitError` - Rate limit exceeded
- `APITimeoutError` - Request timeout

---

### pipmaster

**Версия**: Latest
**Лицензия**: MIT
**Сайт**: https://github.com/ParisNeo/pipmaster

#### Назначение

Dynamic library installation. Автоматически устанавливает dependencies при первом использовании.

#### Использование в LightRAG

**Файл**: Multiple `lightrag/llm/*.py` files

```python
import pipmaster as pm

# Check if library is installed
if not pm.is_installed("openai"):
    # Dynamically install if not present
    pm.install("openai")

# Now safe to import
from openai import AsyncOpenAI
```

**Lazy Import Pattern**:
```python
# In lightrag/llm/openai.py
if not pm.is_installed("openai"):
    pm.install("openai")

# In lightrag/llm/anthropic.py
if not pm.is_installed("anthropic"):
    pm.install("anthropic")

# In lightrag/llm/ollama.py
if not pm.is_installed("ollama"):
    pm.install("ollama")
```

**Преимущества**:
- Минимальная базовая установка (`pip install lightrag-hku`)
- Auto-install при первом использовании
- Не требует manual dependency management

**Недостатки**:
- Может вызвать задержку при первом запуске
- Требует internet connection

---

## Additional LLM Providers

LightRAG поддерживает множество LLM providers через lazy imports:

### Ollama (Local LLMs)

**Файл**: `lightrag/llm/ollama.py`

```python
from ollama import AsyncClient

async def ollama_model_complete(prompt, model="llama3.1", **kwargs):
    """
    Local LLM via Ollama.

    Models:
    • llama3.1, llama3.2 (Meta)
    • mistral, mixtral (Mistral AI)
    • qwen2.5 (Alibaba)
    """
    client = AsyncClient(host=host)
    response = await client.chat(
        model=model,
        messages=[{"role": "user", "content": prompt}]
    )
    return response["message"]["content"]
```

**Use Case**: Privacy-sensitive deployments, offline operation, cost reduction.

### Azure OpenAI

**Файл**: `lightrag/llm/azure_openai.py`

```python
from openai import AsyncAzureOpenAI

async def azure_openai_complete(prompt, deployment_name, **kwargs):
    """Enterprise OpenAI deployment"""
    client = AsyncAzureOpenAI(
        azure_endpoint=endpoint,
        api_key=api_key,
        api_version="2024-02-01"
    )
    ...
```

**Use Case**: Enterprise deployments with Azure infrastructure.

### AWS Bedrock

**Файл**: `lightrag/llm/bedrock.py`

```python
import boto3

async def bedrock_complete(prompt, model_id="anthropic.claude-v2", **kwargs):
    """AWS Bedrock LLMs"""
    client = boto3.client("bedrock-runtime", region_name=region)
    ...
```

**Use Case**: AWS-native deployments, Claude via Bedrock.

### HuggingFace

**Файл**: `lightrag/llm/hf.py`

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

async def hf_model_complete(prompt, model_name="meta-llama/Llama-2-7b", **kwargs):
    """HuggingFace models"""
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModelForCausalLM.from_pretrained(model_name)
    ...
```

**Use Case**: Custom/fine-tuned models, research.

---

## LLM Usage Patterns in LightRAG

### Pattern 1: Entity Extraction (P1+P2)

```python
system_prompt = PROMPTS["entity_extraction_system_prompt"]
user_prompt = PROMPTS["entity_extraction_user_prompt"].format(
    input_text=chunk_content,
    entity_types=["person", "organization", "location", ...],
    ...
)

response = await llm_func(
    user_prompt,
    system_prompt=system_prompt,
    temperature=0.0  # Deterministic
)

entities, relations = parse_extraction_output(response)
```

**LLM Role**: Semantic attractor identification in unstructured text.

### Pattern 2: Gleaning (P3)

```python
conversation_history = [
    {"role": "user", "content": chunk_text},
    {"role": "assistant", "content": initial_extraction}
]

gleaning_prompt = PROMPTS["entity_extraction_gleaning_prompt"].format(
    conversation_history=format_conversation(conversation_history)
)

additional_entities = await llm_func(gleaning_prompt, temperature=0.0)
```

**LLM Role**: Multi-turn self-refinement of extractors.

### Pattern 3: Summarization (P4)

```python
prompt = PROMPTS["summarize_entity_descriptions"].format(
    entity_name="Apple Inc",
    description_list="\n".join([
        '{"Description": "Founded 1976..."}',
        '{"Description": "Headquarters in Cupertino..."}',
        ...
    ]),
    summary_length=500,
    language="English"
)

summary = await llm_func(prompt, temperature=0.0)
```

**LLM Role**: Map-reduce consolidation of manifestations.

### Pattern 4: Keywords Extraction (P5)

```python
prompt = PROMPTS["keywords_extraction"].format(
    query="What innovative products does Apple make?",
    examples=examples,
    language="English"
)

response = await llm_func(prompt, keyword_extraction=True, temperature=0.0)
keywords = json.loads(response)
# {"high_level_keywords": [...], "low_level_keywords": [...]}
```

**LLM Role**: Hierarchical semantic compass generation.

### Pattern 5: Answer Generation (P6)

```python
context = format_rag_context(entities, relations, chunks)

prompt = PROMPTS["rag_response"].format(
    query=user_query,
    context=context
)

answer = await llm_func(
    prompt,
    temperature=0.7  # Creative synthesis
)
```

**LLM Role**: Grounded synthesis from attractor constellation.

---

## Cost Optimization

### Token Usage by Operation

| Operation | Avg Input Tokens | Avg Output Tokens | Cost (GPT-4o-mini) |
|-----------|------------------|-------------------|-------------------|
| **Entity Extraction** | 1,500 | 500 | $0.0003 per chunk |
| **Gleaning** | 2,000 | 300 | $0.0004 per round |
| **Summarization** | 3,000 | 200 | $0.0005 per entity |
| **Keywords** | 100 | 50 | $0.00002 per query |
| **Answer Generation** | 4,000 | 500 | $0.0007 per query |

### Cost Reduction Strategies

```python
# 1. Use smaller models for simple tasks
keywords_model = "gpt-4o-mini"  # Cheaper
extraction_model = "gpt-4o"     # More expensive but better

# 2. Cache LLM responses
llm_response_cache = BaseKVStorage()
if cached_response := await cache.get(prompt_hash):
    return cached_response

# 3. Batch operations
responses = await asyncio.gather(*[
    llm_func(prompt) for prompt in prompts
])
```

---

## Performance Characteristics

### Latency

| Provider | Model | Avg Latency | Throughput |
|----------|-------|-------------|------------|
| **OpenAI** | gpt-4o-mini | 0.5-1.5s | 60 req/min |
| **OpenAI** | gpt-4o | 1-3s | 10 req/min (tier 1) |
| **Anthropic** | claude-3-haiku | 0.8-2s | 50 req/min |
| **Anthropic** | claude-3-5-sonnet | 2-5s | 50 req/min |
| **Ollama** | llama3.1:8b | 0.1-0.5s | Unlimited (local) |

### Async Parallel Processing

```python
# Sequential (slow)
for chunk in chunks:
    entities = await extract_entities(chunk)  # 2s each
# Total: 2s × 100 chunks = 200s

# Parallel with semaphore (fast)
semaphore = asyncio.Semaphore(16)  # Max 16 concurrent
tasks = [extract_with_limit(chunk, semaphore) for chunk in chunks]
results = await asyncio.gather(*tasks)
# Total: ~15s (16 × parallel + overhead)
```

---

## Error Handling

### Common Errors

```python
from openai import APIConnectionError, RateLimitError, APITimeoutError

try:
    response = await llm_func(prompt)
except RateLimitError:
    # Rate limit exceeded - wait and retry
    await asyncio.sleep(60)
    response = await llm_func(prompt)
except APIConnectionError:
    # Network error - retry with backoff
    response = await llm_func_with_retry(prompt)
except APITimeoutError:
    # Timeout - increase timeout or use streaming
    response = await llm_func(prompt, timeout=120)
```

### Validation

```python
# Validate LLM output
response = await llm_func(prompt)

try:
    data = json.loads(response)
    validate_extraction_output(data)
except json.JSONDecodeError:
    # JSON parsing failed - use json_repair
    import json_repair
    data = json_repair.loads(response)
```

---

## Related Documentation

- **[LLM Semantic Processing Patterns](../research/06-llm-semantic-processing.md)** - Conceptual patterns
- **[Prompts Documentation](../prompts/README.md)** - All prompts (P1-P9)
- **[Semantic Transformations](../transform/README.md)** - T2, T3, T5, T7, T10 use LLMs

---

**Версия**: 1.0
**Дата**: 2025-01-13
