# Storage Architecture: Архитектура Хранения Данных

## Обзор

Storage Architecture в LightRAG представляет собой многоуровневую систему хранения данных, поддерживающую различные backend'ы для разных типов данных. Система обеспечивает гибкость, масштабируемость и возможность выбора оптимального хранилища под конкретные требования.

## Трехкомпонентная Архитектура

```
┌────────────────────────────────────────────────┐
│         LIGHTRAG APPLICATION                   │
└─────────────┬──────────────────────────────────┘
              │
              ▼
┌────────────────────────────────────────────────┐
│         STORAGE ABSTRACTION LAYER              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────┐│
│  │ BaseKVStorage│  │BaseVectorDB  │  │BaseGraph││
│  │              │  │              │  │      ││
│  └──────────────┘  └──────────────┘  └──────┘│
└──────┬──────────────────┬───────────────┬─────┘
       │                  │               │
       ▼                  ▼               ▼
┌─────────────┐  ┌──────────────┐  ┌──────────┐
│ KEY-VALUE   │  │  VECTOR DB   │  │ GRAPH DB │
│  STORAGE    │  │              │  │          │
│             │  │              │  │          │
│• JSON       │  │• NanoDB      │  │• NetworkX│
│• MongoDB    │  │• Milvus      │  │• Neo4j   │
│• PostgreSQL │  │• Qdrant      │  │• MongoDB │
└─────────────┘  └──────────────┘  └──────────┘
```

## 1. Key-Value Storage

### Base Class: `BaseKVStorage`

**Расположение**: `lightrag/base.py:318`

```python
class BaseKVStorage(ABC):
    """
    Абстрактный базовый класс для key-value хранилищ

    Использование:
    - Хранение полных документов
    - Хранение text chunks
    - Хранение entities metadata
    - Хранение relationships metadata
    - Хранение LLM response cache
    """

    @abstractmethod
    async def get_by_id(self, id: str) -> dict | None:
        """Получить значение по ключу"""

    @abstractmethod
    async def get_by_ids(
        self,
        ids: list[str],
        fields: list[str] | None = None
    ) -> list[dict]:
        """Получить множество значений по списку ключей"""

    @abstractmethod
    async def filter_keys(self, ids: set[str]) -> set[str]:
        """
        Фильтрация существующих ключей

        Args:
            ids: Set ключей для проверки

        Returns:
            Set ключей которые НЕ существуют в storage
        """

    @abstractmethod
    async def upsert(self, data: dict[str, dict]):
        """
        Insert или Update данных

        Args:
            data: {key: value_dict}
        """

    @abstractmethod
    async def delete(self, ids: list[str]):
        """Удалить записи по ключам"""

    @abstractmethod
    async def drop(self):
        """Полная очистка storage"""

    @abstractmethod
    async def persist(self):
        """Сохранение изменений на диск (для in-memory implementations)"""
```

### Хранимые Данные

#### 1. Full Documents Storage

```python
# Storage: self.full_docs
{
    "doc-abc123": {
        "content": "Full document text...",
        "file_path": "path/to/document.pdf",
        "created_at": 1705012345
    }
}
```

#### 2. Text Chunks Storage

```python
# Storage: self.text_chunks
{
    "chunk-xyz789": {
        "content": "Chunk text content...",
        "tokens": 512,
        "chunk_order_index": 0,
        "full_doc_id": "doc-abc123",
        "file_path": "path/to/document.pdf",
        "created_at": 1705012345
    }
}
```

#### 3. Entities Metadata Storage

```python
# Storage: self.full_entities_storage
{
    "ent-def456": {
        "entity_name": "Apple Inc",
        "entity_type": "organization",
        "description": "Comprehensive entity description...",
        "source_id": "chunk-xyz789,chunk-abc123",
        "file_path": "document.pdf",
        "created_at": 1705012345
    }
}
```

#### 4. Relationships Metadata Storage

```python
# Storage: self.full_relations_storage
{
    "rel-ghi012": {
        "source_entity": "Tim Cook",
        "target_entity": "Apple Inc",
        "keywords": "leadership, management, CEO",
        "description": "Relationship description...",
        "source_id": "chunk-xyz789",
        "file_path": "document.pdf",
        "created_at": 1705012345,
        "weight": 1.0
    }
}
```

#### 5. LLM Response Cache

```python
# Storage: self.llm_response_cache
{
    "extract:chunk-xyz789:md5hash": {
        "response": "LLM output text...",
        "timestamp": 1705012345,
        "cache_type": "extract",
        "chunk_id": "chunk-xyz789"
    },
    "summary:Apple Inc:md5hash": {
        "response": "Merged summary...",
        "timestamp": 1705012346,
        "cache_type": "summary",
        "chunk_id": "Apple Inc"
    }
}
```

### Реализации

#### JsonKVStorage

```python
class JsonKVStorage(BaseKVStorage):
    """
    JSON file-based key-value storage

    Features:
    - Simple file-based persistence
    - In-memory cache for fast access
    - Periodic flushing to disk
    - Good for small-medium datasets
    """

    def __init__(self, namespace: str, global_config: dict):
        working_dir = global_config["working_dir"]
        self.file_path = os.path.join(working_dir, f"{namespace}.json")
        self.data = {}  # In-memory cache
        self._load()

    async def upsert(self, data: dict[str, dict]):
        self.data.update(data)
        # Lazy persist - actual write happens in persist()
```

#### MongoKVStorage

```python
class MongoKVStorage(BaseKVStorage):
    """
    MongoDB-based key-value storage

    Features:
    - Scalable for large datasets
    - Built-in replication and sharding
    - Rich query capabilities
    - ACID transactions support
    """

    def __init__(self, namespace: str, global_config: dict):
        client = AsyncIOMotorClient(global_config["mongo_uri"])
        db_name = global_config.get("mongo_db", "lightrag")
        self.collection = client[db_name][namespace]

    async def upsert(self, data: dict[str, dict]):
        operations = [
            UpdateOne(
                {"_id": key},
                {"$set": value},
                upsert=True
            )
            for key, value in data.items()
        ]
        await self.collection.bulk_write(operations)
```

#### PGKVStorage

```python
class PGKVStorage(BaseKVStorage):
    """
    PostgreSQL-based key-value storage

    Features:
    - ACID compliance
    - Strong consistency guarantees
    - Advanced indexing capabilities
    - JSON/JSONB support for flexible schemas
    """

    def __init__(self, namespace: str, global_config: dict):
        self.pool = await asyncpg.create_pool(global_config["pg_uri"])
        self.table_name = namespace

        # Create table if not exists
        await self._create_table()

    async def upsert(self, data: dict[str, dict]):
        async with self.pool.acquire() as conn:
            await conn.executemany(
                f"""
                INSERT INTO {self.table_name} (id, data)
                VALUES ($1, $2)
                ON CONFLICT (id) DO UPDATE SET data = $2
                """,
                [(k, json.dumps(v)) for k, v in data.items()]
            )
```

## 2. Vector Database Storage

### Base Class: `BaseVectorStorage`

**Расположение**: `lightrag/base.py:218`

```python
class BaseVectorStorage(ABC):
    """
    Абстрактный базовый класс для vector хранилищ

    Использование:
    - Semantic search по chunks
    - Semantic search по entities
    - Semantic search по relationships
    """

    @abstractmethod
    async def upsert(self, data: dict[str, dict]):
        """
        Upsert векторов с embeddings

        Args:
            data: {
                id: {
                    "content": str,  # Текст для embedding
                    "entity_name": str,  # Для entities
                    "source_id": str,
                    ...
                }
            }

        Процесс:
        1. Генерация embeddings для каждого content
        2. Upsert векторов с metadata
        """

    @abstractmethod
    async def query(
        self,
        query_text: str,
        top_k: int = 10
    ) -> list[dict]:
        """
        Semantic search

        Args:
            query_text: Запрос для поиска
            top_k: Количество результатов

        Returns:
            [
                {
                    "id": str,
                    "score": float,
                    "content": str,
                    ...metadata...
                }
            ]
        """

    @abstractmethod
    async def delete_entity(self, entity_name: str):
        """Удалить entity по имени"""

    @abstractmethod
    async def get_by_ids(self, ids: list[str]) -> list[dict]:
        """Получить записи по ID"""
```

### Embedding Generation

```python
# В процессе upsert автоматически генерируются embeddings
async def upsert(self, data: dict[str, dict]):
    # Извлечение текстов для embedding
    texts = [item["content"] for item in data.values()]

    # Генерация embeddings батчами
    embeddings = await self.embedding_func(texts)

    # Сохранение vectors с metadata
    for (item_id, item_data), embedding in zip(data.items(), embeddings):
        await self._store_vector(
            id=item_id,
            vector=embedding,
            metadata=item_data
        )
```

### Реализации

#### NanoVectorDBStorage

```python
class NanoVectorDBStorage(BaseVectorStorage):
    """
    In-memory vector database

    Features:
    - Fast in-memory search
    - No external dependencies
    - Good for development and small datasets
    - Cosine similarity search
    """

    def __init__(self, namespace: str, global_config: dict):
        self.vectors = {}  # {id: (vector, metadata)}
        self.embedding_func = global_config["embedding_func"]

    async def query(self, query_text: str, top_k: int = 10):
        # Generate query embedding
        query_vector = await self.embedding_func([query_text])[0]

        # Compute cosine similarity
        scores = []
        for item_id, (vector, metadata) in self.vectors.items():
            similarity = cosine_similarity(query_vector, vector)
            scores.append((item_id, similarity, metadata))

        # Sort by score and return top_k
        scores.sort(key=lambda x: x[1], reverse=True)
        return [
            {"id": id, "score": score, **metadata}
            for id, score, metadata in scores[:top_k]
        ]
```

#### MilvusVectorDBStorage

```python
class MilvusVectorDBStorage(BaseVectorStorage):
    """
    Milvus vector database storage

    Features:
    - Highly scalable distributed vector DB
    - Multiple index types (IVF, HNSW, etc.)
    - GPU acceleration support
    - Production-ready performance
    """

    def __init__(self, namespace: str, global_config: dict):
        from pymilvus import connections, Collection

        connections.connect(
            host=global_config["milvus_host"],
            port=global_config["milvus_port"]
        )
        self.collection = Collection(namespace)
        self.embedding_func = global_config["embedding_func"]

    async def query(self, query_text: str, top_k: int = 10):
        query_vector = await self.embedding_func([query_text])[0]

        results = self.collection.search(
            data=[query_vector],
            anns_field="embedding",
            param={"metric_type": "COSINE", "params": {"nprobe": 10}},
            limit=top_k
        )

        return [
            {
                "id": hit.id,
                "score": hit.score,
                **hit.entity.to_dict()
            }
            for hit in results[0]
        ]
```

#### QdrantVectorDBStorage

```python
class QdrantVectorDBStorage(BaseVectorStorage):
    """
    Qdrant vector database storage

    Features:
    - Modern vector search engine
    - Rich filtering capabilities
    - Efficient on-disk storage
    - Advanced payload indexing
    """

    def __init__(self, namespace: str, global_config: dict):
        from qdrant_client import AsyncQdrantClient

        self.client = AsyncQdrantClient(
            url=global_config["qdrant_url"]
        )
        self.collection_name = namespace
        self.embedding_func = global_config["embedding_func"]

    async def query(self, query_text: str, top_k: int = 10):
        query_vector = await self.embedding_func([query_text])[0]

        results = await self.client.search(
            collection_name=self.collection_name,
            query_vector=query_vector,
            limit=top_k
        )

        return [
            {
                "id": result.id,
                "score": result.score,
                **result.payload
            }
            for result in results
        ]
```

## 3. Graph Database Storage

### Base Class: `BaseGraphStorage`

**Расположение**: `lightrag/base.py:359`

```python
class BaseGraphStorage(ABC):
    """
    Абстрактный базовый класс для graph хранилищ

    Использование:
    - Хранение entities (nodes)
    - Хранение relationships (edges)
    - Graph traversal queries
    """

    @abstractmethod
    async def upsert_node(
        self,
        node_id: str,
        node_data: dict
    ):
        """
        Insert или Update узла графа

        Args:
            node_id: entity_name
            node_data: {
                "entity_name": str,
                "entity_type": str,
                "description": str,
                ...
            }
        """

    @abstractmethod
    async def upsert_edge(
        self,
        source_id: str,
        target_id: str,
        edge_data: dict
    ):
        """
        Insert или Update ребра графа

        Args:
            source_id: source entity_name
            target_id: target entity_name
            edge_data: {
                "keywords": str,
                "description": str,
                "weight": float,
                ...
            }
        """

    @abstractmethod
    async def get_node(self, node_id: str) -> dict | None:
        """Получить узел по ID"""

    @abstractmethod
    async def get_edge(
        self,
        source_id: str,
        target_id: str
    ) -> dict | None:
        """Получить ребро между двумя узлами"""

    @abstractmethod
    async def get_knowledge_graph(
        self,
        entity_names: list[str],
        depth: int = 2,
        max_nodes: int = 100
    ) -> dict:
        """
        Получить подграф начиная с entities

        Args:
            entity_names: Начальные entities
            depth: Глубина BFS traversal
            max_nodes: Максимум узлов в результате

        Returns:
            {
                "nodes": [node_data, ...],
                "edges": [edge_data, ...]
            }
        """
```

### Реализации

#### NetworkXStorage

```python
class NetworkXStorage(BaseGraphStorage):
    """
    NetworkX-based graph storage

    Features:
    - Pure Python implementation
    - Rich graph algorithms
    - Good for small-medium graphs
    - In-memory with JSON persistence
    """

    def __init__(self, namespace: str, global_config: dict):
        import networkx as nx

        self.graph = nx.Graph()  # Undirected graph
        self.namespace = namespace
        self.working_dir = global_config["working_dir"]

    async def upsert_node(self, node_id: str, node_data: dict):
        self.graph.add_node(node_id, **node_data)

    async def upsert_edge(
        self,
        source_id: str,
        target_id: str,
        edge_data: dict
    ):
        self.graph.add_edge(source_id, target_id, **edge_data)

    async def get_knowledge_graph(
        self,
        entity_names: list[str],
        depth: int = 2,
        max_nodes: int = 100
    ):
        # BFS traversal from starting entities
        visited = set()
        queue = [(e, 0) for e in entity_names]
        nodes = []
        edges = []

        while queue and len(visited) < max_nodes:
            node_id, current_depth = queue.pop(0)

            if node_id in visited or current_depth > depth:
                continue

            visited.add(node_id)
            node_data = dict(self.graph.nodes[node_id])
            nodes.append({"id": node_id, **node_data})

            # Add neighbors to queue
            if current_depth < depth:
                for neighbor in self.graph.neighbors(node_id):
                    if neighbor not in visited:
                        queue.append((neighbor, current_depth + 1))

                        # Add edge
                        edge_data = dict(
                            self.graph.edges[node_id, neighbor]
                        )
                        edges.append({
                            "source": node_id,
                            "target": neighbor,
                            **edge_data
                        })

        return {"nodes": nodes, "edges": edges}
```

#### Neo4jStorage

```python
class Neo4jStorage(BaseGraphStorage):
    """
    Neo4j graph database storage

    Features:
    - Native graph database
    - Cypher query language
    - ACID transactions
    - Highly scalable for large graphs
    """

    def __init__(self, namespace: str, global_config: dict):
        from neo4j import AsyncGraphDatabase

        self.driver = AsyncGraphDatabase.driver(
            global_config["neo4j_uri"],
            auth=(
                global_config["neo4j_user"],
                global_config["neo4j_password"]
            )
        )
        self.namespace = namespace

    async def upsert_node(self, node_id: str, node_data: dict):
        async with self.driver.session() as session:
            await session.run(
                """
                MERGE (n:Entity {name: $name})
                SET n += $properties
                """,
                name=node_id,
                properties=node_data
            )

    async def upsert_edge(
        self,
        source_id: str,
        target_id: str,
        edge_data: dict
    ):
        async with self.driver.session() as session:
            await session.run(
                """
                MATCH (s:Entity {name: $source})
                MATCH (t:Entity {name: $target})
                MERGE (s)-[r:RELATED_TO]->(t)
                SET r += $properties
                """,
                source=source_id,
                target=target_id,
                properties=edge_data
            )

    async def get_knowledge_graph(
        self,
        entity_names: list[str],
        depth: int = 2,
        max_nodes: int = 100
    ):
        async with self.driver.session() as session:
            result = await session.run(
                """
                MATCH path = (start:Entity)-[*..%d]-(end:Entity)
                WHERE start.name IN $entity_names
                WITH nodes(path) as nodes, relationships(path) as rels
                UNWIND nodes as n
                WITH collect(DISTINCT n) as all_nodes, rels
                RETURN all_nodes[..%d] as nodes, rels
                """ % (depth, max_nodes),
                entity_names=entity_names
            )

            record = await result.single()
            nodes = [dict(n) for n in record["nodes"]]
            edges = [dict(r) for r in record["rels"]]

            return {"nodes": nodes, "edges": edges}
```

## Storage Initialization

### LightRAG Configuration

```python
class LightRAG:
    def __init__(
        self,
        working_dir: str = "./lightrag_cache",
        # KV Storage
        kv_storage: str = "JsonKVStorage",
        # Vector Storage
        vector_storage: str = "NanoVectorDBStorage",
        embedding_func: callable = None,
        # Graph Storage
        graph_storage: str = "NetworkXStorage",
        # ...
    ):
        self.working_dir = working_dir
        self.kv_storage = kv_storage
        self.vector_storage = vector_storage
        self.graph_storage = graph_storage

        # Initialize storage backends
        asyncio.run(self.initialize_storages())

    async def initialize_storages(self):
        # KV Storages
        kv_storage_class = locate(f"lightrag.storage.{self.kv_storage}")

        self.full_docs = kv_storage_class("full_docs", self.global_config)
        self.text_chunks = kv_storage_class("text_chunks", self.global_config)
        self.full_entities_storage = kv_storage_class(
            "full_entities", self.global_config
        )
        self.full_relations_storage = kv_storage_class(
            "full_relations", self.global_config
        )
        self.llm_response_cache = kv_storage_class(
            "llm_cache", self.global_config
        )

        # Vector Storages
        vector_storage_class = locate(
            f"lightrag.storage.{self.vector_storage}"
        )

        self.chunks_vdb = vector_storage_class("chunks", self.global_config)
        self.entity_vdb = vector_storage_class("entities", self.global_config)
        self.relationships_vdb = vector_storage_class(
            "relationships", self.global_config
        )

        # Graph Storage
        graph_storage_class = locate(
            f"lightrag.storage.{self.graph_storage}"
        )

        self.knowledge_graph_inst = graph_storage_class(
            "knowledge_graph", self.global_config
        )
```

## Storage Combinations

### Development Setup

```python
rag = LightRAG(
    working_dir="./dev_cache",
    kv_storage="JsonKVStorage",           # File-based
    vector_storage="NanoVectorDBStorage", # In-memory
    graph_storage="NetworkXStorage"       # In-memory
)
```

### Production Setup (Small-Medium Scale)

```python
rag = LightRAG(
    working_dir="./prod_cache",
    kv_storage="MongoKVStorage",          # MongoDB
    vector_storage="QdrantVectorDBStorage", # Qdrant
    graph_storage="Neo4JStorage"          # Neo4j
)
```

### Production Setup (Large Scale)

```python
rag = LightRAG(
    working_dir="./prod_cache",
    kv_storage="PGKVStorage",             # PostgreSQL
    vector_storage="MilvusVectorDBStorage", # Milvus
    graph_storage="Neo4JStorage"          # Neo4j
)
```

## Следующий Этап

Теперь перейдем к детальному описанию **[LLM Agents](06-llm-agents.md)** - ролей и функций языковых агентов в pipeline.
