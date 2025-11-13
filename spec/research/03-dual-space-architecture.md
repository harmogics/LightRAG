# Dual-Space Architecture: Двойственность Graph и Vector Пространств

## Концептуальная Парадигма

LightRAG функционирует в **двух взаимодополняющих семантических пространствах**:

1. **Graph Space** (Дискретное пространство) - структурированное символическое знание
2. **Vector Space** (Непрерывное пространство) - distributed semantic representations

Эта dual-space architecture критична для:
- **Entity resolution** - маппинг между пространствами
- **Hybrid reasoning** - использование сильных сторон обоих пространств
- **Semantic grounding** - связь абстрактного (graph) и конкретного (vectors)

## Архитектура Двух Пространств

### Graph Space (G): Дискретное Структурированное Пространство

```
Properties:
• Discrete nodes and edges
• Symbolic representation (entity names, relation types)
• Explicit structure (A → B → C)
• Logical traversal (BFS, DFS, path finding)
• Deterministic operations

Representation:
G = (V, E)
where:
- V = {entity₁, entity₂, ..., entityₙ} (nodes)
- E = {(src, rel, tgt)} (directed labeled edges)

Example:
V = {"Apple Inc", "iPhone", "Steve Jobs"}
E = {
    ("Apple Inc", "produces", "iPhone"),
    ("Steve Jobs", "founded", "Apple Inc")
}
```

**Код** (lightrag/base.py:293):
```python
@dataclass
class BaseGraphStorage(StorageNameSpace, ABC):
    """
    Abstract interface for GRAPH SPACE.

    Graph space provides:
    • Structural knowledge representation
    • Explicit relationships between entities
    • Traversal operations (neighbors, paths)
    """

    @abstractmethod
    async def get_node(self, node_id: str) -> dict | None:
        """Retrieve entity from graph space"""

    @abstractmethod
    async def get_edge(self, src: str, tgt: str) -> dict | None:
        """Retrieve relation from graph space"""

    @abstractmethod
    async def get_neighbors(self, node_id: str) -> list[str]:
        """Graph traversal operation"""
```

### Vector Space (V): Непрерывное Семантическое Пространство

```
Properties:
• Continuous embeddings in ℝⁿ (n = 768, 1536, 4096, ...)
• Distributed representations (meaning spread across dimensions)
• Soft boundaries (gradient-based similarity)
• Similarity operations (cosine, dot product)
• Non-deterministic (query-dependent results)

Representation:
V = ℝⁿ
where:
- Each entity e → vector v(e) ∈ ℝⁿ
- Similarity: sim(v₁, v₂) = cos(θ) = (v₁·v₂) / (||v₁|| ||v₂||)

Example:
v("Apple Inc") = [0.234, -0.567, 0.123, ..., 0.891] (768-dim)
v("iPhone") = [0.256, -0.512, 0.145, ..., 0.823]
sim(v("Apple Inc"), v("iPhone")) = 0.87
```

**Код** (lightrag/base.py:218):
```python
@dataclass
class BaseVectorStorage(StorageNameSpace, ABC):
    """
    Abstract interface for VECTOR SPACE.

    Vector space provides:
    • Semantic similarity search
    • Continuous representations
    • Soft matching (fuzzy boundaries)
    """

    embedding_func: EmbeddingFunc

    @abstractmethod
    async def query(
        self,
        query: str,
        top_k: int,
        query_embedding: list[float] = None
    ) -> list[dict[str, Any]]:
        """Similarity search in vector space"""
```

## Dual-Space Operations

### Operation 1: Projection (Graph → Vector)

Entities в graph space проецируются в vector space:

```python
# Graph space entity
entity_graph = {
    "entity_name": "Apple Inc",
    "entity_type": "organization",
    "description": "American multinational technology company..."
}

# Projection to vector space
entity_text = f"{entity_graph['entity_name']} {entity_graph['description']}"
entity_vector = await embedding_func(entity_text)

# Result: entity exists in BOTH spaces
# Graph space: discrete node with relations
# Vector space: continuous embedding for similarity search
```

**Код** (lightrag/operate.py:1883):
```python
async def _insert_entity_embedding(
    entity_name: str,
    entity_description: str,
    embedding_func: callable,
    entities_vdb: BaseVectorStorage
):
    """
    PROJECT entity from graph space to vector space.
    """
    # Prepare text for embedding
    entity_text = f"Entity: {entity_name}\nDescription: {entity_description}"

    # Project to vector space
    embedding = await embedding_func(entity_text)

    # Store in vector space (for similarity search)
    await entities_vdb.upsert({
        entity_name: {
            "embedding": embedding,
            "entity_name": entity_name,
            "description": entity_description
        }
    })
```

### Operation 2: Resolution (Vector → Graph)

Query в vector space резолвится в entities в graph space:

```python
# Query in vector space
query = "innovative products from Apple"
query_embedding = await embedding_func(query)

# Similarity search in vector space
vector_results = await entities_vdb.query(
    query_text=query,
    query_embedding=query_embedding,
    top_k=10
)

# Results: entities ranked by similarity
# [("Apple Inc", 0.92), ("iPhone", 0.87), ("iPad", 0.84), ...]

# Resolution to graph space
entity_ids = [result["entity_name"] for result in vector_results]

# Now we have discrete entity IDs for graph traversal
subgraph = await knowledge_graph.get_subgraph(entity_ids, depth=2)
```

**Код** (lightrag/operate.py:2697):
```python
async def _get_node_data(
    keywords: list[str],
    knowledge_graph_inst: BaseGraphStorage,
    entities_vdb: BaseVectorStorage,
    query_param: QueryParam,
):
    """
    RESOLUTION OPERATION: Vector space → Graph space

    1. Query in vector space (similarity search)
    2. Get entity IDs (discrete graph nodes)
    3. Retrieve full graph data
    """

    # Step 1: Vector space query
    logger.info(f"Query entities: {keywords}")
    results = await entities_vdb.query(keywords, top_k=query_param.top_k)

    # Step 2: Extract entity IDs (bridge to graph space)
    entity_ids = [r["entity_name"] for r in results]

    # Step 3: Retrieve graph data
    entities_graph_data = []
    relations_graph_data = []
    for entity_id in entity_ids:
        # Fetch from graph space
        node = await knowledge_graph_inst.get_node(entity_id)
        entities_graph_data.append(node)

        # Get relations in graph space
        neighbors = await knowledge_graph_inst.get_neighbors(entity_id)
        for neighbor in neighbors:
            edge = await knowledge_graph_inst.get_edge(entity_id, neighbor)
            relations_graph_data.append(edge)

    return entities_graph_data, relations_graph_data
```

### Operation 3: Synchronization (Maintaining Consistency)

Both spaces must stay synchronized:

```python
# When entity is added/updated in graph space:
async def update_entity(entity_name, new_description):
    # 1. Update in graph space
    await knowledge_graph.upsert_node({
        "entity_name": entity_name,
        "description": new_description
    })

    # 2. Re-compute embedding and update vector space
    new_embedding = await embedding_func(
        f"Entity: {entity_name}\nDescription: {new_description}"
    )
    await entities_vdb.upsert({
        entity_name: {"embedding": new_embedding}
    })

    # Now both spaces are consistent
```

## Сравнительный Анализ Пространств

### Таблица Сравнения

| Aspect | Graph Space (G) | Vector Space (V) |
|--------|----------------|------------------|
| **Nature** | Discrete, symbolic | Continuous, distributed |
| **Representation** | Nodes + edges | Embeddings in ℝⁿ |
| **Search** | Traversal (BFS, DFS) | Similarity search (cosine) |
| **Complexity** | O(V + E) for BFS | O(log N) for ANN search |
| **Precision** | High (exact match) | Soft (similarity-based) |
| **Recall** | Limited (must know path) | High (fuzzy matching) |
| **Explainability** | High (explicit paths) | Low (black box embeddings) |
| **Scalability** | Moderate (graph size) | High (ANN indexes) |
| **Use Cases** | Multi-hop reasoning | Semantic search |

### Сильные Стороны Graph Space

```
✅ Explicit structure
   → Can trace exact paths: A → B → C → D

✅ Logical reasoning
   → If A → B and B → C, then A → C (transitivity)

✅ Explainability
   → "Answer comes from path: Apple → produces → iPhone"

✅ Deterministic
   → Same traversal always yields same results
```

### Сильные Стороны Vector Space

```
✅ Semantic similarity
   → Find "Apple products" even if query says "Apple items"

✅ Fuzzy matching
   → Handle typos, synonyms, paraphrases

✅ Gradient-based ranking
   → Results sorted by relevance (not binary)

✅ Fast ANN search
   → O(log N) vs O(V+E) for large graphs
```

## Hybrid Reasoning: Сочетание Пространств

LightRAG использует **hybrid reasoning**, объединяя сильные стороны обоих пространств:

### Hybrid Pattern 1: Vector-First, Graph-Second

```
Query
  ↓
[Vector Space] ← Find semantically similar entities (high recall)
  ↓ (top-K entities)
[Graph Space] ← Traverse from seed entities (structured exploration)
  ↓ (subgraph)
Answer
```

**Преимущества**:
- Vector space обеспечивает **soft entry points** (fuzzy matching)
- Graph space обеспечивает **structured context** (explicit relations)

**Код** (lightrag/operate.py:2696):
```python
# Hybrid: Vector search → Graph traversal
if query_param.mode == "local":
    # Step 1: Vector space (find seed entities)
    local_entities, local_relations = await _get_node_data(
        ll_keywords,
        knowledge_graph_inst,
        entities_vdb,  # ← Vector space query
        query_param,
    )
    # local_entities = seed entities from vector search

    # Step 2: Graph space (expand from seeds)
    # _get_node_data internally does graph traversal:
    # for each seed entity, get neighbors in graph
```

### Hybrid Pattern 2: Graph-Constrained Vector Search

```
Subgraph
  ↓ (extract entity IDs)
[Vector Space] ← Search only among entities in subgraph
  ↓ (ranked entities)
Answer
```

**Преимущества**:
- Graph structure **constrains search space** (avoid irrelevant)
- Vector space **ranks within constraints** (soft ordering)

### Hybrid Pattern 3: Iterative Refinement

```
Query → [Vector] → Seed Entities
         ↓
    [Graph] → Expand Subgraph
         ↓
    [Vector] → Rank expanded entities
         ↓
    [Graph] → Extract final subgraph
         ↓
      Answer
```

**Преимущества**:
- Alternating between spaces refines results
- Vector provides relevance, graph provides structure

## Entity Resolution как Ключевой Мост

**Entity Resolution (T8)** - критический оператор, который маппит между пространствами:

```python
def entity_resolution(query_keywords, entities_vdb, knowledge_graph):
    """
    Bridge operator: Vector space ↔ Graph space

    Input: Keywords (symbolic)
    Processing:
      1. Embed keywords → vector space
      2. Similarity search → ranked entities (vector space)
      3. Extract entity IDs → discrete identifiers
      4. Fetch graph data → graph space
    Output: Entities + relations (graph space)
    """

    # Vector space query
    query_embedding = embedding_func(query_keywords)
    vector_results = entities_vdb.query(
        query_text=query_keywords,
        query_embedding=query_embedding,
        top_k=10
    )

    # Bridge: vector results → graph IDs
    entity_ids = [r["entity_name"] for r in vector_results]

    # Graph space retrieval
    graph_nodes = []
    for entity_id in entity_ids:
        node = knowledge_graph.get_node(entity_id)
        graph_nodes.append(node)

    return graph_nodes  # Now in graph space
```

**См**: [T8: Entity Resolution](../transform/02-query-transforms.md#t8-entity-resolution)

## Математическая Формализация

### Graph Space (G)

```
G = (V, E, L_V, L_E)

where:
- V = finite set of vertices (entities)
- E ⊆ V × V = edges (relations)
- L_V: V → Σ_V = vertex labels (entity names/types)
- L_E: E → Σ_E = edge labels (relation types)

Operations:
- neighbors(v) = {u ∈ V | (v,u) ∈ E}
- path(v_s, v_t) = sequence of edges from v_s to v_t
- subgraph(V' ⊆ V) = induced subgraph on V'
```

### Vector Space (V)

```
V = ℝⁿ

where:
- n = embedding dimension (768, 1536, ...)
- Each entity e → v(e) ∈ ℝⁿ

Operations:
- sim(v₁, v₂) = cos(θ) = (v₁·v₂) / (||v₁|| ||v₂||)
- query(q, k) = argmax_{v ∈ V} top-k sim(q, v)
- distance: d(v₁, v₂) = 1 - sim(v₁, v₂)
```

### Bridge Operators

```
π: G → V     (Projection: graph → vector)
π(e) = embed(L_V(e) + description(e))

ρ: V → G     (Resolution: vector → graph)
ρ(q) = {e ∈ V_G | sim(q, π(e)) ≥ θ}

where:
- π = projection operator
- ρ = resolution operator
- θ = similarity threshold
```

## Практические Импликации

### Оптимизация 1: Lazy Vector Projection

Не все entities нужны в vector space:

```python
# Strategy: Lazy projection
# Only project entities to vector space when first queried

entity_vector_cache = {}

async def get_entity_embedding(entity_id):
    if entity_id not in entity_vector_cache:
        # Lazy projection
        entity = await knowledge_graph.get_node(entity_id)
        embedding = await embedding_func(entity["description"])
        entity_vector_cache[entity_id] = embedding

    return entity_vector_cache[entity_id]
```

### Оптимизация 2: Graph-Aware Vector Indexing

Index vectors по graph clusters:

```python
# Partition entities by graph communities
communities = detect_communities(knowledge_graph)

# Create separate vector indexes per community
vector_indexes = {}
for community_id, entities in communities.items():
    vector_indexes[community_id] = create_vector_index(
        entities=[π(e) for e in entities]
    )

# Query: first identify community, then search in that index
def query_with_community(query):
    community_id = classify_query_community(query)
    return vector_indexes[community_id].query(query)
```

### Оптимизация 3: Hybrid Caching

Cache результаты hybrid operations:

```python
hybrid_cache = {
    "query_hash": {
        "vector_results": [...],  # From vector space
        "entity_ids": [...],       # Bridge
        "subgraph": {...},         # From graph space
        "timestamp": ...
    }
}
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
**См. также**: [Query as Semantic Key](01-query-as-semantic-key.md), [Semantic Traversal](04-semantic-traversal-methods.md)
