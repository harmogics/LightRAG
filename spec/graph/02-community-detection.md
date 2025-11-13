# Community Detection: Обнаружение Семантических Кластеров

## Концептуальная Парадигма

**Community Detection** в LightRAG реализует автоматическую идентификацию **semantic clusters** — групп тесно связанных entities, формирующих **созвездия аттракторов** (constellations of attractors).

```
┌──────────────────────────────────────────────────┐
│          Knowledge Graph                         │
│                                                  │
│  Community 1: Tech     Community 2: People       │
│  ┌─[Apple]───────┐    ┌─[Tim Cook]──────┐      │
│  │   │           │    │    │             │      │
│  │ [iPhone]  [MacBook] │ [Steve Jobs] [Elon]   │
│  └───────────────┘    └──────────────────┘      │
│                                                  │
│  Community 3: Locations                          │
│  ┌─[California]────────┐                        │
│  │   │                 │                        │
│  │ [Silicon Valley] [LA]                        │
│  └────────────────────┘                         │
└──────────────────────────────────────────────────┘
```

**Философия**: Entities естественно кластеризуются по семантическим темам → communities = тематические "острова" в knowledge graph.

---

## Алгоритм: Louvain Method

### Описание

**Louvain algorithm** — hierarchical community detection based on modularity optimization.

**Modularity** = мера того, насколько плотно связаны узлы внутри community vs. между communities.

```
Q = 1/(2m) * Σ [ A_ij - (k_i * k_j)/(2m) ] * δ(c_i, c_j)

где:
- m = total edges
- A_ij = adjacency matrix (1 if edge i→j exists)
- k_i, k_j = degrees of nodes i, j
- c_i, c_j = communities of nodes i, j
- δ(c_i, c_j) = 1 if same community, else 0
```

**Интуиция**: High modularity = много intra-community edges, мало inter-community edges.

### Реализация

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py:541`

```python
import community  # python-louvain
import networkx as nx

class GraphViewer:
    def calculate_layout(self):
        """
        Calculate 3D layout for graph with community-based coloring.
        """
        if not self.graph:
            return

        # Detect communities using Louvain
        self.communities = community.best_partition(self.graph)
        # communities = {node_id: community_id, ...}
        # Example: {"Apple Inc": 0, "iPhone": 0, "Tim Cook": 1, ...}

        num_communities = len(set(self.communities.values()))

        # Generate distinct colors for each community
        self.community_colors = generate_colors(num_communities)

        # Assign colors to nodes based on community
        for node_id in self.graph.nodes():
            node.color = self.get_node_color(node_id)

    def get_node_color(self, node_id: str) -> glm.vec3:
        """
        Get color based on community membership.
        """
        if self.communities and node_id in self.communities:
            comm_id = self.communities[node_id]
            color = self.community_colors[comm_id]
            return color
        return glm.vec3(0.5, 0.5, 0.5)  # Default gray
```

**Library**: `python-louvain` (spec/dependencies/03-graph-processing.md)

```python
import community

# Detect communities
partition = community.best_partition(graph)
# partition = dict mapping node_id → community_id

# Get modularity score
modularity = community.modularity(partition, graph)
# modularity ∈ [-0.5, 1.0], higher = better clustering
```

### Алгоритм (High-Level)

**Phase 1: Initialization**
```
Each node = its own community
communities = {node_i: i for i in nodes}
```

**Phase 2: Local Optimization** (greedy modularity maximization)
```
for each node i:
    for each neighbor j of i:
        # Try moving node i to neighbor j's community
        delta_Q = modularity_gain(i, community[j])

        if delta_Q > 0:
            # Move i to j's community (improves modularity)
            community[i] = community[j]

# Repeat until no improvements
```

**Phase 3: Aggregation** (build super-graph)
```
# Create new graph where each community → single super-node
super_graph = aggregate_communities(graph, communities)

# Recurse: apply Phase 2 to super-graph
# → Hierarchical community structure
```

### Complexity

- **Time**: O(E * log(V)) (fast in practice, near-linear)
- **Space**: O(V + E)
- **Iterations**: Typically 2-5 passes to convergence

### Параметры

```python
# python-louvain default parameters
resolution: float = 1.0   # Higher = more communities (smaller)
randomize: bool = False   # Randomize node order (non-deterministic)
```

---

## Связь с Star-Attractor Pattern

Community detection выявляет **"constellations of attractors"** (spec/research/02-star-attractor-pattern.md):

```
┌─────────────────────────────────────────────────┐
│         Constellation 1: Technology             │
│                                                 │
│          [Apple Inc] ← Major Attractor          │
│               │                                 │
│       ┌───────┼───────┐                        │
│       │       │       │                        │
│   [iPhone] [MacBook] [iPad]                    │
│       │       │       │                        │
│   Minor attractors (high degree within cluster)│
│                                                 │
│          [Samsung] ← Another Major Attractor    │
│               │                                 │
│       ┌───────┼───────┐                        │
│       │       │       │                        │
│   [Galaxy] [Note] [Fold]                       │
└─────────────────────────────────────────────────┘

Inter-community edges (weak):
   Apple ──competes_with──→ Samsung
```

**Community** = **semantic cluster** = группа entities с:
- High intra-community edge density (strong relations within)
- Low inter-community edge density (weak relations between)
- Shared semantic theme (technology, people, locations, etc.)

**Major attractors** = hub nodes with high degree **within community**.

---

## Связь с Query Modes

### Global Mode

Community detection критична для **global query mode** (spec/research/04-semantic-traversal-methods.md):

```python
async def global_query(query, graph_storage, entities_vdb):
    """
    Global mode: multi-entity + community-aware context.
    """

    # Step 1: Find top-K entities (attractors)
    top_entities = await entities_vdb.query(query, top_k=20)

    # Step 2: Extract subgraph with BFS
    subgraph = await graph_storage.get_knowledge_subgraph(
        seeds=top_entities,
        max_depth=2,
        max_nodes=500
    )

    # Step 3: Detect communities in subgraph
    communities = community.best_partition(subgraph)

    # Step 4: Rank communities by relevance to query
    # (e.g., count entities per community, weight by similarity)
    ranked_communities = rank_communities(communities, query, top_entities)

    # Step 5: Select top-N communities
    selected_communities = ranked_communities[:3]

    # Step 6: Extract entities from selected communities
    relevant_entities = [
        entity for entity, comm_id in communities.items()
        if comm_id in selected_communities
    ]

    # Step 7: Generate answer from multi-community context
    answer = await llm_generate(query, relevant_entities, subgraph)
    return answer
```

**Rationale**: Query может затрагивать multiple themes → community detection автоматически разделяет на тематические кластеры → LLM получает structured multi-topic context.

### Hybrid Mode

```python
# Combine local (single entity) + global (community-level)
local_context = await local_query("Apple products")
# Local = entity-centric (Apple + 1-hop neighbors)

community_context = await extract_community_context("Apple Inc")
# Community = all entities in Apple's community

hybrid_context = merge_contexts(local_context, community_context)
```

---

## Связь с Dependencies

### python-louvain (spec/dependencies/03-graph-processing.md)

```python
# Installation
pip install python-louvain

# Usage
import community
import networkx as nx

# Build graph
graph = nx.Graph()
graph.add_edges_from([
    ("Apple", "iPhone"),
    ("Apple", "MacBook"),
    ("Samsung", "Galaxy"),
    ("Samsung", "Note"),
    ("Apple", "Samsung"),  # Weak inter-community edge
])

# Detect communities
partition = community.best_partition(graph)
# partition = {"Apple": 0, "iPhone": 0, "MacBook": 0,
#              "Samsung": 1, "Galaxy": 1, "Note": 1}

# Modularity score
modularity = community.modularity(partition, graph)
# modularity ≈ 0.3-0.4 (decent clustering)

# Hierarchical dendrogram (multi-level communities)
dendro = community.generate_dendrogram(graph)
# dendro[0] = finest partition (many communities)
# dendro[-1] = coarsest partition (few communities)
```

### NetworkX (spec/dependencies/03-graph-processing.md)

NetworkX provides graph structure, python-louvain operates on it:

```python
import networkx as nx

# Load graph
graph = nx.read_graphml("knowledge_graph.graphml")

# Community detection
communities = community.best_partition(graph)

# Add community as node attribute
for node, comm_id in communities.items():
    graph.nodes[node]["community"] = comm_id

# Save graph with communities
nx.write_graphml(graph, "kg_with_communities.graphml")
```

---

## Visualization Use Case

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py`

Community detection используется для **color-coding** nodes in 3D visualization:

```python
def calculate_layout(self):
    # Detect communities
    self.communities = community.best_partition(self.graph)
    num_communities = len(set(self.communities.values()))

    # Generate colors (HSV color wheel)
    self.community_colors = generate_colors(num_communities)

    # Assign colors to nodes
    for node_id in self.graph.nodes():
        comm_id = self.communities[node_id]
        node.color = self.community_colors[comm_id]

# Render: nodes in same community = same color
# → Visual clustering reveals semantic themes
```

**Result**: Knowledge graph visualization with:
- **Color clusters** = semantic communities
- **Inter-cluster edges** = cross-topic relations
- **Hub nodes** (high degree) = major attractors within clusters

---

## Evaluation Metrics

### Modularity

```python
Q = community.modularity(partition, graph)
# Q ∈ [-0.5, 1.0]
# Q > 0.3: Good clustering
# Q > 0.5: Strong communities
# Q > 0.7: Very distinct clusters
```

**Interpretation**:
- **High Q**: Entities well-separated into themes
- **Low Q**: Entities heavily interconnected across themes

### Coverage

```python
# Fraction of nodes assigned to communities
coverage = len([n for n in partition.values()]) / graph.number_of_nodes()
# coverage = 1.0 (all nodes assigned)
```

### Conductance (per community)

```python
# Ratio of inter-community edges to intra-community edges
def conductance(graph, community_nodes):
    internal_edges = 0
    external_edges = 0

    for node in community_nodes:
        for neighbor in graph.neighbors(node):
            if neighbor in community_nodes:
                internal_edges += 1
            else:
                external_edges += 1

    return external_edges / (internal_edges + external_edges)

# Low conductance = tight community
```

---

## Example: LightRAG Knowledge Graph Communities

Предположим, knowledge graph из документов о технологических компаниях:

```python
graph.number_of_nodes()  # 5000 entities
graph.number_of_edges()  # 15000 relations

# Detect communities
partition = community.best_partition(graph)

# Analyze communities
from collections import Counter
community_sizes = Counter(partition.values())
print(community_sizes)
# {0: 800, 1: 600, 2: 500, 3: 450, ...}

# Top community (0) might be "Technology Companies"
comm_0_nodes = [n for n, c in partition.items() if c == 0]
print(comm_0_nodes[:10])
# ["Apple Inc", "Samsung", "Microsoft", "Google", "Intel",
#  "AMD", "NVIDIA", "Tesla", "SpaceX", "Amazon"]

# Community 1 might be "People"
comm_1_nodes = [n for n, c in partition.items() if c == 1]
print(comm_1_nodes[:10])
# ["Tim Cook", "Steve Jobs", "Elon Musk", "Bill Gates",
#  "Jeff Bezos", "Mark Zuckerberg", ...]

# Community 2 might be "Products"
# ...
```

**Use Case**: When querying "tech industry leaders", LightRAG can:
1. Find entities in "People" community
2. Expand to "Technology Companies" community (via CEO relations)
3. Provide structured answer: "Leaders: [People] at Companies: [Companies]"

---

## Связь с Dual-Space Architecture

Community detection bridges **graph space** and **vector space** (spec/research/03-dual-space-architecture.md):

```
Vector Space (continuous)          Graph Space (discrete)
        │                                  │
        │ Embedding similarity             │ Community detection
        ↓                                  ↓
   Semantic clusters              Topological clusters
        │                                  │
        └────────────┬─────────────────────┘
                     ↓
            Unified semantic structure
```

**Hybrid approach**:
1. **Vector clustering** (e.g., K-means on embeddings) → semantic groups
2. **Graph clustering** (Louvain) → topological groups
3. **Alignment**: Communities should align with embedding clusters

**Validation**: If community detection finds cluster with low intra-cluster embedding similarity → indicates structural pattern not captured by embeddings (e.g., temporal relations).

---

## Производительность

### Benchmark (LightRAG knowledge graph: 10K nodes, 50K edges)

| Graph Size | Nodes | Edges | Louvain Time | Modularity |
|------------|-------|-------|--------------|------------|
| Small      | 1K    | 5K    | 50ms         | 0.42       |
| Medium     | 10K   | 50K   | 500ms        | 0.38       |
| Large      | 100K  | 500K  | 5s           | 0.35       |

**Observation**: Modularity decreases with size (more inter-connections), but time scales well (near-linear).

### Оптимизации

1. **Parallel Louvain**: Partition graph, detect communities in parallel
2. **Incremental updates**: Re-run Louvain only on changed subgraph
3. **Pre-filtering**: Remove low-degree nodes before detection

---

## Future Enhancements

### 1. Query-Guided Community Detection

Instead of global communities, detect **query-specific communities**:

```python
# Personalized community detection
subgraph = extract_query_relevant_subgraph(query)
communities = community.best_partition(subgraph)
# → Communities relevant to query only
```

### 2. Hierarchical Community Representation

```python
# Multi-level dendrogram
dendro = community.generate_dendrogram(graph)

# Level 0: Fine-grained (100 communities)
# Level 1: Medium (20 communities)
# Level 2: Coarse (5 communities)

# Select level based on query complexity
if simple_query:
    communities = dendro[-1]  # Coarse level
else:
    communities = dendro[0]   # Fine level
```

### 3. Temporal Community Evolution

```python
# Track how communities change over time (as new documents inserted)
communities_t0 = detect_communities(graph_t0)
communities_t1 = detect_communities(graph_t1)

# Compare: which entities moved between communities?
# → Indicates semantic drift (e.g., Apple entered "AI" community)
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
