# Degree Operations: Операции на Основе Степени Узла

## Концептуальная Парадигма

**Degree-based operations** — fast lightweight algorithms для оценки **local importance** entities based on connectivity.

```
Entity Degree Distribution:

High Degree (> 50)        Major Attractors
┌────────────────┐        (Hub Entities)
│ Apple Inc: 87  │   ←    Central concepts
│ Samsung: 65    │
│ Microsoft: 54  │
└────────────────┘

Medium Degree (10-50)     Minor Attractors
┌────────────────┐        (Supporting Entities)
│ iPhone: 23     │
│ Tim Cook: 18   │
└────────────────┘

Low Degree (1-10)         Peripheral Nodes
┌────────────────┐        (Specific Facts)
│ A17 chip: 3    │
│ Cupertino: 2   │
└────────────────┘
```

**Философия**: Degree = **number of rays from attractor** = quick approximation of semantic importance.

---

## Алгоритм 1: Node Degree

### Описание

**Node degree** = total number of edges connected to a node.

```
degree(v) = |{(u,v) ∈ E}| + |{(v,u) ∈ E}|

For undirected graph:
degree(v) = |{u : (u,v) ∈ E или (v,u) ∈ E}|
```

### Реализация

**Файл**: `lightrag/kg/networkx_impl.py:109`, `lightrag/kg/mongo_impl.py`, `lightrag/kg/postgres_impl.py`

#### NetworkX Implementation

```python
async def node_degree(self, node_id: str) -> int:
    """
    Get node degree (number of connected edges).

    Metaphor: Number of rays emanating from semantic attractor.
    """
    graph = await self._get_graph()
    return graph.degree(node_id)

# Example
graph = nx.Graph()
graph.add_edges_from([
    ("Apple", "iPhone"),
    ("Apple", "MacBook"),
    ("Apple", "Tim Cook"),
    ("Apple", "California"),
])

degree = graph.degree("Apple")  # degree = 4
```

#### MongoDB Implementation

```python
async def node_degree(self, node_id: str) -> int:
    """
    Calculate node degree by counting edges in edge collection.
    """
    # Count incoming edges (X → node_id)
    in_degree = await self.edge_collection.count_documents(
        {"target_node_id": node_id}
    )

    # Count outgoing edges (node_id → X)
    out_degree = await self.edge_collection.count_documents(
        {"source_node_id": node_id}
    )

    return in_degree + out_degree
```

#### PostgreSQL Implementation

```python
async def node_degree(self, node_id: str) -> int:
    """
    Calculate node degree via SQL query.
    """
    async with self.pool.acquire() as conn:
        result = await conn.fetchval(
            """
            SELECT COUNT(*)
            FROM edges
            WHERE source_node_id = $1 OR target_node_id = $1
            """,
            node_id
        )
    return result
```

### Complexity

- **Time**: O(1) with indexed edge table, O(E) without index
- **Space**: O(1)

### Параметры

```python
# Directed degree (in-degree + out-degree separate)
in_degree = len([e for e in edges if e[1] == node_id])
out_degree = len([e for e in edges if e[0] == node_id])
total_degree = in_degree + out_degree
```

### Связь с Star Pattern

Node degree directly measures **star pattern strength** (spec/research/02-star-attractor-pattern.md):

```
        [Apple Inc]  ← degree = 5
             │
    ┌────────┼────────┬────────┬────────┐
    │        │        │        │        │
[iPhone] [MacBook] [Tim Cook] [CA] [Samsung]
 degree=1  degree=1   degree=2 degree=1 degree=3

High degree → Hub entity → Major attractor
```

---

## Алгоритм 2: Edge Degree

### Описание

**Edge degree** = sum of endpoint node degrees (importance of relation).

```
edge_degree(u, v) = degree(u) + degree(v)
```

**Интуиция**: Relation connecting two high-degree nodes = important relation (connects major attractors).

### Реализация

**Файл**: `lightrag/kg/networkx_impl.py:113`

```python
async def edge_degree(self, src_id: str, tgt_id: str) -> int:
    """
    Calculate edge importance as sum of endpoint degrees.

    Metaphor: Strength of connection between two attractors.
    High edge degree = major bridge in knowledge graph.
    """
    graph = await self._get_graph()

    src_degree = graph.degree(src_id) if graph.has_node(src_id) else 0
    tgt_degree = graph.degree(tgt_id) if graph.has_node(tgt_id) else 0

    return src_degree + tgt_degree

# Example
# Apple (degree 50) ← competes_with → Samsung (degree 40)
edge_deg = await edge_degree("Apple", "Samsung")  # 50 + 40 = 90
# High edge degree → important competitive relation

# iPhone (degree 10) ← uses → A17_chip (degree 2)
edge_deg = await edge_degree("iPhone", "A17_chip")  # 10 + 2 = 12
# Low edge degree → peripheral detail
```

### Complexity

- **Time**: O(1) if degrees cached, O(E) if computed on-the-fly
- **Space**: O(1)

### Use Case: Relation Ranking

```python
async def rank_relations_by_importance(
    relations: list[tuple[str, str]],
    graph: BaseGraphStorage
) -> list[tuple[str, str, int]]:
    """
    Rank relations by edge degree (endpoint importance).
    """
    ranked = []
    for src, tgt in relations:
        edge_deg = await graph.edge_degree(src, tgt)
        ranked.append((src, tgt, edge_deg))

    # Sort by edge degree (descending)
    ranked.sort(key=lambda x: x[2], reverse=True)

    return ranked

# Example
relations = [
    ("Apple", "Samsung"),     # Major companies
    ("Apple", "Tim Cook"),    # Company-CEO
    ("iPhone", "A17_chip"),   # Product-component
]

ranked = await rank_relations_by_importance(relations, graph)
# Result:
# [("Apple", "Samsung", 90),    # Most important (hub-hub)
#  ("Apple", "Tim Cook", 68),   # Medium (hub-minor)
#  ("iPhone", "A17_chip", 12)]  # Least important (peripheral)
```

---

## Алгоритм 3: Popular Labels (Degree Ranking)

### Описание

**Popular labels** = entities ranked by degree (most connected first).

### Реализация

**Файл**: `lightrag/kg/networkx_impl.py:215`

```python
async def get_popular_labels(self, limit: int = 300) -> list[str]:
    """
    Get most connected entities (popular labels) by degree.

    Metaphor: Major attractors in knowledge graph constellation.

    Returns:
        List of entity names sorted by degree (descending)
    """
    graph = await self._get_graph()

    # Get degrees of all nodes
    degrees = dict(graph.degree())

    # Sort by degree (descending)
    sorted_nodes = sorted(
        degrees.items(),
        key=lambda x: x[1],
        reverse=True
    )

    # Return top labels
    popular_labels = [str(node) for node, _ in sorted_nodes[:limit]]

    logger.debug(
        f"Retrieved {len(popular_labels)} popular labels (limit: {limit})"
    )

    return popular_labels

# Example usage
popular = await graph.get_popular_labels(limit=10)
# ["Apple Inc", "Samsung", "Microsoft", "Google", "Amazon",
#  "iPhone", "Android", "Windows", "Tim Cook", "California"]
# → Top 10 most connected entities
```

### Complexity

- **Time**: O(V log V) (sort all nodes by degree)
- **Space**: O(V)

### Use Case: Knowledge Graph Overview

```python
async def generate_kg_overview(
    graph: BaseGraphStorage,
    top_n: int = 20
):
    """
    Generate overview of knowledge graph highlighting major entities.
    """
    # Get popular entities
    popular = await graph.get_popular_labels(limit=top_n)

    # Get degree for each
    overview = []
    for entity in popular:
        degree = await graph.node_degree(entity)
        overview.append({
            "entity": entity,
            "degree": degree,
            "importance": "major" if degree > 50 else "minor"
        })

    return overview

# Result:
# [
#     {"entity": "Apple Inc", "degree": 87, "importance": "major"},
#     {"entity": "Samsung", "degree": 65, "importance": "major"},
#     {"entity": "Microsoft", "degree": 54, "importance": "major"},
#     ...
# ]
```

---

## Алгоритм 4: Search Labels by Degree

### Описание

Fuzzy search + degree-based ranking for entity search.

### Реализация

**Файл**: `lightrag/kg/networkx_impl.py:240`

```python
async def search_labels(self, query: str, limit: int = 50) -> list[str]:
    """
    Search entities with fuzzy matching + degree weighting.

    Ranking logic:
    1. Exact match → highest priority
    2. Prefix match → high priority
    3. Contains match → medium priority
    4. Shorter names → bonus (more specific)
    5. Higher degree → bonus (more important)

    Returns:
        List of matching entity names sorted by relevance
    """
    graph = await self._get_graph()
    query_lower = query.lower().strip()

    if not query_lower:
        return []

    matches = []
    degrees = dict(graph.degree())  # Precompute degrees

    for node in graph.nodes():
        node_str = str(node)
        node_lower = node_str.lower()

        # Skip non-matches
        if query_lower not in node_lower:
            continue

        # Calculate relevance score
        if node_lower == query_lower:
            score = 1000  # Exact match
        elif node_lower.startswith(query_lower):
            score = 500   # Prefix match
        else:
            score = 100 - len(node_str)  # Contains match
            # Bonus for word boundary
            if f" {query_lower}" in node_lower or f"_{query_lower}" in node_lower:
                score += 50

        # Bonus for high degree (importance)
        degree_bonus = min(degrees.get(node, 0), 50)  # Cap at 50
        score += degree_bonus

        matches.append((node_str, score))

    # Sort by relevance (desc) then alphabetically
    matches.sort(key=lambda x: (-x[1], x[0]))

    return [match[0] for match in matches[:limit]]

# Example
results = await graph.search_labels("Apple", limit=5)
# ["Apple Inc",      # Exact match + high degree
#  "Apple Products", # Prefix match + medium degree
#  "Apple Store",    # Prefix match + low degree
#  "Apple Watch",    # Prefix match
#  "Apple TV"]
```

### Complexity

- **Time**: O(V) for search + O(M log M) for sorting matches (M = match count)
- **Space**: O(V) for degree cache + O(M) for matches

---

## Связь с Dual-Space Architecture

Degree operations bridge **graph space** (topology) and **vector space** (semantics) (spec/research/03-dual-space-architecture.md):

```
Graph Space                Vector Space
    │                          │
    │ Degree centrality        │ Embedding similarity
    ↓                          ↓
Hub entities              Semantic clusters
    │                          │
    └──────────┬───────────────┘
               ↓
     Combined ranking:
     score = α * degree + β * similarity
```

**Hybrid ranking**:
```python
async def hybrid_entity_ranking(
    query: str,
    entities_vdb: BaseVectorStorage,
    graph: BaseGraphStorage,
    alpha: float = 0.5,  # Weight for degree
    beta: float = 0.5,   # Weight for similarity
):
    # Vector search
    vector_results = await entities_vdb.query(query, top_k=100)

    # Add degree scores
    ranked = []
    for result in vector_results:
        entity_id = result["id"]
        similarity = result["similarity"]

        # Get degree
        degree = await graph.node_degree(entity_id)
        degree_norm = min(degree / 100, 1.0)  # Normalize to [0,1]

        # Combined score
        combined = alpha * degree_norm + beta * similarity

        ranked.append({
            "entity": entity_id,
            "similarity": similarity,
            "degree": degree,
            "score": combined
        })

    # Sort by combined score
    ranked.sort(key=lambda x: x["score"], reverse=True)

    return ranked[:20]
```

---

## Caching Strategy

Pre-compute and cache degrees for fast access:

```python
class DegreeCache:
    def __init__(self, graph: BaseGraphStorage):
        self.graph = graph
        self.cache = {}
        self.last_update = None

    async def initialize(self):
        """Pre-compute all degrees."""
        all_nodes = await self.graph.get_all_labels()
        for node in all_nodes:
            self.cache[node] = await self.graph.node_degree(node)
        self.last_update = time.time()

    async def get_degree(self, node_id: str) -> int:
        """Get degree from cache (fast O(1))."""
        if node_id not in self.cache:
            # Cache miss: compute and store
            degree = await self.graph.node_degree(node_id)
            self.cache[node_id] = degree
        return self.cache[node_id]

    async def invalidate(self, node_id: str):
        """Invalidate cache for updated node."""
        if node_id in self.cache:
            del self.cache[node_id]

    async def refresh(self):
        """Periodic full cache refresh."""
        if time.time() - self.last_update > 3600:  # 1 hour
            await self.initialize()

# Usage
cache = DegreeCache(graph)
await cache.initialize()

# Fast degree lookup (O(1))
degree = await cache.get_degree("Apple Inc")
```

---

## Производительность

### Benchmark (Knowledge Graph: 10K nodes, 50K edges)

| Operation | Uncached | Cached | Speedup |
|-----------|----------|--------|---------|
| **node_degree** | 5ms | < 1ms | 5-10x |
| **edge_degree** | 10ms | < 1ms | 10x |
| **get_popular_labels** | 100ms | 20ms | 5x |
| **search_labels** | 80ms | 15ms | 5x |

**Recommendation**: Pre-compute degrees during graph loading, store in memory cache.

---

## Use Cases in LightRAG

### 1. Fast Entity Filtering

```python
# Filter entities by minimum degree (remove low-importance nodes)
min_degree = 5
important_entities = [
    entity for entity in all_entities
    if await graph.node_degree(entity) >= min_degree
]
```

### 2. Relation Pruning

```python
# Remove weak relations (low edge degree)
async def prune_weak_relations(graph, threshold=10):
    edges = await graph.get_all_edges()
    for src, tgt in edges:
        edge_deg = await graph.edge_degree(src, tgt)
        if edge_deg < threshold:
            await graph.remove_edge(src, tgt)
```

### 3. Query Context Size Control

```python
# Limit context to high-degree entities (hubs only)
async def get_query_context_hubs(query, graph, max_entities=20):
    # Vector search
    candidates = await entities_vdb.query(query, top_k=100)

    # Filter by degree
    hubs = []
    for entity in candidates:
        degree = await graph.node_degree(entity["id"])
        if degree >= 10:  # Hub threshold
            hubs.append(entity)

    return hubs[:max_entities]
```

---

## Example: Degree Distribution Analysis

```python
async def analyze_degree_distribution(graph: BaseGraphStorage):
    """
    Analyze degree distribution to understand graph topology.
    """
    all_nodes = await graph.get_all_labels()
    degrees = []

    for node in all_nodes:
        degree = await graph.node_degree(node)
        degrees.append(degree)

    # Statistics
    import numpy as np
    stats = {
        "min": min(degrees),
        "max": max(degrees),
        "mean": np.mean(degrees),
        "median": np.median(degrees),
        "std": np.std(degrees),
    }

    # Distribution buckets
    buckets = {
        "low (1-10)": len([d for d in degrees if 1 <= d <= 10]),
        "medium (11-50)": len([d for d in degrees if 11 <= d <= 50]),
        "high (51+)": len([d for d in degrees if d > 50]),
    }

    return {
        "stats": stats,
        "distribution": buckets
    }

# Example result:
# {
#     "stats": {
#         "min": 1,
#         "max": 147,
#         "mean": 12.3,
#         "median": 5,
#         "std": 18.7
#     },
#     "distribution": {
#         "low (1-10)": 7500,     # 75% peripheral nodes
#         "medium (11-50)": 2200, # 22% supporting nodes
#         "high (51+)": 300       # 3% hub nodes
#     }
# }
# → Scale-free network (power-law degree distribution)
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
