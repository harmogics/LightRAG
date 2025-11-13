# Semantic Traversal Methods: Методы Обхода Семантического Пространства

## Концептуальная Парадигма

LightRAG использует **три концептуально различных метода** семантического обхода, каждый оптимизирован для разных типов queries и balance между скоростью, полнотой и точностью:

1. **Naive Traversal**: Vector-only (no graph structure)
2. **Local Traversal**: Entity-centric + limited hop radius
3. **Global Traversal**: Multi-hop + community detection

Каждый метод представляет разную **философию навигации** по семантическому пространству.

## Method 1: Naive Traversal (Прямой Векторный Поиск)

### Концепция

**Naive traversal** = **pure vector space navigation** без использования graph structure.

```
Query
  ↓ [Embedding]
Query Vector
  ↓ [Cosine Similarity Search]
Top-K Chunks (ranked by similarity)
  ↓ [LLM]
Answer
```

**Философия**: "Текст говорит сам за себя" - релевантность определяется чисто семантической близостью в continuous space.

### Архитектура

```python
async def naive_traversal(query, chunks_vdb, top_k=10):
    """
    Naive method: Direct vector search, no graph.

    Metaphor: Casting a net in semantic ocean,
    catching closest floating chunks.
    """

    # Project query to vector space
    query_embedding = await embedding_func([query])

    # Similarity search in continuous space
    results = await chunks_vdb.query(
        query_text=query,
        query_embedding=query_embedding[0],
        top_k=top_k
    )

    # results = chunks sorted by cosine similarity
    # No graph structure, no entity resolution
    return results
```

**Код** (lightrag/operate.py:3866):
```python
async def naive_query(...):
    """
    Naive query mode: chunks-only, no knowledge graph.

    Semantic Navigation Method:
    • Pure vector space traversal
    • No entity attractors
    • Direct similarity-based retrieval
    """

    # Vector search (no entity resolution)
    chunks = await _get_vector_context(
        query,
        chunks_vdb,
        query_param
    )

    # Assemble context (chunks only, no entities/relations)
    context = await _assemble_query_context(
        chunks_vdb.global_config,
        query_param,
        chunks=chunks,
        entities=[],  # ← No entities
        relations=[]   # ← No relations
    )

    return await _generate_answer(query, context, ...)
```

### Свойства Naive Traversal

```
✅ Advantages:
   • Fast (50-150ms)
   • Simple (no graph complexity)
   • Good for factual lookups

❌ Disadvantages:
   • No multi-hop reasoning
   • No entity-centric context
   • Limited for complex queries
   • Misses structural relationships

📊 Performance:
   • Latency: 50-150ms
   • Precision: 0.6-0.7
   • Recall: 0.5-0.6
```

### Use Cases

```python
good_for_naive = [
    "What is the capital of France?",        # Simple fact
    "Define machine learning",               # Direct definition
    "When was iPhone released?",             # Specific date
]

not_good_for_naive = [
    "How are Apple and Samsung related?",    # Requires entity graph
    "Compare iPhone and Galaxy features",    # Multi-entity reasoning
    "What led to the AI revolution?",        # Multi-hop causal chain
]
```

---

## Method 2: Local Traversal (Локальный Обход от Аттракторов)

### Концепция

**Local traversal** = **entity-centric navigation** с ограниченным радиусом (1-2 hops) от seed attractors.

```
Query
  ↓ [Keywords Extraction]
Keywords (high + low level)
  ↓ [Entity Resolution]
Seed Entities (attractors)
  ↓ [Local Subgraph Extraction: 1-2 hops]
Entity Neighborhood
  ↓ [Chunk Retrieval]
Context
  ↓ [LLM]
Answer
```

**Философия**: "Знание структурировано вокруг концепций" - navigate by activating concept-attractors and exploring local neighborhood.

### Архитектура

```python
async def local_traversal(
    query,
    entities_vdb,
    knowledge_graph_inst,
    text_chunks_db,
    depth=2,
    max_nodes=50
):
    """
    Local method: Entity-centric with limited expansion.

    Metaphor: Drop anchor at entity-attractor,
    explore immediate neighborhood within radius.
    """

    # 1. Extract keywords (intent decomposition)
    hl_keywords, ll_keywords = await extract_keywords(query)

    # 2. Entity resolution (activate attractors)
    seed_entities = await entities_vdb.query(
        " ".join(ll_keywords),  # Use low-level for local
        top_k=10
    )

    # 3. Local subgraph extraction (BFS with depth limit)
    subgraph = await knowledge_graph_inst.get_subgraph(
        entity_names=[e["entity_name"] for e in seed_entities],
        depth=depth,        # ← Limited radius (1-2 hops)
        max_nodes=max_nodes
    )

    # 4. Chunk retrieval (materialize concepts)
    chunks = []
    for entity in subgraph["nodes"]:
        entity_chunks = await get_chunks_by_entity(
            entity["source_id"],
            text_chunks_db
        )
        chunks.extend(entity_chunks)

    # 5. Deduplicate and rank
    unique_chunks = deduplicate(chunks)
    return {
        "entities": subgraph["nodes"],
        "relations": subgraph["edges"],
        "chunks": unique_chunks
    }
```

**Код** (lightrag/operate.py:2696):
```python
if query_param.mode == "local" and len(ll_keywords) > 0:
    """
    Local mode: Entity-centric + 1-2 hop expansion.

    Semantic Navigation Method:
    • Activate local attractors (entities)
    • Explore immediate neighborhood
    • Limited depth prevents explosion
    """

    local_entities, local_relations = await _get_node_data(
        ll_keywords,           # ← Use low-level keywords
        knowledge_graph_inst,
        entities_vdb,
        query_param,
    )

    # local_entities = seed attractors
    # local_relations = 1-hop relations from seeds
    # (internally, _get_node_data does BFS with depth=1-2)
```

### BFS Traversal Algorithm

```python
async def local_bfs_traversal(seed_entities, depth=2, max_nodes=50):
    """
    Breadth-First Search from seed attractors.

    Algorithm:
    1. Start from seed entities (depth 0)
    2. For each entity at depth d < max_depth:
       a. Get neighbors
       b. Add to queue at depth d+1
    3. Stop when depth > max_depth or nodes > max_nodes
    """

    visited = set()
    queue = deque([(entity, 0) for entity in seed_entities])
    nodes = []
    edges = []

    while queue and len(visited) < max_nodes:
        current_entity, current_depth = queue.popleft()

        if current_entity in visited or current_depth > depth:
            continue

        visited.add(current_entity)
        nodes.append(await get_node_data(current_entity))

        # Expand neighborhood if within depth
        if current_depth < depth:
            neighbors = await get_neighbors(current_entity)
            for neighbor in neighbors:
                if neighbor not in visited:
                    queue.append((neighbor, current_depth + 1))
                    edges.append(await get_edge(current_entity, neighbor))

    return {"nodes": nodes, "edges": edges}
```

### Свойства Local Traversal

```
✅ Advantages:
   • Entity-focused (high precision)
   • Structured context (explicit relations)
   • Moderate speed (150-300ms)
   • Good for "What/Who/Where" queries

❌ Disadvantages:
   • Limited to local neighborhood
   • May miss distant relevant entities
   • Depth limit prevents deep reasoning

📊 Performance:
   • Latency: 150-300ms
   • Precision: 0.75-0.85
   • Recall: 0.70-0.80
   • Entities: 5-15
   • Hops: 1-2
```

### Use Cases

```python
good_for_local = [
    "What products does Apple make?",         # Entity + 1-hop relations
    "Who founded Apple Inc?",                 # Entity + founder relation
    "Where is Apple headquartered?",          # Entity + location
    "Tell me about iPhone features",          # Product entity + attributes
]

not_good_for_local = [
    "How has AI transformed healthcare?",     # Too broad, needs global
    "Compare tech industry across decades",   # Temporal + multi-entity
]
```

---

## Method 3: Global Traversal (Глобальное Исследование Графа)

### Концепция

**Global traversal** = **comprehensive graph exploration** с deep multi-hop traversal + community detection.

```
Query
  ↓ [Keywords Extraction]
Keywords (hierarchical: high + low)
  ↓ [Entity Resolution]
Seed Entities
  ↓ [Deep Subgraph Extraction: 3+ hops]
Large Subgraph
  ↓ [Community Detection]
Thematic Clusters
  ↓ [Community Ranking]
Relevant Communities
  ↓ [Comprehensive Chunk Retrieval]
Rich Context
  ↓ [LLM]
Detailed Answer
```

**Философия**: "Знание interconnected через тематические кластеры" - explore broad semantic landscape, identify themes, synthesize comprehensive answer.

### Архитектура

```python
async def global_traversal(
    query,
    entities_vdb,
    relationships_vdb,
    knowledge_graph_inst,
    text_chunks_db,
    depth=3,
    max_nodes=200
):
    """
    Global method: Comprehensive graph exploration.

    Metaphor: Satellite view of semantic landscape,
    identify continents (communities), zoom into relevant ones.
    """

    # 1. Hierarchical keywords extraction
    hl_keywords, ll_keywords = await extract_keywords(query)

    # 2. Seed entities from HIGH-LEVEL keywords
    seed_entities = await entities_vdb.query(
        " ".join(hl_keywords),  # ← Use high-level for global themes
        top_k=5
    )

    # 3. Deep subgraph extraction (3+ hops)
    large_subgraph = await knowledge_graph_inst.get_subgraph(
        entity_names=[e["entity_name"] for e in seed_entities],
        depth=depth,         # ← Deep exploration (3+)
        max_nodes=max_nodes   # ← Large neighborhood
    )

    # 4. Community detection (find thematic clusters)
    nx_graph = build_networkx_graph(large_subgraph)
    communities = detect_communities_louvain(nx_graph)
    # communities = {
    #     0: ["Apple Inc", "iPhone", "Tim Cook", ...],    # Tech cluster
    #     1: ["Healthcare", "AI", "Diagnosis", ...],      # Healthcare cluster
    #     2: ["Samsung", "Galaxy", "Android", ...],       # Competitor cluster
    # }

    # 5. Rank communities by relevance
    ranked_communities = await rank_communities(
        communities,
        query,
        hl_keywords,
        ll_keywords
    )

    # 6. Retrieve chunks from top-N communities
    all_chunks = []
    for comm_id in ranked_communities[:3]:  # Top 3 communities
        for entity in communities[comm_id]:
            chunks = await get_chunks_by_entity(entity)
            all_chunks.extend(chunks)

    unique_chunks = deduplicate_and_rank(all_chunks, query)

    return {
        "entities": large_subgraph["nodes"],
        "relations": large_subgraph["edges"],
        "communities": communities,
        "chunks": unique_chunks
    }
```

**Код** (lightrag/operate.py:2704):
```python
elif query_param.mode == "global" and len(hl_keywords) > 0:
    """
    Global mode: Comprehensive exploration via relationships.

    Semantic Navigation Method:
    • Use HIGH-LEVEL keywords (themes)
    • Search relationship_vdb (not entity_vdb)
    • Extract large subgraph (3+ hops)
    • Community detection for thematic clustering
    """

    global_relations, global_entities = await _get_edge_data(
        hl_keywords,          # ← Use high-level keywords
        knowledge_graph_inst,
        relationships_vdb,    # ← Search relations, not entities!
        query_param,
    )

    # global_relations = thematic relationship clusters
    # global_entities = entities from these relations
```

### Community Detection

```python
def detect_communities_louvain(graph):
    """
    Detect thematic communities in subgraph.

    Algorithm: Louvain method (modularity optimization)
    - Iteratively merge nodes into communities
    - Maximize modularity (within-community edges)
    """
    import networkx as nx
    import community as community_louvain

    # Convert to NetworkX graph
    G = nx.Graph()
    for entity in graph["nodes"]:
        G.add_node(entity["entity_name"])
    for relation in graph["edges"]:
        G.add_edge(relation["src_id"], relation["tgt_id"])

    # Detect communities
    partition = community_louvain.best_partition(G)

    # Group entities by community
    communities = {}
    for entity, comm_id in partition.items():
        if comm_id not in communities:
            communities[comm_id] = []
        communities[comm_id].append(entity)

    return communities
```

### Свойства Global Traversal

```
✅ Advantages:
   • Comprehensive coverage
   • Multi-hop reasoning (3+ hops)
   • Thematic clustering (communities)
   • Good for complex, broad queries

❌ Disadvantages:
   • Slow (500ms - 2s for graph + LLM)
   • Expensive (large context → LLM)
   • May include irrelevant info
   • Complex implementation

📊 Performance:
   • Latency: 500-2000ms
   • Precision: 0.70-0.80
   • Recall: 0.80-0.90
   • Entities: 20-100
   • Communities: 3-10
   • Hops: 3+
```

### Use Cases

```python
good_for_global = [
    "How has AI transformed healthcare?",       # Multi-domain, causal
    "Compare Apple and Samsung ecosystems",     # Multi-entity, comprehensive
    "What are the trends in tech industry?",    # Thematic, broad
    "Explain the history of smartphones",       # Temporal, multi-hop
]

not_good_for_global = [
    "What is Apple's stock price?",            # Simple fact (use naive)
    "Who is Apple's CEO?",                     # Simple relation (use local)
]
```

---

## Comparative Analysis

### Table: Traversal Methods Comparison

| Aspect | Naive | Local | Global |
|--------|-------|-------|--------|
| **Graph Usage** | No graph | Limited (1-2 hops) | Deep (3+ hops) |
| **Entity Resolution** | No | Yes (low-level keywords) | Yes (high-level keywords) |
| **Traversal Depth** | 0 | 1-2 | 3+ |
| **Semantic Space** | Vector only | Hybrid (vector + graph) | Hybrid (vector + graph + communities) |
| **Latency** | 50-150ms | 150-300ms | 500-2000ms |
| **Precision** | 0.6-0.7 | 0.75-0.85 | 0.70-0.80 |
| **Recall** | 0.5-0.6 | 0.70-0.80 | 0.80-0.90 |
| **Complexity** | Low | Medium | High |
| **Use Case** | Simple facts | Entity queries | Complex reasoning |

### Visual Comparison

```
Naive Traversal:
    Query → [Embedding] → Chunks
    (direct line, no graph)

Local Traversal:
    Query → [Entity Resolution] → [Apple Inc] ← seed
                                      ↓
                              ┌───────┼────────┐
                           iPhone   Tim Cook  HQ
                           (1-2 hops from seed)

Global Traversal:
    Query → [Entity Resolution] → [Apple Inc, Samsung, AI] ← seeds
                                         ↓
                    ┌────────────────────┼──────────────────┐
                    │                    │                  │
               Community 1          Community 2       Community 3
             (Tech Giants)       (AI Healthcare)    (Mobile Devices)
                 (3+ hops, thematic clustering)
```

---

## Hybrid Mode: Best of All Worlds

LightRAG also supports **Hybrid mode**, combining multiple traversal methods:

```python
async def hybrid_traversal(query, ...):
    """
    Hybrid mode: Combine local + global + naive

    Strategy:
    1. Local entities (low-level keywords)
    2. Global relations (high-level keywords)
    3. Vector chunks (for additional context)
    4. Merge and rank all results
    """

    # Local branch
    if len(ll_keywords) > 0:
        local_entities, local_relations = await local_traversal(...)

    # Global branch
    if len(hl_keywords) > 0:
        global_entities, global_relations = await global_traversal(...)

    # Vector branch (optional)
    vector_chunks = await naive_traversal(...)

    # Merge results
    merged_entities = deduplicate(local_entities + global_entities)
    merged_relations = deduplicate(local_relations + global_relations)
    merged_chunks = deduplicate(vector_chunks + entity_chunks)

    return comprehensive_context
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
**См. также**: [Search Patterns](../nodes/04-search-patterns.md), [Query Transformations](../transform/02-query-transforms.md)
