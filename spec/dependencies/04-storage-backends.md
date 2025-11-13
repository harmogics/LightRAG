# Storage Backends: Системы Хранения Данных

## Обзор

Интеграции с различными storage backends для persistent хранения entities, relations, chunks и embeddings. LightRAG поддерживает **pluggable storage architecture** с множеством опций.

## Storage Architecture

```
┌─────────────────────────────────────────────────┐
│         LightRAG Storage Abstraction            │
├─────────────────────────────────────────────────┤
│                                                 │
│  BaseKVStorage  BaseVectorStorage  BaseGraphStorage
│       ↓               ↓                  ↓      │
│   ┌───────┐      ┌────────┐        ┌─────────┐│
│   │ JSON  │      │ NanoVDB│        │NetworkX ││  Default
│   │ Redis │      │ FAISS  │        │  Neo4j  ││  Production
│   │MongoDB│      │ Milvus │        │Memgraph ││  Options
│   │Postgres      │ Qdrant │                   ││
│   └───────┘      └────────┘        └─────────┘ │
└─────────────────────────────────────────────────┘
```

## Vector Databases

### Milvus

**Package**: `pymilvus`
**Type**: Distributed vector database
**License**: Apache 2.0
**Site**: https://milvus.io

#### Features
- Billion-scale vectors
- GPU acceleration (optional)
- Multiple index types (HNSW, IVF, etc.)
- Distributed architecture

#### Usage

**File**: `lightrag/kg/milvus_impl.py`

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType

class MilvusVectorDBStorage(BaseVectorStorage):
    def __init__(self, namespace, ...):
        # Connect to Milvus
        connections.connect(host=host, port=port)

        # Create collection schema
        fields = [
            FieldSchema("id", DataType.VARCHAR, is_primary=True, max_length=512),
            FieldSchema("embedding", DataType.FLOAT_VECTOR, dim=embedding_dim),
            FieldSchema("entity_name", DataType.VARCHAR, max_length=512),
        ]
        schema = CollectionSchema(fields)

        # Create or load collection
        self.collection = Collection(namespace, schema)

        # Create HNSW index for fast search
        index_params = {
            "metric_type": "IP",  # Inner Product (cosine after L2 norm)
            "index_type": "HNSW",
            "params": {"M": 16, "efConstruction": 256}
        }
        self.collection.create_index("embedding", index_params)

    async def query(self, query_text, top_k=10, query_embedding=None):
        query_vector = query_embedding or await self.embedding_func([query_text])[0]

        results = self.collection.search(
            data=[query_vector],
            anns_field="embedding",
            param={"metric_type": "IP", "params": {"ef": 64}},
            limit=top_k,
            output_fields=["id", "entity_name"]
        )

        return [{"id": hit.id, "score": hit.distance, ...} for hit in results[0]]
```

**Use Case**: Production deployments with millions/billions of vectors.

---

### Qdrant

**Package**: `qdrant-client`
**Type**: Vector search engine
**License**: Apache 2.0
**Site**: https://qdrant.tech

#### Features
- Modern REST/gRPC API
- Payload filtering
- Hybrid search
- Easy deployment (Docker)

#### Usage

**File**: `lightrag/kg/qdrant_impl.py`

```python
from qdrant_client import AsyncQdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

class QdrantVectorDBStorage(BaseVectorStorage):
    def __init__(self, namespace, ...):
        self.client = AsyncQdrantClient(host=host, port=port)

        # Create collection
        await self.client.create_collection(
            collection_name=namespace,
            vectors_config=VectorParams(
                size=embedding_dim,
                distance=Distance.COSINE
            )
        )

    async def upsert(self, data):
        points = [
            PointStruct(
                id=id,
                vector=item["embedding"],
                payload={"entity_name": item["entity_name"], ...}
            )
            for id, item in data.items()
        ]

        await self.client.upsert(collection_name=self.namespace, points=points)

    async def query(self, query_text, top_k=10, query_embedding=None):
        query_vector = query_embedding or await self.embedding_func([query_text])[0]

        results = await self.client.search(
            collection_name=self.namespace,
            query_vector=query_vector,
            limit=top_k,
            score_threshold=self.cosine_better_than_threshold
        )

        return [{"id": hit.id, "score": hit.score, **hit.payload} for hit in results]
```

---

### FAISS

**Package**: `faiss-cpu` or `faiss-gpu`
**Type**: Similarity search library (Facebook AI)
**License**: MIT
**Site**: https://github.com/facebookresearch/faiss

#### Features
- Extremely fast (C++/CUDA)
- Multiple index types
- GPU acceleration
- Good for offline search

#### Usage

**File**: `lightrag/kg/faiss_impl.py`

```python
import faiss
import numpy as np

class FAISSVectorDBStorage(BaseVectorStorage):
    def __init__(self, namespace, embedding_dim=768, ...):
        # Create index (inner product for cosine similarity)
        self.index = faiss.IndexFlatIP(embedding_dim)

        # Optional: Add IVF for faster search on large datasets
        # quantizer = faiss.IndexFlatIP(embedding_dim)
        # self.index = faiss.IndexIVFFlat(quantizer, embedding_dim, 100)

        self.id_to_data = {}  # FAISS doesn't store metadata

    async def upsert(self, data):
        vectors = []
        for id, item in data.items():
            vector = np.array(item["embedding"], dtype=np.float32)
            # L2 normalize for cosine similarity
            faiss.normalize_L2(vector.reshape(1, -1))
            vectors.append(vector)
            self.id_to_data[len(self.id_to_data)] = {"id": id, **item}

        vectors = np.vstack(vectors)
        self.index.add(vectors)

    async def query(self, query_text, top_k=10, query_embedding=None):
        query_vector = np.array(query_embedding or await self.embedding_func([query_text])[0])
        faiss.normalize_L2(query_vector.reshape(1, -1))

        distances, indices = self.index.search(query_vector, top_k)

        results = []
        for dist, idx in zip(distances[0], indices[0]):
            if idx != -1:  # Valid result
                data = self.id_to_data[int(idx)]
                results.append({"score": float(dist), **data})

        return results
```

**Use Case**: Large-scale offline processing, GPU acceleration needed.

---

## Graph Databases

### Neo4j

**Package**: `neo4j`
**Type**: Native graph database
**License**: GPL/Commercial
**Site**: https://neo4j.com

#### Features
- ACID transactions
- Cypher query language
- Scalable (billions of nodes/edges)
- Graph algorithms library

#### Usage

**File**: `lightrag/kg/neo4j_impl.py`

```python
from neo4j import AsyncGraphDatabase

class Neo4JStorage(BaseGraphStorage):
    def __init__(self, namespace, ...):
        self.driver = AsyncGraphDatabase.driver(
            uri=neo4j_uri,
            auth=(user, password)
        )

    async def upsert_node(self, node_id, node_data):
        async with self.driver.session() as session:
            await session.run(
                """
                MERGE (n:Entity {id: $node_id})
                SET n.entity_type = $entity_type,
                    n.description = $description,
                    n.source_id = $source_id
                """,
                node_id=node_id,
                entity_type=node_data["entity_type"],
                description=node_data["description"],
                source_id=node_data["source_id"]
            )

    async def get_neighbors(self, node_id):
        async with self.driver.session() as session:
            result = await session.run(
                "MATCH (a:Entity {id: $node_id})-[r]-(b:Entity) RETURN b.id",
                node_id=node_id
            )
            neighbors = [record["b.id"] async for record in result]
            return neighbors
```

**Use Case**: Production graph workloads, complex queries, ACID requirements.

---

## Key-Value Stores

### Redis

**Package**: `redis`
**Type**: In-memory data structure store
**License**: BSD
**Site**: https://redis.io

#### Usage

**File**: `lightrag/kg/redis_impl.py`

```python
from redis.asyncio import Redis

class RedisKVStorage(BaseKVStorage):
    def __init__(self, namespace, ...):
        self.client = Redis(
            host=host,
            port=port,
            db=db,
            decode_responses=False
        )
        self.namespace = namespace

    async def upsert(self, data: dict):
        for key, value in data.items():
            namespaced_key = f"{self.namespace}:{key}"
            await self.client.set(namespaced_key, json.dumps(value))

    async def get_by_id(self, id: str):
        namespaced_key = f"{self.namespace}:{id}"
        value = await self.client.get(namespaced_key)
        return json.loads(value) if value else None
```

**Use Case**: Fast KV lookups, caching, session storage.

---

### MongoDB

**Package**: `motor` (async MongoDB driver)
**Type**: Document database
**License**: SSPL
**Site**: https://www.mongodb.com

#### Usage

**File**: `lightrag/kg/mongo_impl.py`

```python
from motor.motor_asyncio import AsyncIOMotorClient

class MongoKVStorage(BaseKVStorage):
    def __init__(self, namespace, ...):
        self.client = AsyncIOMotorClient(mongo_uri)
        self.db = self.client[database]
        self.collection = self.db[namespace]

    async def upsert(self, data: dict):
        operations = [
            {"replaceOne": {"filter": {"_id": id}, "replacement": value, "upsert": True}}
            for id, value in data.items()
        ]
        await self.collection.bulk_write(operations)

    async def get_by_id(self, id: str):
        doc = await self.collection.find_one({"_id": id})
        return doc if doc else None
```

**Use Case**: Document-oriented data, flexible schema, aggregation pipelines.

---

### PostgreSQL

**Package**: `asyncpg`
**Type**: Relational database
**License**: PostgreSQL License
**Site**: https://www.postgresql.org

#### Usage

**File**: `lightrag/kg/postgres_impl.py`

```python
import asyncpg

class PostgresKVStorage(BaseKVStorage):
    async def initialize_storages(self):
        self.pool = await asyncpg.create_pool(
            host=host,
            port=port,
            database=database,
            user=user,
            password=password
        )

        # Create table
        async with self.pool.acquire() as conn:
            await conn.execute(f"""
                CREATE TABLE IF NOT EXISTS {self.namespace} (
                    id TEXT PRIMARY KEY,
                    data JSONB,
                    workspace TEXT
                )
            """)

    async def upsert(self, data: dict):
        async with self.pool.acquire() as conn:
            await conn.executemany(
                f"INSERT INTO {self.namespace} (id, data, workspace) "
                f"VALUES ($1, $2, $3) "
                f"ON CONFLICT (id) DO UPDATE SET data = EXCLUDED.data",
                [(id, json.dumps(value), self.workspace) for id, value in data.items()]
            )
```

**Use Case**: ACID transactions, complex queries, joins.

---

## Comparison Matrix

| Backend | Type | Scale | Latency | Distributed | ACID | Use Case |
|---------|------|-------|---------|-------------|------|----------|
| **nano-vectordb** | Vector | < 1M | 10-50ms | No | No | Dev/small |
| **FAISS** | Vector | Billions | 1-10ms | No | No | Large offline |
| **Milvus** | Vector | Billions | 5-20ms | Yes | Yes | Production scale |
| **Qdrant** | Vector | Millions | 5-15ms | Yes | Yes | Production |
| **NetworkX** | Graph | < 100K | 10-100ms | No | No | Dev/small |
| **Neo4j** | Graph | Billions | 10-50ms | Yes | Yes | Production graph |
| **Redis** | KV | Millions | < 1ms | Yes | Partial | Fast lookup/cache |
| **MongoDB** | Document | Billions | 5-20ms | Yes | Yes | Flexible schema |
| **PostgreSQL** | Relational | Billions | 10-50ms | Yes | Yes | ACID/complex queries |

---

**Версия**: 1.0
**Дата**: 2025-01-13
