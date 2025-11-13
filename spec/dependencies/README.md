# External Dependencies: Сторонние Зависимости LightRAG

## Обзор

Эта директория содержит подробную документацию о всех сторонних библиотеках, пакетах и зависимостях, используемых в LightRAG. Документация структурирована по **аспектам обработки информации**.

## Классификация по Аспектам

### 1. LLM Integration (Интеграция Языковых Моделей)
**Файл**: [01-llm-integration.md](01-llm-integration.md)

Библиотеки для взаимодействия с большими языковыми моделями:
- **openai** - OpenAI API (GPT-4, GPT-3.5, embeddings)
- **anthropic** - Anthropic Claude API
- **tenacity** - Retry logic для LLM вызовов
- **pipmaster** - Dynamic library installation

**Роль**: Ядро семантической обработки - entity extraction, summarization, answer generation.

---

### 2. Vector & Embedding Processing (Векторная Обработка)
**Файл**: [02-vector-embedding.md](02-vector-embedding.md)

Библиотеки для работы с embeddings и векторными представлениями:
- **tiktoken** - Tokenization для OpenAI models
- **numpy** - Numerical operations на vectors
- **nano-vectordb** - Lightweight vector database

**Роль**: Проекция в semantic space, similarity search, entity resolution.

---

### 3. Graph Processing (Обработка Графов)
**Файл**: [03-graph-processing.md](03-graph-processing.md)

Библиотеки для работы с графами знаний:
- **networkx** - Graph algorithms и data structures
- **python-louvain** (optional) - Community detection

**Роль**: Knowledge graph construction, traversal, community detection.

---

### 4. Storage Backends (Системы Хранения)
**Файл**: [04-storage-backends.md](04-storage-backends.md)

Интеграции с различными системами хранения:

**Vector Databases**:
- **milvus** - Distributed vector database
- **qdrant-client** - Qdrant vector search
- **faiss** - Facebook AI Similarity Search

**Graph Databases**:
- **neo4j** - Graph database
- **memgraph** - In-memory graph database

**General Databases**:
- **redis** - In-memory KV store
- **motor** (MongoDB) - Document database
- **asyncpg** (PostgreSQL) - Relational database

**Роль**: Persistent storage для entities, relations, chunks, embeddings.

---

### 5. Data Processing (Обработка Данных)
**Файл**: [05-data-processing.md](05-data-processing.md)

Библиотеки для обработки и валидации данных:
- **pandas** - DataFrame operations
- **pydantic** - Data validation
- **json_repair** - JSON parsing/repair
- **xlsxwriter** - Excel export

**Роль**: Data transformation, validation, export.

---

### 6. Async & Networking (Асинхронность и Сеть)
**Файл**: [06-async-networking.md](06-async-networking.md)

Библиотеки для асинхронных операций и сетевых запросов:
- **asyncio** (stdlib) - Async framework
- **aiohttp** - Async HTTP client/server
- **httpx** - Modern HTTP client
- **httpcore** - HTTP protocol implementation

**Роль**: Non-blocking I/O для LLM calls, DB operations, parallel processing.

---

### 7. API & Server (API и Сервер)
**Файл**: [07-api-server.md](07-api-server.md)

Библиотеки для REST API и веб-сервера:
- **fastapi** - Modern web framework
- **uvicorn** - ASGI server
- **pyjwt** / **python-jose** - JWT authentication
- **passlib** - Password hashing
- **python-multipart** - File uploads

**Роль**: HTTP API endpoints, authentication, file handling.

---

### 8. Utilities (Утилиты)
**Файл**: [08-utilities.md](08-utilities.md)

Вспомогательные библиотеки:
- **python-dotenv** - Environment variables
- **configparser** - Configuration files
- **pypinyin** - Chinese pinyin conversion
- **psutil** - System monitoring
- **pytz** - Timezone handling

**Роль**: Configuration management, internationalization, system utilities.

---

## Dependency Graph (Граф Зависимостей)

```
┌──────────────────────────────────────────────────────────┐
│                    LightRAG Core                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────┐      ┌─────────────────┐           │
│  │  LLM Layer     │      │  Storage Layer  │           │
│  │                │      │                 │           │
│  │  openai        │──┐   │  networkx       │           │
│  │  anthropic     │  │   │  milvus         │           │
│  │  tenacity      │  │   │  neo4j          │           │
│  └────────────────┘  │   │  redis          │           │
│         ↓            │   │  postgresql     │           │
│  ┌────────────────┐  │   └─────────────────┘           │
│  │ Embedding      │  │           ↑                      │
│  │                │  │           │                      │
│  │  tiktoken      │  │   ┌─────────────────┐           │
│  │  numpy         │  │   │  Data Layer     │           │
│  │  nano-vectordb │←─┘   │                 │           │
│  └────────────────┘      │  pandas         │           │
│         ↓                │  pydantic       │           │
│  ┌────────────────┐      │  json_repair    │           │
│  │  Graph Layer   │      └─────────────────┘           │
│  │                │              ↑                      │
│  │  networkx      │──────────────┘                      │
│  └────────────────┘                                     │
│         ↓                                               │
│  ┌────────────────────────────────────────────┐        │
│  │          Async & API Layer                 │        │
│  │                                            │        │
│  │  asyncio, aiohttp, fastapi, uvicorn       │        │
│  └────────────────────────────────────────────┘        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Core vs Optional Dependencies

### Core Dependencies (Обязательные)

Необходимы для базовой функциональности:

```python
core_deps = [
    "networkx",          # Graph processing
    "numpy",             # Vector operations
    "tiktoken",          # Tokenization
    "pydantic",          # Data validation
    "python-dotenv",     # Config management
    "json_repair",       # JSON parsing
    "tenacity",          # Retry logic
    "nano-vectordb",     # Default vector DB
    "pandas",            # Data processing
]
```

### Optional Dependencies (Опциональные)

Нужны для специфических функций:

```python
# LLM providers (choose one or more)
llm_providers = ["openai", "anthropic", "ollama", ...]

# Storage backends (choose one or more)
storage_backends = [
    "milvus",      # For Milvus vector DB
    "qdrant",      # For Qdrant vector DB
    "neo4j",       # For Neo4j graph DB
    "redis",       # For Redis KV store
    "motor",       # For MongoDB
    "asyncpg",     # For PostgreSQL
]

# API server (optional)
api_deps = ["fastapi", "uvicorn", "pyjwt", "passlib"]
```

## Installation Patterns

### Minimal Installation

```bash
pip install lightrag-hku
```

Includes only core dependencies.

### With API Server

```bash
pip install lightrag-hku[api]
```

Includes API server dependencies (FastAPI, uvicorn, authentication).

### With Specific Storage Backend

```bash
# For Neo4j
pip install lightrag-hku neo4j

# For Milvus
pip install lightrag-hku pymilvus

# For PostgreSQL
pip install lightrag-hku asyncpg
```

### Full Installation (All Optional)

```bash
pip install lightrag-hku[api]
pip install openai anthropic
pip install pymilvus qdrant-client faiss-cpu
pip install neo4j redis motor asyncpg
```

## Version Requirements

### Python Version

```
Python >= 3.10
```

LightRAG требует Python 3.10+ для:
- Modern type hints (PEP 604: `X | Y` syntax)
- `match` statement (structural pattern matching)
- Improved asyncio performance

### Key Version Constraints

```toml
pandas >= 2.0.0         # Modern DataFrame API
xlsxwriter >= 3.1.0     # Latest Excel features
setuptools >= 64        # Modern build system
```

## Dependency Installation Strategy

### Lazy Import Pattern

LightRAG использует **lazy imports** для optional dependencies:

```python
# In lightrag/llm/openai.py
import pipmaster as pm

if not pm.is_installed("openai"):
    pm.install("openai")  # Dynamic install if not present

from openai import AsyncOpenAI
```

**Преимущества**:
- Не требует pre-installation всех dependencies
- Автоматически устанавливает при первом использовании
- Минимальный размер базовой установки

### Import Guards

```python
# In lightrag/utils.py
try:
    import pypinyin
    _PYPINYIN_AVAILABLE = True
except ImportError:
    pypinyin = None
    _PYPINYIN_AVAILABLE = False
    logger.warning("pypinyin not installed, using fallback")
```

**Преимущества**:
- Graceful degradation
- Optional features не блокируют core functionality

## Security Considerations

### Authentication Libraries

```python
passlib[bcrypt]         # Password hashing
PyJWT                   # JWT tokens
python-jose[cryptography]  # JOSE/JWS/JWT
```

**Использование**:
- API authentication (lightrag/api/auth.py)
- Token-based access control
- Secure password storage

### Network Security

```python
httpx                   # Modern HTTP with security features
aiohttp                 # Async HTTP client/server
```

**Особенности**:
- TLS/SSL support
- Certificate verification
- Timeout handling

## Performance Characteristics

### High-Performance Libraries

| Library | Performance Benefit | Use Case |
|---------|-------------------|----------|
| **numpy** | C-optimized array operations | Vector similarity |
| **tiktoken** | Rust-based tokenizer | Fast tokenization |
| **asyncio** | Non-blocking I/O | Concurrent LLM calls |
| **faiss** | GPU-accelerated search | Large-scale vector search |
| **redis** | In-memory storage | Fast KV lookup |

### Scalability

```python
# Async-first design
asyncio.gather()        # Parallel operations
aiohttp                 # Concurrent HTTP requests
asyncpg                 # Async PostgreSQL

# Distributed storage
milvus                  # Distributed vector DB
redis (cluster mode)    # Distributed KV store
neo4j (cluster)         # Distributed graph DB
```

## Troubleshooting Common Issues

### Issue 1: tiktoken Installation

**Problem**: `tiktoken` fails to install (requires Rust compiler)

**Solution**:
```bash
# Use pre-built wheels
pip install tiktoken --prefer-binary

# Or install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Issue 2: faiss GPU Support

**Problem**: Need GPU acceleration for FAISS

**Solution**:
```bash
# CPU version (default)
pip install faiss-cpu

# GPU version (requires CUDA)
pip install faiss-gpu
```

### Issue 3: Neo4j Driver Compatibility

**Problem**: Neo4j driver version mismatch

**Solution**:
```bash
# Match Neo4j server version
pip install neo4j==5.14.0  # For Neo4j 5.x server
```

## Related Documentation

- **[Architecture Patterns](../architecture/README.md)** - How dependencies are used in design patterns
- **[Storage Abstraction](../architecture/01-storage-abstraction.md)** - Storage backend implementations
- **[LLM Agents](../06-llm-agents.md)** - LLM integration details
- **[API Documentation](../../lightrag/api/README.md)** - API server setup

---

## Содержание Файлов

1. **[LLM Integration](01-llm-integration.md)** - OpenAI, Anthropic, tenacity, pipmaster
2. **[Vector & Embedding](02-vector-embedding.md)** - tiktoken, numpy, nano-vectordb
3. **[Graph Processing](03-graph-processing.md)** - networkx, community detection
4. **[Storage Backends](04-storage-backends.md)** - Milvus, Neo4j, Redis, PostgreSQL, MongoDB
5. **[Data Processing](05-data-processing.md)** - pandas, pydantic, json_repair, xlsxwriter
6. **[Async & Networking](06-async-networking.md)** - asyncio, aiohttp, httpx, tenacity
7. **[API & Server](07-api-server.md)** - FastAPI, uvicorn, JWT, authentication
8. **[Utilities](08-utilities.md)** - dotenv, configparser, pypinyin, psutil

---

**Версия**: 1.0
**Дата**: 2025-01-13
**Источник**: pyproject.toml, setup.py, source code analysis
