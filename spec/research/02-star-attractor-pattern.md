# Star Attractor Pattern: Звездная Архитектура с Семантическими Аттракторами

## Концептуальная Парадигма

Knowledge Graph в LightRAG организован как **constellation of semantic attractors** (созвездие семантических аттракторов), где:

- **Entities** = аттракторы (центры притяжения в семантическом пространстве)
- **Relations** = лучи (rays/spokes), исходящие от аттракторов
- **Query** = навигационный зонд, активирующий ближайшие аттракторы
- **Subgraph extraction** = расширение вдоль лучей от активированного аттрактора

## Архитектура Звездного Паттерна

### Базовая Звездная Структура (Single Star)

```
                    [Context Chunks]
                           │
                           │ source_id
                           ↓
           ┌─────── [Entity: Apple Inc] ───────┐
           │              │ (Hub/Attractor)    │
           │              │                     │
       produces        founded_by           located_in
           │              │                     │
           ↓              ↓                     ↓
      [iPhone]    [Steve Jobs]         [California]
     (Spoke 1)     (Spoke 2)              (Spoke 3)
```

**Ключевые элементы**:
- **Hub (центр)**: Entity с высоким degree (много relations)
- **Spokes (лучи)**: Relations, исходящие от hub
- **Peripheral nodes**: Entities на концах лучей
- **Attractor field**: Semantic neighborhood вокруг hub

### Созвездие Аттракторов (Constellation of Stars)

```
        [Tim Cook]
             │ CEO_of
             ↓
[iPhone] ←─ [Apple Inc] ──→ [California]
  │                              │
  │ competes_with                │
  ↓                              ↓
[Galaxy] ←─ [Samsung] ──→ [South Korea]
             ↑
             │ CEO_of
        [Lee Jae-yong]
```

**Свойства созвездия**:
- **Multiple attractors**: Множественные entity-hubs
- **Cross-connections**: Relations между разными звездами
- **Hierarchical structure**: Major attractors (Apple, Samsung) + minor (products, people)
- **Semantic clusters**: Близкие аттракторы образуют communities

## Концептуальные Свойства Аттракторов

### Свойство 1: Semantic Gravity (Семантическое Притяжение)

Entities с высокой "semantic mass" притягивают:

1. **Relations** (incoming + outgoing edges)
2. **Chunks** (text manifestations via source_id)
3. **Query attention** (high cosine similarity with queries)

**Метрики semantic mass**:
```python
semantic_mass(entity) =
    α * degree(entity) +              # Graph centrality
    β * chunk_count(entity) +          # Text groundings
    γ * embedding_quality(entity)      # Semantic richness
```

**Код** (lightrag/operate.py:3051):
```python
async def _get_node_data(keywords, knowledge_graph_inst, entities_vdb, query_param):
    """
    Activate attractors based on semantic similarity.
    Entities with high similarity = strong attractors for this query.
    """

    # Vector search activates nearest attractors
    results = await entities_vdb.query(keywords, top_k=query_param.top_k)

    # results = activated attractors, sorted by semantic gravity
    # (cosine similarity score)
```

### Свойство 2: Hub-Spoke Topology

**Hub entities** (major attractors):
```python
hub_characteristics = {
    "high_degree": "> 10 relations",
    "central_role": "organization, person, location",
    "frequent_mentions": "> 5 source chunks",
    "semantic_importance": "high betweenness centrality"
}

examples = ["Apple Inc", "Steve Jobs", "United States", "AI"]
```

**Spoke relations** (rays from hubs):
```python
spoke_characteristics = {
    "directionality": "from hub → peripheral or bidirectional",
    "semantic_types": ["produces", "founded_by", "located_in", ...],
    "strength": "based on relation description richness"
}
```

### Свойство 3: Multi-hop Radiation (Многошаговое Излучение)

От активированного аттрактора "излучение" распространяется по лучам:

```
Depth 0: [Apple Inc] ← Seed attractor (activated by query)
              │
              └─── produces ───→ [iPhone]
                                      │
                                      └─── uses ───→ [A17 Chip]
                                                          │
                                                          └─── made_by ───→ [TSMC]

Depth:   0                1                2                   3
```

**Radiation decay**:
```python
relevance_score(entity, depth) = initial_score * decay_factor^depth

# Example:
initial_score("Apple Inc") = 0.92  # From vector search
relevance_score("iPhone", depth=1) = 0.92 * 0.9 = 0.828
relevance_score("A17 Chip", depth=2) = 0.92 * 0.9^2 = 0.745
relevance_score("TSMC", depth=3) = 0.92 * 0.9^3 = 0.671
```

## Звездные Паттерны в LightRAG

### Pattern 1: Single-Hub (Entity-Centric Query)

**Запрос**: "Tell me about Apple Inc"

```
Query activates SINGLE attractor:
                [Apple Inc]
                     │
        ┌────────────┼────────────┐
        │            │            │
    produces     founded_by    located_in
        │            │            │
        ↓            ↓            ↓
   [iPhone]    [Steve Jobs]  [California]
   [iPad]      [Tim Cook]
   [Mac]
```

**Характеристики**:
- **Depth**: 1-hop (local mode)
- **Activated attractors**: 1 primary (Apple Inc)
- **Peripheral nodes**: Products, people, locations
- **Answer focus**: Comprehensive info about central entity

### Pattern 2: Multi-Hub (Comparative Query)

**Запрос**: "Compare Apple and Samsung products"

```
Query activates MULTIPLE attractors:

    [iPhone] ←─ [Apple Inc] ──→ [California]
                     ↕ competes_with
    [Galaxy] ←─ [Samsung] ──→ [South Korea]
```

**Характеристики**:
- **Depth**: 2-hop (hybrid mode)
- **Activated attractors**: 2 primary (Apple, Samsung)
- **Cross-connections**: competes_with relation bridges stars
- **Answer focus**: Comparative analysis

### Pattern 3: Constellation (Broad Thematic Query)

**Запрос**: "How has AI revolutionized technology?"

```
Query activates CONSTELLATION of attractors:

[Machine Learning] ←── influences ──→ [Deep Learning]
         │                                    │
    applied_in                           applied_in
         │                                    │
         ↓                                    ↓
   [Healthcare] ←─── transforms ───→ [Computer Vision]
         │                                    │
         ↓                                    ↓
   [Diagnosis]                           [Recognition]
```

**Характеристики**:
- **Depth**: 3+ hops (global mode)
- **Activated attractors**: Multiple themes (AI, ML, Healthcare, Vision)
- **Community detection**: Clusters of related attractors
- **Answer focus**: Comprehensive thematic analysis

## Аттракторная Динамика

### Activation Phase (Фаза Активации)

```python
# Step 1: Query projects into vector space
query_embedding = embedding_func(query_keywords)

# Step 2: Similarity search activates nearest attractors
activated_attractors = []
for entity in all_entities:
    similarity = cosine(query_embedding, entity.embedding)
    if similarity >= threshold:
        activated_attractors.append((entity, similarity))

# Step 3: Sort by semantic gravity
activated_attractors.sort(key=lambda x: x[1], reverse=True)

# Result: Ranked list of activated attractors
# [("Apple Inc", 0.92), ("iPhone", 0.87), ("Steve Jobs", 0.84), ...]
```

### Expansion Phase (Фаза Расширения)

```python
# Step 1: Start from seed attractors
seed_entities = [a[0] for a in activated_attractors[:top_k]]

# Step 2: BFS traversal along rays (relations)
subgraph = extract_subgraph_bfs(seed_entities, depth=max_depth)

# Step 3: Collect peripheral nodes
peripheral_nodes = []
for entity in subgraph.nodes:
    if entity not in seed_entities:
        peripheral_nodes.append(entity)

# Result: Expanded star/constellation
# Central attractors + rays + peripheral nodes
```

**Код** (lightrag/operate.py:206):
```python
async def extract_local_subgraph(seed_entities, depth, max_nodes):
    """
    BFS traversal from seed attractors (hubs).
    Follows rays (relations) to discover peripheral nodes.
    """

    visited = set()
    queue = [(entity, 0) for entity in seed_entities]

    while queue and len(visited) < max_nodes:
        current_entity, current_depth = queue.pop(0)

        if current_entity in visited or current_depth > depth:
            continue

        visited.add(current_entity)
        nodes.append(get_node_data(current_entity))

        # Get neighbors (follow rays)
        if current_depth < depth:
            neighbors = get_neighbors(current_entity)
            for neighbor in neighbors:
                queue.append((neighbor, current_depth + 1))
                edges.append(get_edge(current_entity, neighbor))

    return {"nodes": nodes, "edges": edges}
```

### Materialization Phase (Фаза Материализации)

```python
# Step 1: For each entity in subgraph, retrieve chunks
materialized_context = []
for entity in subgraph.nodes:
    # Entity = abstract concept (attractor)
    # Chunks = concrete manifestations
    chunks = get_chunks_by_source_id(entity.source_id)
    materialized_context.extend(chunks)

# Step 2: Deduplicate and rank chunks
unique_chunks = deduplicate(materialized_context)
ranked_chunks = rank_by_relevance(unique_chunks, query)

# Result: Grounded text context from attractor neighborhood
```

## Аналогии с Другими Системами

### Аналогия 1: Star Schema в Data Warehousing

```
Fact Table (центр) ←→ Entity Hub
      ↓
Dimension Tables (rays) ←→ Related Entities

Пример:
Sales Fact ←→ [Apple Inc]
  ↓ dimensions          ↓ relations
Product, Time, Location ←→ Products, Dates, Locations
```

### Аналогия 2: Hub-Spoke Network Topology

```
Airport Hub (центр) ←→ Entity Hub
      ↓
Routes (rays) ←→ Relations
      ↓
Spoke Cities ←→ Peripheral Entities

Пример:
Atlanta (hub) ←→ [Apple Inc]
  ↓ routes           ↓ produces
NYC, LA, Miami ←→ iPhone, iPad, Mac
```

### Аналогия 3: Attractor Basins в Динамических Системах

```
Attractor State ←→ Entity Hub
      ↑
Trajectories (flows) ←→ Relations
      ↑
Initial States ←→ Peripheral Entities

Система "стекается" к attractor states,
так же как query "стекается" к релевантным entity hubs.
```

## Практические Импликации

### Оптимизация 1: Hub Identification и Prioritization

**Identify hubs** (major attractors):
```python
def identify_hubs(graph, threshold_degree=10):
    """Find entities that are major attractors (hubs)."""
    hubs = []
    for entity in graph.nodes:
        if entity.degree >= threshold_degree:
            hubs.append(entity)
    return sorted(hubs, key=lambda e: e.degree, reverse=True)

# Cache hub subgraphs for fast retrieval
hub_cache = {
    "Apple Inc": precomputed_subgraph_1hop,
    "Steve Jobs": precomputed_subgraph_1hop,
    ...
}
```

### Оптимизация 2: Adaptive Depth по Attractor Density

```python
def adaptive_depth(seed_entities, graph):
    """
    Adjust expansion depth based on attractor density.
    High density (many relations) → shallow depth
    Low density (few relations) → deeper depth
    """
    avg_degree = mean([graph.degree(e) for e in seed_entities])

    if avg_degree > 20:  # Dense hubs
        return 1  # Limited expansion
    elif avg_degree > 10:
        return 2  # Moderate expansion
    else:  # Sparse nodes
        return 3  # Deep expansion to find context
```

### Оптимизация 3: Relation Strength Weighting

```python
# Weight rays (relations) by semantic richness
relation_weight = {
    "produces": 1.0,      # Strong semantic connection
    "related_to": 0.5,    # Weak semantic connection
    "mentions": 0.3       # Very weak
}

# During BFS, prioritize strong rays
priority_queue = sorted(
    neighbors,
    key=lambda n: relation_weight[n.relation_type],
    reverse=True
)
```

## Связь с Топологией Графа

**См**: [Graph Topology](../nodes/03-graph-topology.md)

```python
# Hub nodes = high degree centrality
hubs = identify_hubs(graph, threshold=10)

# Star patterns = subgraphs around hubs
for hub in hubs:
    star_subgraph = extract_1hop_neighbors(hub)
    # star_subgraph has star topology with hub at center

# Betweenness centrality identifies critical connectors
connectors = identify_high_betweenness(graph)
# Connectors bridge different star clusters
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
**См. также**: [Semantic Traversal Methods](04-semantic-traversal-methods.md), [Graph Topology](../nodes/03-graph-topology.md)
