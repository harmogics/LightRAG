# Graph Topology: Топология и Структура Графа

## Обзор

Knowledge Graph в LightRAG обладает специфичной топологической структурой, определяющей эффективность поиска и reasoning. Понимание топологии критично для оптимизации query performance и design эффективных search strategies.

## Graph Properties

### Scale-Free Network

**Характеристика**: Распределение степеней узлов следует power law

```
P(k) ∝ k^(-γ)

где:
- P(k) = вероятность что узел имеет k связей
- γ ≈ 2-3 (типичное значение для knowledge graphs)
```

**Визуализация**:

```
Degree Distribution (Log-Log Scale)

log P(k) │
         │ ●
         │  ●
         │   ●
         │    ●●
         │      ●●
         │        ●●●
         │           ●●●●
         │               ●●●●●●
         └──────────────────────── log k
```

**Implications**:
- ✅ Небольшое число hub nodes (highly connected)
- ✅ Большинство nodes имеют few connections
- ✅ Robust к random node failures
- ⚠️ Vulnerable к targeted removal of hubs

**Hub Examples в LightRAG**:
- **Apple Inc**: связано с products, people, locations, events
- **United States**: связано с companies, people, locations
- **Technology**: связано с concepts, products, organizations

---

### Small-World Property

**Характеристика**: Short path length между любыми узлами

```
Average Path Length (L):
L ≈ log(N) / log(⟨k⟩)

где:
- N = количество nodes
- ⟨k⟩ = средняя степень узла

Typical: L = 3-6 hops для большинства пар узлов
```

**Six Degrees of Separation** в Knowledge Graphs:

```
Tim Cook ─[2 hops]→ Stanford University

Path:
Tim Cook → Apple Inc → Stanford University
           (works_for)  (recruits_from)
```

**Implications**:
- ✅ Efficient information propagation
- ✅ Fast path finding между entities
- ✅ Multi-hop reasoning остается feasible

---

### High Clustering Coefficient

**Характеристика**: Neighbors узла склонны быть связаны между собой

```
Clustering Coefficient (C):
C = (# triangles connected to node) / (# possible triangles)

Global Clustering:
C_global = Σ C_i / N

Typical: C_global ≈ 0.3-0.6
```

**Triangle Example**:

```
    Tim Cook
       / \
      /   \
     /     \
Apple Inc - iPhone
```

Все три entities связаны → high local clustering

**Implications**:
- ✅ Strong community structure
- ✅ Entities в одном context связаны
- ✅ Good для community detection

---

## Centrality Metrics

### 1. Degree Centrality

**Определение**: Количество прямых connections узла

```python
degree_centrality(v) = degree(v) / (N - 1)

где degree(v) = количество ребер узла v
```

**Computation**:

```python
import networkx as nx

def compute_degree_centrality(graph):
    """Compute degree centrality for all nodes"""
    return nx.degree_centrality(graph)

# Example output:
# {
#     "Apple Inc": 0.15,      # 15% of possible connections
#     "Tim Cook": 0.08,
#     "iPhone": 0.12,
#     ...
# }
```

**Use Cases**:
- Идентификация hub entities
- Ranking по "influence"
- Priority scoring в search

**Top Entities by Degree**:
```
1. Apple Inc (degree: 150)
2. United States (degree: 120)
3. Technology (degree: 95)
4. Machine Learning (degree: 80)
```

---

### 2. PageRank

**Определение**: Итеративный алгоритм для оценки importance

```
PageRank Formula:
PR(v) = (1-d)/N + d × Σ PR(u)/L(u)
                    u∈In(v)

где:
- d = damping factor (обычно 0.85)
- In(v) = incoming neighbors
- L(u) = out-degree узла u
```

**Computation**:

```python
def compute_pagerank(graph, damping=0.85, max_iter=100):
    """Compute PageRank for all nodes"""
    return nx.pagerank(graph, alpha=damping, max_iter=max_iter)

# Example output:
# {
#     "Apple Inc": 0.0145,
#     "Tim Cook": 0.0089,
#     "iPhone": 0.0112,
#     ...
# }
```

**Personalized PageRank**:

```python
def personalized_pagerank(graph, source_entities, damping=0.85):
    """
    PageRank biased towards source entities

    Use case: Ranking entities relevant to query
    """
    personalization = {
        entity: 1.0 if entity in source_entities else 0.0
        for entity in graph.nodes()
    }

    return nx.pagerank(
        graph,
        alpha=damping,
        personalization=personalization
    )
```

**Use Cases**:
- **Global Search**: Ranking entities by importance
- **Query Biasing**: Personalized PageRank для query entities
- **Recommendation**: Suggest related entities

---

### 3. Betweenness Centrality

**Определение**: Частота нахождения узла на shortest paths

```
Betweenness(v) = Σ σ(s,t|v) / σ(s,t)
                s≠v≠t

где:
- σ(s,t) = количество shortest paths от s к t
- σ(s,t|v) = количество путей проходящих через v
```

**Computation**:

```python
def compute_betweenness(graph):
    """Compute betweenness centrality"""
    return nx.betweenness_centrality(graph)

# Example:
# {
#     "Apple Inc": 0.28,      # Bridge между многими entities
#     "Technology": 0.35,     # Высокий betweenness (connector)
#     "iPhone": 0.12
# }
```

**Use Cases**:
- Идентификация "bridge" entities
- Finding connectors между communities
- Critical nodes для information flow

**Bridge Example**:

```
[Tech Products] ←→ Apple Inc ←→ [People]
                       ↕
                  [Locations]
```

Apple Inc acts as bridge между разными типами entities

---

### 4. Closeness Centrality

**Определение**: Average distance к другим узлам

```
Closeness(v) = (N-1) / Σ d(v,u)
                       u≠v

где d(v,u) = shortest path distance
```

**Computation**:

```python
def compute_closeness(graph):
    """Compute closeness centrality"""
    return nx.closeness_centrality(graph)

# Example:
# {
#     "Apple Inc": 0.42,      # Close to many entities
#     "iPhone": 0.38,
#     "Remote Entity": 0.15   # Far from others
# }
```

**Use Cases**:
- Measuring "accessibility" of entity
- Finding central concepts
- Optimizing starting points для traversal

---

## Path Analysis

### Average Path Length

**Computation**:

```python
def compute_avg_path_length(graph):
    """
    Average shortest path length в графе

    Typical range: 3-6 hops
    """
    if not nx.is_connected(graph):
        # Для несвязного графа - largest component
        largest_cc = max(nx.connected_components(graph), key=len)
        subgraph = graph.subgraph(largest_cc)
    else:
        subgraph = graph

    return nx.average_shortest_path_length(subgraph)
```

**Example Results**:

```
Graph Statistics:
- Nodes: 10,000 entities
- Edges: 50,000 relationships
- Average Path Length: 4.2 hops
- Diameter: 8 hops
```

**Implications**:
- Multi-hop reasoning within 4-5 hops covers most entities
- Global mode может эффективно explore граф
- Path-based explanations остаются comprehensible

---

### Diameter

**Определение**: Maximum distance между любыми двумя узлами

```python
def compute_diameter(graph):
    """Maximum shortest path length"""
    if not nx.is_connected(graph):
        # Для несвязного графа - largest component
        largest_cc = max(nx.connected_components(graph), key=len)
        subgraph = graph.subgraph(largest_cc)
    else:
        subgraph = graph

    return nx.diameter(subgraph)

# Typical: 6-10 hops
```

**Use Cases**:
- Setting max_depth для graph traversal
- Understanding graph "size"
- Optimization boundaries

---

### Path Distribution

```python
def analyze_path_distribution(graph, sample_size=1000):
    """
    Analyze distribution of path lengths

    Returns histogram of path lengths
    """
    nodes = list(graph.nodes())
    paths = []

    for _ in range(sample_size):
        source, target = random.sample(nodes, 2)
        try:
            length = nx.shortest_path_length(graph, source, target)
            paths.append(length)
        except nx.NetworkXNoPath:
            pass

    return Counter(paths)
```

**Example Distribution**:

```
Path Length Histogram:

Count │
  300 │     ██████
  250 │   ████████████
  200 │ ██████████████████
  150 │████████████████████████
  100 │████████████████████████████
   50 │████████████████████████████████
    0 └──────────────────────────────────
      1   2   3   4   5   6   7   8   Length

Most paths: 3-5 hops
```

---

## Community Structure

### Community Detection Algorithms

#### 1. Louvain Method

**Принцип**: Modularity optimization

```python
import community as community_louvain

def detect_communities_louvain(graph):
    """
    Louvain community detection

    Fast, hierarchical, good quality
    """
    communities = community_louvain.best_partition(graph)

    # Group by community
    community_map = defaultdict(list)
    for node, comm_id in communities.items():
        community_map[comm_id].append(node)

    return dict(community_map)

# Example output:
# {
#     0: ["Apple Inc", "iPhone", "Tim Cook", "Mac", ...],  # Tech Products
#     1: ["Microsoft", "Windows", "Azure", ...],            # Microsoft Ecosystem
#     2: ["Machine Learning", "AI", "Neural Networks", ...] # AI/ML Concepts
# }
```

**Modularity Score**:

```python
def compute_modularity(graph, communities):
    """
    Measure quality of community detection

    Range: -0.5 to 1.0
    Good: > 0.3
    """
    return community_louvain.modularity(communities, graph)
```

---

#### 2. Label Propagation

**Принцип**: Nodes adopt labels from neighbors

```python
def detect_communities_label_propagation(graph):
    """
    Fast algorithm for large graphs

    Less stable but very fast
    """
    communities_generator = nx.algorithms.community.label_propagation_communities(
        graph
    )

    return {
        i: list(community)
        for i, community in enumerate(communities_generator)
    }
```

---

#### 3. Girvan-Newman (Hierarchical)

**Принцип**: Edge betweenness-based removal

```python
def detect_communities_girvan_newman(graph, num_communities=5):
    """
    Hierarchical community detection

    Slower but produces dendrogram
    """
    communities_generator = nx.algorithms.community.girvan_newman(graph)

    # Get specific number of communities
    for _ in range(num_communities - 1):
        communities = next(communities_generator)

    return {
        i: list(community)
        for i, community in enumerate(communities)
    }
```

---

### Community Characteristics

**Size Distribution**:

```
Community Size Histogram:

Count │
   15 │ ████████
   10 │ ████████████████
    5 │ ████████████████████████
    0 └────────────────────────────
      10  50  100 150 200  Size

Most communities: 20-50 entities
```

**Intra- vs Inter-community Edges**:

```python
def analyze_community_edges(graph, communities):
    """
    Compare edges within vs between communities
    """
    intra_edges = 0
    inter_edges = 0

    for u, v in graph.edges():
        u_comm = get_community(u, communities)
        v_comm = get_community(v, communities)

        if u_comm == v_comm:
            intra_edges += 1
        else:
            inter_edges += 1

    return {
        "intra": intra_edges,
        "inter": inter_edges,
        "ratio": intra_edges / inter_edges
    }

# Typical: ratio = 4-8 (much more intra-community edges)
```

---

## Subgraph Patterns

### Pattern 1: Star (Hub-Spoke)

**Structure**:

```
                 Entity 2
                    │
    Entity 1 ───── HUB ───── Entity 3
                    │
                 Entity 4
```

**Detection**:

```python
def detect_star_patterns(graph, min_degree=10):
    """
    Detect hub entities (star centers)
    """
    stars = []

    for node in graph.nodes():
        degree = graph.degree(node)
        if degree >= min_degree:
            neighbors = list(graph.neighbors(node))
            stars.append({
                "hub": node,
                "spokes": neighbors,
                "degree": degree
            })

    return sorted(stars, key=lambda x: x["degree"], reverse=True)
```

**Example**: Apple Inc (hub) → {iPhone, iPad, Mac, Tim Cook, ...}

**Search Optimization**:
- Direct neighbor retrieval очень быстрый
- Good для "What does X have?" queries

---

### Pattern 2: Chain (Sequential)

**Structure**:

```
Entity 1 ──→ Entity 2 ──→ Entity 3 ──→ Entity 4
```

**Detection**:

```python
def detect_chain_patterns(graph, min_length=3):
    """
    Detect linear chains of entities
    """
    chains = []

    for node in graph.nodes():
        if graph.in_degree(node) == 1 and graph.out_degree(node) == 1:
            # Potential chain member
            chain = build_chain(graph, node)
            if len(chain) >= min_length:
                chains.append(chain)

    return chains
```

**Example**: Steve Jobs → Apple Inc → iPhone → iOS

**Search Optimization**:
- Path traversal для causal/temporal reasoning
- Good для "How did X lead to Y?" queries

---

### Pattern 3: Clique (Fully Connected)

**Structure**:

```
    A ──── B
    │ \  / │
    │  \/  │
    │  /\  │
    │ /  \ │
    C ──── D
```

**Detection**:

```python
def detect_cliques(graph, min_size=3):
    """
    Find maximal cliques (fully connected subgraphs)
    """
    cliques = list(nx.find_cliques(graph))

    # Filter by size
    return [
        clique for clique in cliques
        if len(clique) >= min_size
    ]
```

**Example**: {Tim Cook, Apple Inc, Cupertino} - все связаны

**Search Optimization**:
- Strong communities
- Good для "What is strongly related to X?"

---

### Pattern 4: Bipartite (Two-Mode)

**Structure**:

```
  [Group A]          [Group B]
   Person 1 ────────── Org 1
   Person 2 ────────── Org 2
   Person 3 ────────── Org 1
```

**Detection**:

```python
def detect_bipartite_patterns(graph):
    """
    Detect bipartite subgraphs (e.g., Person-Organization)
    """
    # Find Person and Organization nodes
    persons = [n for n in graph.nodes() if graph.nodes[n].get('type') == 'person']
    orgs = [n for n in graph.nodes() if graph.nodes[n].get('type') == 'organization']

    # Check if edges only between groups
    subgraph = graph.subgraph(persons + orgs)

    if nx.is_bipartite(subgraph):
        return {
            "set_a": persons,
            "set_b": orgs
        }

    return None
```

**Example**: People ↔ Organizations they work for

**Search Optimization**:
- Projection для entity type specific queries
- Good для "Who works where?" queries

---

## Graph Evolution Over Time

### Growth Patterns

```python
def analyze_graph_growth(snapshots):
    """
    Analyze how graph grows over time

    snapshots: list of graphs at different timestamps
    """
    metrics = []

    for snapshot in snapshots:
        metrics.append({
            "timestamp": snapshot.timestamp,
            "nodes": snapshot.number_of_nodes(),
            "edges": snapshot.number_of_edges(),
            "avg_degree": sum(dict(snapshot.degree()).values()) / snapshot.number_of_nodes(),
            "diameter": nx.diameter(snapshot) if nx.is_connected(snapshot) else None,
            "communities": len(detect_communities_louvain(snapshot))
        })

    return metrics
```

**Typical Growth**:

```
Nodes Over Time:

Count │                          ╱
10000 │                      ╱
 8000 │                  ╱
 6000 │              ╱
 4000 │          ╱
 2000 │      ╱
    0 └──────────────────────────
      0   10  20  30  40  50  Days

Linear to sublinear growth
```

### Preferential Attachment

**Principle**: New entities preferentially connect to hubs

```
P(new_edge → v) ∝ degree(v)

"Rich get richer" phenomenon
```

**Implications**:
- Hub entities grow connections faster
- Network becomes more scale-free over time
- Important для predicting future connections

---

## Optimization Strategies

### 1. Index Structures

```python
# Degree index for fast hub lookup
degree_index = {
    degree: [node for node in graph.nodes() if graph.degree(node) == degree]
    for degree in range(max_degree + 1)
}

# Type index for entity filtering
type_index = defaultdict(list)
for node in graph.nodes():
    node_type = graph.nodes[node].get('entity_type')
    type_index[node_type].append(node)
```

### 2. Precomputed Metrics

```python
# Precompute expensive metrics at build time
precomputed = {
    "pagerank": nx.pagerank(graph),
    "betweenness": nx.betweenness_centrality(graph),
    "communities": detect_communities_louvain(graph),
    "hubs": get_hub_nodes(graph, min_degree=20)
}

# Store in cache
await cache.set("graph_metrics", precomputed)
```

### 3. Subgraph Caching

```python
# Cache frequently accessed subgraphs
subgraph_cache = {}

def get_subgraph_cached(entity_names, depth=2):
    cache_key = f"{sorted(entity_names)}:{depth}"

    if cache_key in subgraph_cache:
        return subgraph_cache[cache_key]

    subgraph = extract_subgraph(entity_names, depth)
    subgraph_cache[cache_key] = subgraph

    return subgraph
```

---

## Visualization

### NetworkX Visualization

```python
import matplotlib.pyplot as plt

def visualize_subgraph(graph, entities, depth=2):
    """Visualize subgraph around entities"""
    subgraph = extract_subgraph(entities, depth)

    # Layout
    pos = nx.spring_layout(subgraph)

    # Node colors by type
    colors = [type_to_color(graph.nodes[n]['entity_type']) for n in subgraph.nodes()]

    # Node sizes by degree
    sizes = [graph.degree(n) * 100 for n in subgraph.nodes()]

    # Draw
    nx.draw(subgraph, pos,
            node_color=colors,
            node_size=sizes,
            with_labels=True,
            font_size=8)

    plt.show()
```

### Interactive Visualization (Plotly)

```python
import plotly.graph_objects as go

def interactive_graph_viz(graph):
    """Create interactive 3D graph visualization"""
    # ... implementation
```

---

**Следующий раздел**: [04-search-patterns.md](04-search-patterns.md) - Паттерны поиска по графу
