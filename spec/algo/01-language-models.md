# Language Models: Большие Языковые Модели

## Концептуальная Парадигма

**Language models (LLMs)** = autoregressive transformers for **text generation** and **semantic understanding**.

```
Input Text → Tokenization → Transformer Decoder → Output Text
"Extract entities from: Apple Inc produces iPhones"
                      ↓
         [Entity Extraction Prompt]
                      ↓
              GPT-4 / Claude
                      ↓
"entity<|#|>Apple Inc<|#|>organization<|#|>..."
```

**Философия**: LLMs as **semantic processors** — не просто generation, но structured information extraction и reasoning.

---

## Architecture: Autoregressive Transformers

### Decoder-Only Transformer

```
Input: "Apple Inc produces"
       ↓
Tokenization: [15496, 3457, 19159]
       ↓
Embedding Layer: → ℝ^4096
       ↓
┌────────────────────────────────────┐
│   Transformer Decoder Layers (×32) │
│                                    │
│   • Self-Attention (causal mask)   │
│   • Feed-Forward Networks          │
│   • Layer Normalization            │
│   • Residual Connections           │
└────────────────────────────────────┘
       ↓
Output Logits: → Vocabulary (50K-100K tokens)
       ↓
Next Token Prediction: "iPhones" (probability distribution)
```

**Key Properties**:
- **Causal masking**: Each token attends only to previous tokens
- **Autoregressive**: Generate one token at a time (sequential)
- **Context window**: 4K-128K tokens (model-dependent)

---

## Models in LightRAG

### 1. OpenAI GPT Models

**Файл**: `lightrag/llm/openai.py:51`

```python
async def openai_complete_if_cache(
    model: str,
    prompt: str,
    system_prompt: str | None = None,
    history_messages: list | None = None,
    **kwargs,
) -> str:
    """
    OpenAI completion with caching and retry logic.

    Supported models:
    - gpt-4-turbo (128K context)
    - gpt-4 (8K context)
    - gpt-3.5-turbo (16K context)

    Args:
        model: Model name
        prompt: User prompt
        system_prompt: Optional system instruction
        history_messages: Optional conversation history

    Returns:
        Generated text
    """
    openai_async_client = kwargs.pop("openai_async_client", None)

    # Build messages
    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})

    if history_messages:
        messages.extend(history_messages)

    messages.append({"role": "user", "content": prompt})

    # API call with retry logic
    response = await openai_async_client.chat.completions.create(
        model=model,
        messages=messages,
        **kwargs
    )

    return response.choices[0].message.content
```

**Model Characteristics**:

| Model | Context | Cost (per 1M tokens) | Quality | Speed |
|-------|---------|---------------------|---------|-------|
| **gpt-4-turbo** | 128K | $10 (in) + $30 (out) | Best | Medium |
| **gpt-4** | 8K | $30 (in) + $60 (out) | Best | Slow |
| **gpt-3.5-turbo** | 16K | $0.50 (in) + $1.50 (out) | Good | Fast |

**Dependencies**: `openai>=1.0.0`

### 2. Anthropic Claude Models

**Файл**: `lightrag/llm/anthropic_impl.py`

```python
async def anthropic_complete(
    prompt: str,
    system_prompt: str | None = None,
    model: str = "claude-3-5-sonnet-20240620",
    **kwargs
) -> str:
    """
    Anthropic Claude completion.

    Models:
    - claude-3-5-sonnet-20240620 (200K context)
    - claude-3-opus-20240229 (200K context)
    - claude-3-haiku-20240307 (200K context)
    """
    import anthropic

    client = anthropic.AsyncAnthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

    response = await client.messages.create(
        model=model,
        system=system_prompt or "",
        messages=[{"role": "user", "content": prompt}],
        **kwargs
    )

    return response.content[0].text
```

**Model Characteristics**:

| Model | Context | Cost (per 1M tokens) | Quality | Speed |
|-------|---------|---------------------|---------|-------|
| **Claude 3.5 Sonnet** | 200K | $3 (in) + $15 (out) | Best | Fast |
| **Claude 3 Opus** | 200K | $15 (in) + $75 (out) | Excellent | Medium |
| **Claude 3 Haiku** | 200K | $0.25 (in) + $1.25 (out) | Good | Very Fast |

**Dependencies**: `anthropic>=0.18.0`

### 3. HuggingFace Transformers (Local Models)

**Файл**: `lightrag/llm/hf.py:44`

```python
async def hf_model_complete(
    prompt: str,
    system_prompt: str | None = None,
    model_name: str = "meta-llama/Llama-3-8B-Instruct",
    **kwargs,
) -> str:
    """
    Local HuggingFace model inference.

    Popular models:
    - meta-llama/Llama-3-8B-Instruct (8B params)
    - mistralai/Mistral-7B-Instruct-v0.3 (7B params)
    - Qwen/Qwen2.5-7B-Instruct (7B params)
    """
    from transformers import AutoModelForCausalLM, AutoTokenizer
    import torch

    # Load model and tokenizer
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        torch_dtype=torch.float16,
        device_map="auto"  # Auto GPU/CPU placement
    )

    # Format prompt
    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": prompt})

    # Tokenize
    inputs = tokenizer.apply_chat_template(
        messages,
        return_tensors="pt"
    ).to(model.device)

    # Generate
    with torch.no_grad():
        outputs = model.generate(
            inputs,
            max_new_tokens=kwargs.get("max_tokens", 512),
            temperature=kwargs.get("temperature", 0.0),
            do_sample=kwargs.get("temperature", 0.0) > 0
        )

    # Decode
    response = tokenizer.decode(outputs[0], skip_special_tokens=True)

    return response
```

**Model Characteristics**:

| Model | Parameters | Memory (FP16) | Quality | Speed (GPU) |
|-------|-----------|---------------|---------|-------------|
| **Llama 3 8B** | 8B | 16 GB | Good | 20 tokens/s |
| **Mistral 7B** | 7B | 14 GB | Good | 25 tokens/s |
| **Qwen 2.5 7B** | 7B | 14 GB | Good | 22 tokens/s |

**Dependencies**: `transformers>=4.30.0`, `torch>=2.0.0`, `accelerate>=0.20.0`

---

## LLM Applications in LightRAG

### 1. Entity Extraction (NER + Relation Extraction)

**Файл**: `lightrag/prompt.py:19`

```python
GRAPH_FIELD_SEP = "<|>"
PROMPTS = {}

PROMPTS["entity_extraction"] = """-Goal-
Given a text document, identify all entities and their relationships.

-Steps-
1. Identify all entities (people, organizations, locations, etc.)
2. For each entity, extract:
   - Entity name
   - Entity type
   - Entity description
3. Identify relationships between entities

-Output Format-
("entity"{GRAPH_FIELD_SEP}<entity_name>{GRAPH_FIELD_SEP}<entity_type>{GRAPH_FIELD_SEP}<entity_description>)
("relationship"{GRAPH_FIELD_SEP}<source_entity>{GRAPH_FIELD_SEP}<target_entity>{GRAPH_FIELD_SEP}<relationship_description>)

-Examples-
Text: "Apple Inc, founded by Steve Jobs, produces iPhones."

Output:
("entity"{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}organization{GRAPH_FIELD_SEP}A technology company)
("entity"{GRAPH_FIELD_SEP}Steve Jobs{GRAPH_FIELD_SEP}person{GRAPH_FIELD_SEP}Founder of Apple Inc)
("entity"{GRAPH_FIELD_SEP}iPhone{GRAPH_FIELD_SEP}product{GRAPH_FIELD_SEP}Smartphone by Apple)
("relationship"{GRAPH_FIELD_SEP}Steve Jobs{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}founded)
("relationship"{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}iPhone{GRAPH_FIELD_SEP}produces)

-Real Data-
Text: {input_text}
######################
Output:
"""
```

**Process**:

```python
# Entity extraction pipeline
async def extract_entities_and_relations(
    text: str,
    llm_func: callable,
    entity_types: list[str] = None
) -> tuple[list[dict], list[dict]]:
    """
    Extract entities and relations using LLM.

    Returns:
        entities: [{name, type, description}, ...]
        relations: [{src, tgt, description}, ...]
    """
    # Format prompt
    prompt = PROMPTS["entity_extraction"].format(input_text=text)

    # LLM call
    response = await llm_func(
        prompt=prompt,
        model="gpt-4",
        temperature=0.0  # Deterministic extraction
    )

    # Parse structured output
    entities = []
    relations = []

    for line in response.split("\n"):
        if line.startswith('("entity"'):
            # Parse: ("entity"<|>Apple Inc<|>organization<|>...)
            parts = line.split(GRAPH_FIELD_SEP)
            entities.append({
                "name": parts[1],
                "type": parts[2],
                "description": parts[3].rstrip(")")
            })

        elif line.startswith('("relationship"'):
            parts = line.split(GRAPH_FIELD_SEP)
            relations.append({
                "src": parts[1],
                "tgt": parts[2],
                "description": parts[3].rstrip(")")
            })

    return entities, relations
```

**Complexity**: O(T) where T = text length (LLM inference time)

### 2. Text Summarization (Gleaning)

**Файл**: `lightrag/prompt.py:89`

```python
PROMPTS["summarize_entity_descriptions"] = """You are a helpful assistant that merges entity descriptions.
Given multiple descriptions of the same entity from different sources, create a comprehensive summary.

Descriptions:
{descriptions}

Output a single merged description:
"""

async def glean_entity_description(
    entity_name: str,
    descriptions: list[str],
    llm_func: callable,
    rounds: int = 2
) -> str:
    """
    Iterative gleaning: refine entity description over multiple rounds.

    Args:
        entity_name: Entity to summarize
        descriptions: List of descriptions from different chunks
        llm_func: Language model function
        rounds: Number of refinement rounds

    Returns:
        Final gleaned description
    """
    current_desc = "\n".join(descriptions)

    for round_idx in range(rounds):
        prompt = PROMPTS["summarize_entity_descriptions"].format(
            descriptions=current_desc
        )

        # LLM summarization
        current_desc = await llm_func(
            prompt=prompt,
            model="gpt-3.5-turbo",  # Cheaper for summarization
            max_tokens=256
        )

    return current_desc
```

**Use Case**: Merging entity descriptions from multiple text chunks.

### 3. Answer Generation (RAG)

**Файл**: `lightrag/prompt.py:156`

```python
PROMPTS["rag_response"] = """---Role---
You are a helpful assistant responding to questions about provided documents.

---Goal---
Generate a response to the user's question using only the information in the context below.

---Context---
{context}

---Question---
{query}

---Instructions---
- Only use information from the context
- Cite specific entities/facts when possible
- If context insufficient, say "I don't have enough information"

---Response---
"""

async def generate_rag_response(
    query: str,
    context_chunks: list[str],
    entities: list[dict],
    llm_func: callable
) -> str:
    """
    Generate answer using retrieved context (RAG).

    Args:
        query: User question
        context_chunks: Top-K retrieved chunks
        entities: Relevant entities from graph
        llm_func: Language model

    Returns:
        Generated answer
    """
    # Format context
    context_text = "\n\n".join([
        f"Chunk {i+1}: {chunk}"
        for i, chunk in enumerate(context_chunks)
    ])

    # Add entity information
    entity_text = "\n".join([
        f"- {e['name']} ({e['type']}): {e['description']}"
        for e in entities
    ])

    full_context = f"Documents:\n{context_text}\n\nEntities:\n{entity_text}"

    # Generate answer
    prompt = PROMPTS["rag_response"].format(
        context=full_context,
        query=query
    )

    response = await llm_func(
        prompt=prompt,
        model="gpt-4",
        temperature=0.3  # Slight creativity
    )

    return response
```

### 4. Chain of Thought (CoT) Reasoning

**Файл**: `lightrag/llm/openai.py:128`

```python
async def openai_complete_with_cot(
    query: str,
    context: str,
    llm_func: callable
) -> str:
    """
    Use Chain of Thought prompting for complex reasoning.

    CoT = ask LLM to show reasoning steps before answer.
    """
    cot_prompt = f"""Question: {query}

Context: {context}

Let's solve this step by step:
1. First, identify the key information from the context
2. Then, reason about the relationships
3. Finally, formulate the answer

Reasoning:
"""

    response = await llm_func(
        prompt=cot_prompt,
        model="gpt-4",
        temperature=0.0
    )

    return response
```

---

## Training Paradigm: Supervised Fine-Tuning (SFT)

### Pre-Training

```
Objective: Next token prediction (unsupervised)
Data: Large web corpus (trillions of tokens)

Loss = CrossEntropy(predicted_token, actual_token)

Training: Autoregressive language modeling
"Apple Inc produces" → predict "iPhones"
```

### Instruction Fine-Tuning

```
Objective: Follow instructions (supervised)
Data: (instruction, response) pairs (millions)

Example:
Instruction: "Extract entities from: Apple Inc produces iPhones"
Response: "entity<|#|>Apple Inc<|#|>organization<|#|>..."

Loss = CrossEntropy on response tokens only
```

### RLHF (Reinforcement Learning from Human Feedback)

```
Objective: Align with human preferences
Data: Human preference rankings

Process:
1. Collect human ratings (response A > response B)
2. Train reward model (predict human preference)
3. Optimize LLM policy using PPO (Proximal Policy Optimization)

Result: More helpful, harmless, honest responses
```

---

## Связь с Vector Algorithms

LLMs работают совместно с vector embeddings:

**Файл**: spec/vectors/01-embedding-generation.md

```python
# LLM for entity extraction → Embedding for similarity search
entities, relations = await extract_entities_and_relations(text, llm_func)

# Generate embeddings for entities
entity_embeddings = await embedding_func([e["name"] for e in entities])

# Store in vector DB
await entities_vdb.upsert({
    f"ent-{e['name']}": {
        "content": e["description"],
        "entity_type": e["type"]
    }
    for e in entities
})
```

**Interaction**: LLM extracts **structured data**, embeddings enable **semantic search**.

---

## Связь с Graph Algorithms

LLM-generated entities/relations → Graph construction:

**Файл**: spec/graph/04-entity-merging.md

```python
# LLM extracts entities from multiple chunks
chunk1_entities = await extract_entities("Apple Inc produces iPhones", llm_func)
chunk2_entities = await extract_entities("Apple company makes smartphones", llm_func)

# Graph algorithm merges duplicates
await graph.amerge_entities(
    source_entities=["Apple Inc", "Apple company"],
    target_entity="Apple Inc",
    merge_strategy={"description": "concatenate"}
)
```

**Interaction**: LLM provides **raw extractions**, graph algorithms provide **deduplication**.

---

## Performance Characteristics

### Latency (100 tokens output)

| Model | API Latency | Local (GPU) | Local (CPU) |
|-------|-------------|-------------|-------------|
| **GPT-4** | 3-5s | N/A | N/A |
| **GPT-3.5-turbo** | 1-2s | N/A | N/A |
| **Claude 3.5 Sonnet** | 2-4s | N/A | N/A |
| **Llama 3 8B** | N/A | 2-3s | 15-25s |
| **Mistral 7B** | N/A | 1.5-2.5s | 12-20s |

### Throughput (tokens/second)

```python
# API models (parallel requests)
GPT-4: ~20 tokens/s per request, 100+ concurrent
GPT-3.5-turbo: ~40 tokens/s per request, 200+ concurrent

# Local models (single GPU)
Llama 3 8B (A100): ~50 tokens/s
Mistral 7B (A100): ~60 tokens/s

# Batch processing (local)
Batch size 8: ~200 tokens/s total (8 × 25)
```

### Cost Comparison (Entity Extraction Task)

```
Input: 1000 text chunks (500 tokens each)
Output: 200 tokens per chunk (entities/relations)

Total tokens:
Input: 1000 × 500 = 500K tokens
Output: 1000 × 200 = 200K tokens

Cost:
GPT-4: 500K × $30/1M + 200K × $60/1M = $15 + $12 = $27
GPT-3.5-turbo: 500K × $0.50/1M + 200K × $1.50/1M = $0.25 + $0.30 = $0.55
Claude 3.5 Sonnet: 500K × $3/1M + 200K × $15/1M = $1.50 + $3.00 = $4.50

Llama 3 8B (local): $0 (infrastructure: ~$1/hour GPU)
```

**Recommendation**:
- **Best quality**: GPT-4 or Claude 3.5 Sonnet
- **Best cost-performance**: GPT-3.5-turbo for most tasks
- **Privacy/control**: Local Llama 3 or Mistral

---

## Optimization Strategies

### 1. Caching

```python
from functools import lru_cache
import hashlib

# Cache LLM responses
@lru_cache(maxsize=10000)
async def cached_llm_call(prompt_hash: str, **kwargs):
    # Actual LLM call only if not cached
    return await llm_func(prompt, **kwargs)

# Usage
prompt_hash = hashlib.md5(prompt.encode()).hexdigest()
response = await cached_llm_call(prompt_hash, model="gpt-4")
```

**Speedup**: 100x for repeated queries (instant cache retrieval vs 3s API call)

### 2. Batching

```python
# Batch entity extraction (parallel)
chunks = [chunk1, chunk2, ..., chunk100]

# Sequential (slow): 100 × 3s = 300s
for chunk in chunks:
    await extract_entities(chunk, llm_func)

# Parallel (fast): max(3s) = 3s
import asyncio
tasks = [extract_entities(chunk, llm_func) for chunk in chunks]
results = await asyncio.gather(*tasks)
```

**Speedup**: 100x for batch processing (parallelism)

### 3. Prompt Compression

```python
# Verbose prompt (1000 tokens)
prompt = f"""
Extract all entities and relationships from the following text.
For each entity, provide the name, type, and description.
For each relationship, provide the source entity, target entity, and description.
...
Text: {long_text}
"""

# Compressed prompt (300 tokens)
prompt = f"Extract entities/relations:\n{long_text}"

# Cost savings: 70% reduction in input tokens
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
