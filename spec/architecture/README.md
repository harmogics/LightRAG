# LightRAG Architecture: Patterns & Design

## Обзор

Этот каталог содержит документацию архитектурных и семантических паттернов, используемых в LightRAG на уровне исходного кода. Система построена на комбинации классических design patterns и специализированных агентских паттернов для работы с LLM.

---

## Категории Паттернов

### 🏗️ Structural Patterns (Структурные)

| Паттерн | Файл | Назначение |
|---------|------|------------|
| **Abstract Storage Pattern** | [01-storage-abstraction.md](01-storage-abstraction.md) | Унифицированный интерфейс для KV/Vector/Graph storage |
| **Layered Architecture** | [02-layered-architecture.md](02-layered-architecture.md) | Разделение на слои: Storage, Operations, API |
| **Namespace Pattern** | [03-namespace-pattern.md](03-namespace-pattern.md) | Workspace isolation и multi-tenancy |

### 🎯 Behavioral Patterns (Поведенческие)

| Паттерн | Файл | Назначение |
|---------|------|------------|
| **Strategy Pattern** | [04-strategy-pattern.md](04-strategy-pattern.md) | LLM providers, chunking strategies, embeddings |
| **Pipeline Pattern** | [05-pipeline-pattern.md](05-pipeline-pattern.md) | Document processing и query execution pipelines |
| **Agent Pattern** | [06-agent-pattern.md](06-agent-pattern.md) | LLM agents для extraction, gleaning, summarization |

### ⚡ Concurrency Patterns (Конкурентные)

| Паттерн | Файл | Назначение |
|---------|------|------------|
| **Async/Await Pattern** | [07-async-pattern.md](07-async-pattern.md) | Асинхронная обработка документов и запросов |
| **Semaphore Pattern** | [08-semaphore-pattern.md](08-semaphore-pattern.md) | Управление параллельными LLM calls |
| **Lock Pattern** | [09-lock-pattern.md](09-lock-pattern.md) | Keyed locks для consistency в distributed scenarios |

### 🧠 Semantic Patterns (Семантические)

| Паттерн | Файл | Назначение |
|---------|------|------------|
| **Map-Reduce Pattern** | [10-map-reduce-pattern.md](10-map-reduce-pattern.md) | Параллельная обработка chunks и summarization |
| **Multi-Turn Dialog Pattern** | [11-multi-turn-dialog.md](11-multi-turn-dialog.md) | Gleaning и conversation history |
| **Retrieval Augmentation** | [12-rag-pattern.md](12-rag-pattern.md) | Hybrid retrieval (vector + graph) |

### 💾 Data Patterns (Данные)

| Паттерн | Файл | Назначение |
|---------|------|------------|
| **Cache Pattern** | [13-cache-pattern.md](13-cache-pattern.md) | LLM response caching |
| **JSONL Streaming** | [14-jsonl-pattern.md](14-jsonl-pattern.md) | Incremental data processing |
| **Checkpoint Pattern** | [15-checkpoint-pattern.md](15-checkpoint-pattern.md) | Document processing resumption |

---

## Архитектурный Обзор

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        LightRAG System                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     API Layer                             │  │
│  │  • REST API (FastAPI)                                     │  │
│  │  • Python SDK                                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Operations Layer                         │  │
│  │  • Document Processing (operate.py)                       │  │
│  │  • Entity Extraction                                      │  │
│  │  • Graph Construction                                     │  │
│  │  • Query Processing                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Agent Layer                             │  │
│  │  • Extraction Agent (LLM)                                 │  │
│  │  • Gleaning Agent (LLM)                                   │  │
│  │  • Summarization Agent (LLM)                              │  │
│  │  • Query Agent (LLM)                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Storage Layer                            │  │
│  │  ┌──────────────┐ ┌────────────┐ ┌──────────────────┐   │  │
│  │  │ KV Storage   │ │   Vector   │ │  Graph Storage   │   │  │
│  │  │ (Docs/Cache) │ │  Storage   │ │   (NetworkX/     │   │  │
│  │  │              │ │  (Embeddings)│ │    Neo4j)        │   │  │
│  │  └──────────────┘ └────────────┘ └──────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              External Services Layer                      │  │
│  │  • LLM Providers (OpenAI, Anthropic, ...)                │  │
│  │  • Embedding Models                                       │  │
│  │  • Rerankers                                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Design Principles

### 1. Separation of Concerns

```python
# Clear separation of responsibilities
responsibilities = {
    "lightrag.py": "Orchestration, configuration",
    "operate.py": "Core algorithms (extraction, query)",
    "base.py": "Abstract interfaces",
    "storage/*.py": "Storage implementations",
    "llm/*.py": "LLM provider adapters"
}
```

**Benefits**:
- ✅ Testability (isolated components)
- ✅ Maintainability (changes локализованы)
- ✅ Extensibility (add new implementations)

### 2. Abstract Interfaces (ABC)

```python
# Base storage interface
class BaseVectorStorage(ABC):
    @abstractmethod
    async def query(self, query: str, top_k: int) -> list[dict]:
        """Must be implemented by all vector storage"""

    @abstractmethod
    async def upsert(self, data: dict) -> None:
        """Must be implemented by all vector storage"""
```

**Benefits**:
- ✅ Pluggable implementations (NanoVDB, Milvus, Qdrant)
- ✅ Consistent API across providers
- ✅ Type safety via ABC enforcement

### 3. Async-First Design

```python
# All I/O operations are async
async def extract_entities(chunks, global_config):
    tasks = [process_chunk(c) for c in chunks]
    results = await asyncio.gather(*tasks)
    return merge_results(results)
```

**Benefits**:
- ✅ Non-blocking I/O (LLM calls, DB operations)
- ✅ High concurrency (100+ parallel tasks)
- ✅ Efficient resource utilization

### 4. Configuration-Driven

```python
# Single source of truth for configuration
rag = LightRAG(
    working_dir="./workspace",
    llm_model_func=llm_func,
    embedding_func=embedding_func,
    chunk_token_size=1024,
    entity_extract_max_gleaning=1,
    # ... many more parameters
)
```

**Benefits**:
- ✅ Flexible configuration
- ✅ Environment-based overrides
- ✅ Easy A/B testing

---

## Key Architectural Patterns

### Pattern 1: Three-Layer Storage Abstraction

```
Application Layer
       ↓
Abstract Interface (BaseXStorage)
       ↓
Concrete Implementation (JsonKVStorage, MilvusVectorStorage, Neo4jGraphStorage)
```

**Purpose**:
- Support multiple storage backends
- Easy migration between providers
- Consistent API

**Files**: `base.py`, `storage/*.py`, `kg/*.py`

### Pattern 2: Strategy Pattern для LLM Providers

```python
# Different strategies for LLM calls
llm_providers = {
    "openai": openai_complete_if_cache,
    "anthropic": anthropic_complete_if_cache,
    "azure": azure_openai_complete,
    "ollama": ollama_model_complete,
    # ... etc
}

# Select strategy at runtime
llm_func = llm_providers[provider_name]
```

**Purpose**:
- Swap LLM providers без изменения кода
- A/B testing моделей
- Fallback mechanisms

**Files**: `llm/*.py`

### Pattern 3: Pipeline Pattern

```python
# Document processing pipeline
Document → Chunk → Extract → Glean → Parse → Merge → Vectorize → Store

# Query pipeline
Query → Extract Keywords → Resolve Entities → Assemble Context → Generate → Format
```

**Purpose**:
- Clear data flow
- Composable operations
- Easy debugging (inspect intermediate results)

**Files**: `operate.py`, `lightrag.py`

### Pattern 4: Agent Pattern для LLM

```python
# Specialized agents for different tasks
agents = {
    "extraction": EntityExtractionAgent(prompt=P1+P2),
    "gleaning": GleaningAgent(prompt=P1+P3),
    "summarization": SummarizationAgent(prompt=P4),
    "query": QueryAgent(prompt=P6/P7)
}
```

**Purpose**:
- Encapsulate LLM behavior
- Prompt management
- Specialized semantic operations

**Files**: `operate.py`, `prompt.py`

### Pattern 5: Semaphore для Rate Limiting

```python
# Control concurrent LLM calls
semaphore = asyncio.Semaphore(max_async_calls)

async def process_with_limit(chunk):
    async with semaphore:
        return await llm_call(chunk)
```

**Purpose**:
- Prevent API rate limit errors
- Control resource usage
- Graceful degradation

**Files**: `operate.py`

### Pattern 6: Map-Reduce для Parallel Processing

```python
# Map phase: parallel processing
tasks = [process_chunk(c) for c in chunks]
results = await asyncio.gather(*tasks)

# Reduce phase: merge results
final = merge_entity_descriptions(results)
```

**Purpose**:
- Scalability (100+ chunks)
- Efficient parallelization
- Aggregation of distributed results

**Files**: `operate.py`

---

## Code Organization

### Module Structure

```
lightrag/
├── __init__.py          # Public API
├── lightrag.py          # Main LightRAG class (orchestration)
├── base.py              # Abstract base classes
├── operate.py           # Core algorithms (extraction, query)
├── prompt.py            # Prompt templates
├── utils.py             # Utilities (embedding, cache, etc.)
├── constants.py         # Configuration constants
├── types.py             # Type definitions
│
├── storage/             # KV & Vector storage implementations
│   ├── json_kv.py
│   ├── mongo_kv.py
│   ├── postgres_kv.py
│   ├── nano_vdb.py
│   ├── milvus_vdb.py
│   └── qdrant_vdb.py
│
├── kg/                  # Graph storage implementations
│   ├── networkx_impl.py
│   ├── neo4j_impl.py
│   └── memgraph_impl.py
│
└── llm/                 # LLM provider adapters
    ├── openai.py
    ├── anthropic.py
    ├── azure_openai.py
    ├── ollama.py
    └── ...
```

### Dependency Graph

```
lightrag.py
    ↓
operate.py → prompt.py
    ↓
base.py (ABC interfaces)
    ↓
storage/*.py + kg/*.py + llm/*.py (Implementations)
```

---

## Pattern Application Examples

### Example 1: Adding New Storage Backend

```python
# 1. Implement abstract interface
from lightrag.base import BaseVectorStorage

class MyCustomVectorStorage(BaseVectorStorage):
    async def query(self, query: str, top_k: int) -> list[dict]:
        # Custom implementation
        ...

    async def upsert(self, data: dict) -> None:
        # Custom implementation
        ...

# 2. Use it
rag = LightRAG(
    vector_db_storage_cls_kwargs={
        "cls": MyCustomVectorStorage,
        "kwargs": {"my_param": "value"}
    }
)
```

**Pattern Used**: Abstract Storage Pattern

### Example 2: Adding New LLM Provider

```python
# 1. Implement LLM function with standard signature
async def my_custom_llm_func(
    prompt: str,
    system_prompt: str = None,
    **kwargs
) -> str:
    # Call your LLM API
    response = await my_llm_api.complete(prompt, system_prompt)
    return response

# 2. Use it
rag = LightRAG(
    llm_model_func=my_custom_llm_func
)
```

**Pattern Used**: Strategy Pattern

### Example 3: Extending Pipeline

```python
# Custom transformation in pipeline
async def custom_transform(entities: list[dict]) -> list[dict]:
    """Custom entity enrichment"""
    enriched = []
    for entity in entities:
        # Add custom fields
        entity["custom_score"] = compute_score(entity)
        enriched.append(entity)
    return enriched

# Inject into pipeline (after extraction, before merge)
# Would require modification to operate.py
```

**Pattern Used**: Pipeline Pattern

---

## Metrics & Observability

### Performance Patterns

```python
# Instrumentation points
instrumentation = {
    "llm_calls": "Track latency, cost, errors",
    "storage_ops": "Track read/write latency",
    "pipeline_stages": "Track stage completion time",
    "cache_hits": "Track cache hit rate"
}
```

### Logging Patterns

```python
import logging

logger = logging.getLogger("lightrag")

# Structured logging at key points
logger.info(f"Processing document {doc_id}: {len(chunks)} chunks")
logger.debug(f"Extracted {len(entities)} entities from chunk {chunk_id}")
logger.warning(f"Gleaning found {new_entities} missed entities")
```

---

## Testing Patterns

### Unit Testing

```python
# Test abstract implementations
@pytest.mark.asyncio
async def test_vector_storage():
    storage = MyVectorStorage()

    # Test query
    results = await storage.query("test", top_k=5)
    assert len(results) <= 5

    # Test upsert
    await storage.upsert({"id1": {"content": "test"}})
```

### Integration Testing

```python
# Test full pipeline
@pytest.mark.asyncio
async def test_document_processing():
    rag = LightRAG(working_dir="./test_workspace")

    # Insert document
    await rag.ainsert("Test document content")

    # Query
    result = await rag.aquery("test query")

    assert result is not None
```

---

## Best Practices

### 1. Use Abstract Interfaces

```python
# ✅ Good: Depend on abstraction
def process(storage: BaseVectorStorage):
    ...

# ❌ Bad: Depend on concrete class
def process(storage: NanoVectorDB):
    ...
```

### 2. Prefer Async

```python
# ✅ Good: Non-blocking
async def process():
    result = await async_operation()

# ❌ Bad: Blocking
def process():
    result = sync_operation()
```

### 3. Use Type Hints

```python
# ✅ Good: Type safe
async def query(text: str, top_k: int) -> list[dict]:
    ...

# ❌ Bad: No types
async def query(text, top_k):
    ...
```

### 4. Handle Errors Gracefully

```python
# ✅ Good: Graceful degradation
try:
    result = await llm_call(prompt)
except LLMError:
    logger.warning("LLM call failed, using fallback")
    result = fallback_response()
```

---

## Дальнейшее Чтение

Для детального изучения каждого паттерна см. соответствующие файлы:

**Structural**:
1. [Storage Abstraction Pattern](01-storage-abstraction.md)
2. [Layered Architecture](02-layered-architecture.md)
3. [Namespace Pattern](03-namespace-pattern.md)

**Behavioral**:
4. [Strategy Pattern](04-strategy-pattern.md)
5. [Pipeline Pattern](05-pipeline-pattern.md)
6. [Agent Pattern](06-agent-pattern.md)

**Concurrency**:
7. [Async/Await Pattern](07-async-pattern.md)
8. [Semaphore Pattern](08-semaphore-pattern.md)
9. [Lock Pattern](09-lock-pattern.md)

**Semantic**:
10. [Map-Reduce Pattern](10-map-reduce-pattern.md)
11. [Multi-Turn Dialog Pattern](11-multi-turn-dialog.md)
12. [RAG Pattern](12-rag-pattern.md)

**Data**:
13. [Cache Pattern](13-cache-pattern.md)
14. [JSONL Streaming](14-jsonl-pattern.md)
15. [Checkpoint Pattern](15-checkpoint-pattern.md)

---

**Контакты для вопросов**: См. [LightRAG GitHub](https://github.com/HKUDS/LightRAG)
