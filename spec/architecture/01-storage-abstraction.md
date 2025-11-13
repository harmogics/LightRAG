# Storage Abstraction Pattern

## Overview

**Pattern Type**: Structural  
**Category**: Abstract Base Class (ABC) Pattern  
**Purpose**: Унифицированный интерфейс для различных storage backends (KV, Vector, Graph)

---

## Problem

LightRAG должна поддерживать множество storage backends:
- **KV Storage**: JSON files, MongoDB, PostgreSQL, Redis
- **Vector Storage**: NanoVDB, Milvus, Qdrant, Chroma
- **Graph Storage**: NetworkX (in-memory), Neo4j, MemGraph

**Challenges**:
1. Каждый backend имеет свой API
2. Нужна возможность менять backends без изменения кода
3. Консистентность операций across different providers

---

## Solution: Abstract Storage Pattern

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│            (lightrag.py, operate.py)                         │
└────────────────────────┬────────────────────────────────────┘
                         │ Uses abstract interfaces
                         ↓
┌─────────────────────────────────────────────────────────────┐
│               Abstract Interface Layer                       │
│                    (base.py)                                 │
│                                                              │
│  ┌──────────────────┐ ┌─────────────────┐ ┌──────────────┐│
│  │ BaseKVStorage    │ │BaseVectorStorage│ │BaseGraph     ││
│  │ (ABC)            │ │    (ABC)        │ │Storage (ABC) ││
│  │                  │ │                 │ │              ││
│  │ + get()          │ │ + query()       │ │ + upsert()   ││
│  │ + upsert()       │ │ + upsert()      │ │ + get_node() ││
│  │ + delete()       │ │ + delete()      │ │ + get_edge() ││
│  └──────────────────┘ └─────────────────┘ └──────────────┘│
└────────────────────────┬────────────────────────────────────┘
                         │ Implemented by
                         ↓
┌─────────────────────────────────────────────────────────────┐
│            Concrete Implementation Layer                     │
│                                                              │
│  KV Implementations    Vector Implementations  Graph Impls  │
│  ├─ JsonKVStorage     ├─ NanoVectorDB         ├─ NetworkX  │
│  ├─ MongoKVStorage    ├─ MilvusVectorDB       ├─ Neo4j     │
│  ├─ PostgresKVStorage ├─ QdrantVectorDB       └─ MemGraph  │
│  └─ RedisKVStorage    └─ ChromaVectorDB                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Abstract Base Class

```python
# From lightrag/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any

@dataclass
class StorageNameSpace(ABC):
    """Base class for all storage types"""

    namespace: str      # Logical namespace (e.g., "entities", "chunks")
    workspace: str      # Physical workspace directory
    global_config: dict[str, Any]

    async def initialize(self):
        """Initialize storage (optional hook)"""
        pass

    async def finalize(self):
        """Finalize storage (optional hook)"""
        pass

    @abstractmethod
    async def index_done_callback(self) -> None:
        """Commit operations after indexing"""
        # Must be implemented by all storages

    @abstractmethod
    async def drop(self) -> dict[str, str]:
        """Drop all data and clean up"""
        # Must be implemented by all storages
```

### Vector Storage ABC

```python
@dataclass
class BaseVectorStorage(StorageNameSpace, ABC):
    """Abstract interface for vector databases"""

    embedding_func: EmbeddingFunc
    cosine_better_than_threshold: float = 0.2
    meta_fields: set[str] = field(default_factory=set)

    @abstractmethod
    async def query(
        self,
        query: str,
        top_k: int,
        query_embedding: list[float] = None
    ) -> list[dict[str, Any]]:
        """
        Query vector storage and retrieve top_k results

        Args:
            query: Text query
            top_k: Number of results
            query_embedding: Optional pre-computed embedding

        Returns:
            List of results with similarity scores
        """

    @abstractmethod
    async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
        """Insert or update vectors"""

    @abstractmethod
    async def delete_entity(self, entity_name: str) -> None:
        """Delete single entity by name"""

    @abstractmethod
    async def delete_entity_relation(self, entity_name: str) -> None:
        """Delete entity and all its relations"""
```

### KV Storage ABC

```python
@dataclass
class BaseKVStorage(StorageNameSpace, ABC):
    """Abstract interface for key-value stores"""

    @abstractmethod
    async def all_keys(self) -> list[str]:
        """Get all keys"""

    @abstractmethod
    async def get_by_id(self, id: str) -> dict | None:
        """Get single item by ID"""

    @abstractmethod
    async def get_by_ids(
        self,
        ids: list[str],
        fields: list[str] | None = None
    ) -> list[dict]:
        """Get multiple items by IDs"""

    @abstractmethod
    async def filter_keys(self, data: list[str]) -> set[str]:
        """Filter keys that exist in storage"""

    @abstractmethod
    async def upsert(self, data: dict[str, dict]) -> None:
        """Insert or update items"""

    @abstractmethod
    async def drop(self) -> dict[str, str]:
        """Drop all data"""
```

### Graph Storage ABC

```python
@dataclass
class BaseGraphStorage(StorageNameSpace, ABC):
    """Abstract interface for graph databases"""

    @abstractmethod
    async def has_node(self, node_id: str) -> bool:
        """Check if node exists"""

    @abstractmethod
    async def has_edge(self, source_node_id: str, target_node_id: str) -> bool:
        """Check if edge exists"""

    @abstractmethod
    async def get_node(self, node_id: str) -> dict | None:
        """Get node by ID"""

    @abstractmethod
    async def node_degree(self, node_id: str) -> int:
        """Get node degree (number of edges)"""

    @abstractmethod
    async def edge_degree(self, src_id: str, tgt_id: str) -> int:
        """Get edge degree"""

    @abstractmethod
    async def get_edge(
        self,
        source_node_id: str,
        target_node_id: str
    ) -> dict | None:
        """Get edge between nodes"""

    @abstractmethod
    async def get_node_edges(
        self,
        source_node_id: str
    ) -> list[tuple[str, str]]:
        """Get all edges from node"""

    @abstractmethod
    async def upsert_node(self, node_id: str, node_data: dict) -> None:
        """Insert or update node"""

    @abstractmethod
    async def upsert_edge(
        self,
        source_node_id: str,
        target_node_id: str,
        edge_data: dict
    ) -> None:
        """Insert or update edge"""
```

---

## Concrete Implementations

### Example: NanoVectorDB

```python
# From lightrag/storage/nano_vdb.py

class NanoVectorDBStorage(BaseVectorStorage):
    """In-memory vector database implementation"""

    def __post_init__(self):
        self._client_file_name = os.path.join(
            self.workspace,
            f"vdb_{self.namespace}.json"
        )
        self._client = NanoVectorDB(
            self.embedding_func,
            storage_file=self._client_file_name
        )

    async def query(self, query: str, top_k: int, ...) -> list[dict]:
        """Implement abstract method"""
        if query_embedding is None:
            query_embedding = await self.embedding_func([query])
            query_embedding = query_embedding[0]

        results = self._client.query(query_embedding, top_k=top_k)
        return results

    async def upsert(self, data: dict[str, dict]) -> None:
        """Implement abstract method"""
        embeddings = await self.embedding_func(
            [dp["content"] for dp in data.values()]
        )

        for i, (key, value) in enumerate(data.items()):
            self._client.upsert(
                [(key, value, embeddings[i])]
            )

    async def index_done_callback(self) -> None:
        """Implement abstract method"""
        # Save to disk
        self._client.save()
```

### Example: Neo4j Graph Storage

```python
# From lightrag/kg/neo4j_impl.py

class Neo4jStorage(BaseGraphStorage):
    """Neo4j graph database implementation"""

    def __post_init__(self):
        self._driver = GraphDatabase.driver(
            uri=self.global_config["neo4j_uri"],
            auth=(username, password)
        )

    async def has_node(self, node_id: str) -> bool:
        """Implement abstract method"""
        query = "MATCH (n {id: $node_id}) RETURN count(n) > 0 as exists"
        result = await self._run_query(query, node_id=node_id)
        return result[0]["exists"]

    async def upsert_node(self, node_id: str, node_data: dict) -> None:
        """Implement abstract method"""
        query = """
        MERGE (n {id: $node_id})
        SET n += $node_data
        """
        await self._run_query(query, node_id=node_id, node_data=node_data)

    async def get_node_edges(self, source_node_id: str) -> list[tuple]:
        """Implement abstract method"""
        query = """
        MATCH (n {id: $node_id})-[r]->(m)
        RETURN m.id as target
        """
        results = await self._run_query(query, node_id=source_node_id)
        return [(source_node_id, r["target"]) for r in results]
```

---

## Benefits

### 1. Pluggability

```python
# Easy to swap backends
rag_nano = LightRAG(
    vector_db_storage_cls_kwargs={
        "cls": NanoVectorDBStorage,
        "kwargs": {}
    }
)

rag_milvus = LightRAG(
    vector_db_storage_cls_kwargs={
        "cls": MilvusVectorDBStorage,
        "kwargs": {"host": "localhost", "port": 19530}
    }
)

# Same code works with both!
```

### 2. Consistent API

```python
# Regardless of backend, same interface
async def process(vector_db: BaseVectorStorage):
    # Works with ANY vector storage implementation
    results = await vector_db.query("test", top_k=5)

    await vector_db.upsert({
        "id1": {"content": "data"}
    })
```

### 3. Testability

```python
# Mock storage for testing
class MockVectorStorage(BaseVectorStorage):
    def __init__(self):
        self.data = {}

    async def query(self, query, top_k):
        return list(self.data.values())[:top_k]

    async def upsert(self, data):
        self.data.update(data)

# Use in tests
@pytest.mark.asyncio
async def test_extraction():
    mock_storage = MockVectorStorage()
    result = await extract_entities(chunks, mock_storage)
    assert len(result) > 0
```

---

## Design Decisions

### Why ABC Instead of Protocol?

```python
# Option 1: Protocol (structural typing)
from typing import Protocol

class VectorStorageProtocol(Protocol):
    async def query(self, ...): ...

# Option 2: ABC (nominal typing) ✅ CHOSEN
from abc import ABC, abstractmethod

class BaseVectorStorage(ABC):
    @abstractmethod
    async def query(self, ...): ...
```

**Reasons for ABC**:
1. **Explicit contract**: Forces implementation
2. **Runtime checks**: `isinstance(obj, BaseVectorStorage)`
3. **Better IDE support**: Auto-completion, type hints
4. **Inheritance chain**: Shared functionality via mixins

### Why Dataclass?

```python
@dataclass
class BaseVectorStorage(ABC):
    namespace: str
    workspace: str
    embedding_func: EmbeddingFunc
```

**Benefits**:
- Auto-generated `__init__`
- Type hints enforced
- Default values support
- `__post_init__` hook for initialization

---

## Extension Points

### Adding Custom Storage

```python
# 1. Subclass appropriate ABC
class MyCustomVectorDB(BaseVectorStorage):

    def __post_init__(self):
        # Initialize your custom DB
        self.client = MyDBClient(...)

    async def query(self, query: str, top_k: int, ...) -> list[dict]:
        # Implement using your DB's API
        embedding = await self.embedding_func([query])
        results = self.client.search(embedding[0], limit=top_k)
        return self._format_results(results)

    async def upsert(self, data: dict) -> None:
        # Implement upsert
        for key, value in data.items():
            embedding = await self.embedding_func([value["content"]])
            self.client.insert(key, embedding[0], value)

    async def index_done_callback(self) -> None:
        # Commit/flush if needed
        self.client.flush()

    async def drop(self) -> dict[str, str]:
        # Clean up
        self.client.clear()
        return {"status": "success"}

# 2. Use it
rag = LightRAG(
    vector_db_storage_cls_kwargs={
        "cls": MyCustomVectorDB,
        "kwargs": {"custom_param": "value"}
    }
)
```

---

## Related Patterns

- **Strategy Pattern**: Storage selection is a strategy
- **Factory Pattern**: Storage creation (see `lightrag.py:__post_init__`)
- **Adapter Pattern**: Each implementation adapts specific DB API

---

## See Also

- [Strategy Pattern](04-strategy-pattern.md)
- [Layered Architecture](02-layered-architecture.md)

