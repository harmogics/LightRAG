# Community Detection: Обнаружение Сообществ в Графах

## Концептуальная Парадигма

**Community detection** = unsupervised clustering algorithm для graph data, identifying **dense subgraphs** (communities).

```
Graph → Community Detection → Clusters (Communities)

Entity Graph:                Communities:
   A ─── B                   Community 1: {A, B, C}
   │ ╲   │                   Community 2: {D, E, F}
   │   ╲ │                   Community 3: {G, H}
   C     D ─── E
         │     │
         F ─── G ─── H

Dense connections within communities, sparse between communities.
```

**Философия**: Unsupervised discovery of **semantic clusters** — entities in same community = related topics/concepts.

---

## Algorithm: Louvain Community Detection

### Conceptual Overview

**Louvain algorithm** = greedy modularity optimization для unsupervised graph clustering.

**Modularity (Q)**: Measure of community structure quality.

```
Q = (1/2m) Σ[A_ij - (k_i * k_j)/(2m)] * δ(c_i, c_j)

where:
- A_ij = adjacency matrix (1 if edge exists, 0 otherwise)
- k_i = degree of node i
- m = total edges
- c_i = community of node i
- δ(c_i, c_j) = 1 if c_i == c_j, else 0

Interpretation:
Q = (actual edges within communities) - (expected edges if random)

Range: Q ∈ [-0.5, 1.0]
- Q > 0.3: Strong community structure
- Q < 0.1: Weak/no community structure
```

### Algorithm Steps

```
Phase 1: Local Optimization (Node Movement)
──────────────────────────────────────────
1. Initialize: Each node = own community
   Communities: {A}, {B}, {C}, {D}, ...

2. For each node i:
   a. Compute ΔQ for moving i to each neighbor's community
   b. Move i to community with max ΔQ (if ΔQ > 0)
   c. Repeat until no improvement

Result: Communities after local optimization


Phase 2: Network Aggregation (Community Collapse)
──────────────────────────────────────────────────
1. Collapse each community into single "super-node"
2. Edges between communities → edges between super-nodes
3. Self-loops for internal edges

Result: Coarser graph


Repeat Phase 1 and 2 until modularity Q converges
```

### Modularity Gain Formula

```python
def modularity_gain(node_i, community_c, graph):
    """
    Compute ΔQ for moving node i to community c.

    ΔQ = [Σ_in + k_i,in] / (2m) - [(Σ_tot + k_i) / (2m)]²
         - [Σ_in / (2m) - (Σ_tot / (2m))² - (k_i / (2m))²]

    where:
    - Σ_in = sum of weights inside community c
    - Σ_tot = sum of weights incident to community c
    - k_i = degree of node i
    - k_i,in = sum of weights from i to nodes in c
    - m = total edge weight
    """
    m = graph.size(weight="weight")
    k_i = graph.degree(node_i, weight="weight")

    # Weights from i to nodes in community c
    k_i_in = sum(
        graph[node_i][neighbor].get("weight", 1)
        for neighbor in graph.neighbors(node_i)
        if graph.nodes[neighbor]["community"] == community_c
    )

    # Community statistics
    sigma_in = community_internal_edges(c)
    sigma_tot = community_total_edges(c)

    # ΔQ calculation
    delta_q = (
        (sigma_in + k_i_in) / (2 * m)
        - ((sigma_tot + k_i) / (2 * m)) ** 2
        - (sigma_in / (2 * m) - (sigma_tot / (2 * m)) ** 2 - (k_i / (2 * m)) ** 2)
    )

    return delta_q
```

---

## Implementation in LightRAG

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py:541`

```python
import community as community_louvain

class GraphVisualizer:
    """Graph visualization with community detection."""

    def __init__(self, graph: nx.Graph):
        self.graph = graph
        self.communities = None

    def detect_communities(self):
        """
        Louvain community detection.

        Returns:
            dict: {node_id: community_id}
        """
        # Louvain algorithm
        self.communities = community_louvain.best_partition(self.graph)

        num_communities = len(set(self.communities.values()))
        logger.info(f"Detected {num_communities} communities")

        return self.communities

    def get_community_colors(self):
        """
        Assign colors to communities for visualization.

        Returns:
            dict: {node_id: color}
        """
        if self.communities is None:
            self.detect_communities()

        # Generate distinct colors for each community
        num_communities = len(set(self.communities.values()))
        colors = plt.cm.rainbow(np.linspace(0, 1, num_communities))

        node_colors = {}
        for node, community_id in self.communities.items():
            node_colors[node] = colors[community_id]

        return node_colors

    def visualize_with_communities(self, layout: str = "spring"):
        """
        Visualize graph with community-based coloring.

        Args:
            layout: Layout algorithm ("spring", "circular", etc.)
        """
        # Compute layout
        if layout == "spring":
            pos = nx.spring_layout(self.graph, dim=3, iterations=100)
        else:
            pos = nx.circular_layout(self.graph)

        # Detect communities and assign colors
        node_colors = self.get_community_colors()

        # Plot
        for node in self.graph.nodes():
            x, y, z = pos[node]
            color = node_colors[node]
            plt.scatter(x, y, c=[color], s=100)

        plt.title(f"Graph with {len(set(self.communities.values()))} communities")
        plt.show()
```

**Dependencies**: `python-louvain>=0.16`, `networkx>=2.5`

---

## Integration with LightRAG Query

### Global Query Mode with Communities

**Файл**: spec/research/04-semantic-traversal-methods.md (Global Mode)

```python
async def global_query_with_communities(
    query: str,
    entities_vdb: BaseVectorStorage,
    graph: BaseGraphStorage,
    query_param: QueryParam
):
    """
    Global query mode: Community-aware retrieval.

    Process:
    1. Detect communities in entity graph
    2. For each community, generate summary
    3. Retrieve relevant communities by vector search
    4. Return entities from top communities
    """
    # Step 1: Community detection
    entity_graph = await graph.get_full_graph()

    communities = community_louvain.best_partition(entity_graph)
    num_communities = len(set(communities.values()))

    logger.info(f"Detected {num_communities} communities")

    # Step 2: Generate community summaries (using LLM)
    community_summaries = {}
    for community_id in set(communities.values()):
        # Get entities in this community
        community_entities = [
            node for node, cid in communities.items() if cid == community_id
        ]

        # Concatenate entity descriptions
        entity_descriptions = await graph.get_nodes(community_entities)
        combined_desc = "\n".join([e["description"] for e in entity_descriptions])

        # Summarize using LLM
        summary = await llm_func(
            prompt=f"Summarize the following entities:\n{combined_desc}",
            model="gpt-3.5-turbo",
            max_tokens=256
        )

        community_summaries[community_id] = summary

    # Step 3: Embed community summaries
    summaries = list(community_summaries.values())
    summary_embeddings = await embedding_func(summaries)

    # Store in temporary vector DB
    temp_vdb = {}
    for community_id, summary, embedding in zip(
        community_summaries.keys(), summaries, summary_embeddings
    ):
        temp_vdb[community_id] = {
            "summary": summary,
            "embedding": embedding
        }

    # Step 4: Query community summaries
    query_embedding = await embedding_func([query])

    similarities = []
    for community_id, data in temp_vdb.items():
        similarity = cosine_similarity(query_embedding[0], data["embedding"])
        similarities.append((community_id, similarity))

    # Sort by similarity
    similarities.sort(key=lambda x: x[1], reverse=True)

    # Step 5: Retrieve entities from top communities
    top_communities = [cid for cid, _ in similarities[:5]]  # Top-5 communities

    relevant_entities = []
    for community_id in top_communities:
        community_entities = [
            node for node, cid in communities.items() if cid == community_id
        ]
        relevant_entities.extend(community_entities)

    # Step 6: Return entity information
    entity_info = await graph.get_nodes(relevant_entities)

    return entity_info
```

**Use Case**: Large graphs where communities represent **thematic clusters** (e.g., "AI entities", "Finance entities").

---

## Связь с Graph Algorithms

Community detection uses graph topology:

**Файл**: spec/graph/03-centrality-algorithms.md

```python
# Community detection + Centrality
communities = community_louvain.best_partition(graph)

# Find important nodes in each community
for community_id in set(communities.values()):
    community_nodes = [n for n, c in communities.items() if c == community_id]

    # Subgraph for this community
    subgraph = graph.subgraph(community_nodes)

    # Degree centrality within community
    centrality = nx.degree_centrality(subgraph)

    # Most important node in community
    hub = max(centrality, key=centrality.get)

    print(f"Community {community_id}: Hub = {hub}")
```

**Interaction**: Communities provide **clustering**, centrality identifies **hubs** within clusters.

---

## Training Paradigm: Unsupervised Optimization

### No Training Data Required

Unlike supervised ML algorithms, community detection is **fully unsupervised**:

```
Input: Graph G = (V, E)
Output: Community assignments {node: community_id}

No labels needed — algorithm optimizes modularity Q automatically.
```

### Optimization Objective

```
Maximize: Q = Σ[actual_edges_within - expected_edges_within]

Greedy algorithm:
1. Start with Q = 0 (each node = own community)
2. Iteratively move nodes to maximize ΔQ
3. Stop when no move increases Q

Complexity:
- Best case: O(n log n)
- Worst case: O(n²)
- Typical: O(n log² n) for sparse graphs
```

### Hyperparameters

Louvain algorithm has **no hyperparameters** — fully automatic.

Optional variations:
- **Resolution parameter γ**: Controls community size
  ```python
  communities = community_louvain.best_partition(graph, resolution=γ)

  γ > 1: More smaller communities
  γ < 1: Fewer larger communities
  γ = 1: Default (standard modularity)
  ```

- **Randomness**: Algorithm has stochastic elements (node order)
  ```python
  # Run multiple times and pick best modularity
  best_communities = None
  best_q = -1

  for _ in range(10):
      communities = community_louvain.best_partition(graph)
      q = community_louvain.modularity(communities, graph)

      if q > best_q:
          best_q = q
          best_communities = communities
  ```

---

## Performance Characteristics

### Scalability

| Graph Size | Nodes | Edges | Time (Louvain) | Memory |
|-----------|-------|-------|----------------|--------|
| **Small** | 1K | 5K | 10ms | 1 MB |
| **Medium** | 10K | 50K | 100ms | 10 MB |
| **Large** | 100K | 500K | 1s | 100 MB |
| **Very Large** | 1M | 5M | 10s | 1 GB |

**Complexity**: O(n log² n) for sparse graphs, O(n²) for dense graphs.

### Quality Metrics

**Modularity (Q)**:
```
Real-world graphs:
- Social networks: Q = 0.3-0.7 (strong communities)
- Citation networks: Q = 0.4-0.6
- Random graphs: Q ≈ 0 (no community structure)

LightRAG entity graphs:
- Typical Q = 0.4-0.5 (moderate-strong communities)
```

**Number of Communities**:
```
Heuristic: C ≈ √N (where N = nodes)

Examples:
- 1K nodes → ~30 communities
- 10K nodes → ~100 communities
- 100K nodes → ~300 communities
```

---

## Use Cases in LightRAG

### 1. Graph Visualization

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py`

```python
# Color nodes by community
communities = community_louvain.best_partition(graph)

colors = []
for node in graph.nodes():
    community_id = communities[node]
    colors.append(community_id)

nx.draw(graph, node_color=colors, cmap=plt.cm.rainbow)
```

**Benefit**: Visual clustering helps understand graph structure.

### 2. Global Query Context

```python
# Retrieve entire communities (instead of individual entities)
query = "machine learning algorithms"

# Find relevant communities
community_summaries = await generate_community_summaries(graph)
relevant_communities = await search_communities(query, community_summaries)

# Return all entities in relevant communities
context_entities = []
for community_id in relevant_communities:
    context_entities.extend(get_community_entities(community_id))
```

**Benefit**: Broader context retrieval (entire thematic clusters).

### 3. Entity Deduplication

```python
# Entities in same community = likely duplicates
communities = community_louvain.best_partition(entity_graph)

for community_id in set(communities.values()):
    community_entities = [n for n, c in communities.items() if c == community_id]

    # Check for similar names within community
    for i, entity1 in enumerate(community_entities):
        for entity2 in community_entities[i+1:]:
            similarity = string_similarity(entity1, entity2)

            if similarity > 0.9:
                # Merge duplicates
                await graph.amerge_entities([entity2], entity1)
```

**Benefit**: Community structure guides deduplication (search within clusters, not entire graph).

---

## Alternative Algorithms

### 1. Label Propagation

```python
import networkx as nx

communities = nx.algorithms.community.label_propagation_communities(graph)

# Convert to dict format
communities_dict = {}
for community_id, nodes in enumerate(communities):
    for node in nodes:
        communities_dict[node] = community_id
```

**Comparison**:
- **Louvain**: Better quality (optimizes modularity), slower
- **Label Propagation**: Faster, simpler, lower quality

**Use case**: Label propagation for very large graphs (millions of nodes).

### 2. Girvan-Newman

```python
from networkx.algorithms.community import girvan_newman

# Hierarchical community detection
communities_generator = girvan_newman(graph)

# Get first level (2 communities)
top_level_communities = next(communities_generator)

# Get second level (4 communities)
second_level_communities = next(communities_generator)
```

**Comparison**:
- **Louvain**: Flat clustering, O(n log² n)
- **Girvan-Newman**: Hierarchical clustering, O(n³) (very slow)

**Use case**: Girvan-Newman for small graphs needing hierarchy.

### 3. Spectral Clustering

```python
from sklearn.cluster import SpectralClustering

# Convert graph to adjacency matrix
adjacency = nx.to_numpy_array(graph)

# Spectral clustering
clustering = SpectralClustering(n_clusters=10, affinity="precomputed")
labels = clustering.fit_predict(adjacency)

communities_dict = {node: label for node, label in zip(graph.nodes(), labels)}
```

**Comparison**:
- **Louvain**: Automatically determines number of communities
- **Spectral Clustering**: Requires predefined number of clusters

**Use case**: Spectral clustering when number of communities known a priori.

---

## Связь с Research Concepts

### Star-Attractor Pattern

**Файл**: spec/research/02-star-attractor-pattern.md

```
Communities as "Semantic Galaxy Clusters":

Community 1 (Tech):          Community 2 (Finance):
    Apple ★                      Goldman Sachs ★
   /  |  \                       /    |    \
 iPhone iPad Mac              Trading Bonds Equity

Each community = cluster of related attractors
Inter-community edges = weak semantic connections
```

**Interaction**: Community detection reveals **semantic neighborhoods** in star-attractor graph.

### Dual-Space Architecture

**Файл**: spec/research/03-dual-space-architecture.md

```
Graph Space (Discrete):         Vector Space (Continuous):
Communities = clusters          Embeddings cluster in ℝⁿ
      ↓                                  ↓
   [A, B, C]                    [E(A), E(B), E(C)] close in cosine space

Correspondence: Graph communities ≈ Vector clusters
```

**Interaction**: Community detection in graph space complements k-means clustering in vector space.

---

## Example: Full Community Detection Pipeline

```python
import networkx as nx
import community as community_louvain

# Load entity graph
entity_graph = nx.Graph()

# Add entities and relations
entities = ["Apple Inc", "iPhone", "iPad", "Samsung", "Galaxy", "Google", "Android"]
relations = [
    ("Apple Inc", "iPhone"),
    ("Apple Inc", "iPad"),
    ("Samsung", "Galaxy"),
    ("Google", "Android"),
    ("iPhone", "iPad"),  # Similar products
    ("Galaxy", "Android"),  # Related
]

entity_graph.add_edges_from(relations)

# Step 1: Detect communities
communities = community_louvain.best_partition(entity_graph)

print("Communities:")
for node, community_id in communities.items():
    print(f"  {node}: Community {community_id}")

# Output:
# Communities:
#   Apple Inc: Community 0
#   iPhone: Community 0
#   iPad: Community 0
#   Samsung: Community 1
#   Galaxy: Community 1
#   Google: Community 1
#   Android: Community 1

# Step 2: Compute modularity
modularity = community_louvain.modularity(communities, entity_graph)
print(f"\nModularity Q = {modularity:.3f}")

# Output:
# Modularity Q = 0.306 (moderate community structure)

# Step 3: Visualize
import matplotlib.pyplot as plt

pos = nx.spring_layout(entity_graph)
colors = [communities[node] for node in entity_graph.nodes()]

nx.draw(
    entity_graph,
    pos,
    node_color=colors,
    cmap=plt.cm.rainbow,
    with_labels=True,
    node_size=500
)
plt.title("Entity Graph with Communities")
plt.show()
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
