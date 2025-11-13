# Graph Layout Algorithms: Алгоритмы Визуализации Графа

## Концептуальная Парадигма

**Graph layout algorithms** преобразуют abstract knowledge graph в **visual spatial representation** для человеческого анализа.

```
Abstract Graph              Visual Layout
┌──────────────┐           ┌──────────────┐
│ Nodes: 1000  │           │              │
│ Edges: 5000  │   ──→     │  ● ───── ●   │
│ (topology)   │           │   ╲     ╱    │
└──────────────┘           │    ● ●      │
                           │ (2D/3D space)│
                           └──────────────┘
```

**Философия**: Layout algorithms reveal **semantic structure** through spatial proximity — related entities cluster together.

---

## Применение в LightRAG

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py`

LightRAG visualizer использует layout algorithms для интерактивной 3D визуализации knowledge graph:

- **Spring Layout**: Force-directed (default)
- **Circular Layout**: Entities в круговом расположении
- **Shell Layout**: Community-based concentric shells
- **Random Layout**: Baseline для сравнения

---

## Алгоритм 1: Spring Layout (Force-Directed)

### Описание

**Spring Layout** (aka Fruchterman-Reingold) моделирует graph как физическую систему:
- **Nodes** = charged particles (repel each other)
- **Edges** = springs (attract connected nodes)
- **Layout** = equilibrium state (energy minimization)

```
Initial (random):        After iterations:
  ●    ●                      ●───●
        ●    ●           ╱   │   ╲
    ●                   ●────●────●
       ●                     │
                            ●
```

**Интуиция**: Strongly connected nodes cluster together, weakly connected spread apart.

### Реализация

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py:213, 547`

```python
import networkx as nx
import numpy as np

def calculate_layout(self):
    """
    Calculate 3D spring layout for knowledge graph.

    Metaphor: Semantic attractors naturally cluster by relation density.
    """
    if not self.graph:
        return

    # Spring layout in 3D
    pos = nx.spring_layout(
        self.graph,
        dim=3,              # 3D space
        k=2.0,              # Spring constant (distance between nodes)
        iterations=100,     # Optimization iterations
        weight=None,        # Edge weights (optional)
        seed=42             # Reproducibility
    )

    # pos = {node_id: [x, y, z], ...}

    # Scale positions to viewport
    positions = np.array(list(pos.values()))
    scale = 10.0 / max(1.0, np.max(np.abs(positions)))
    pos = {node: coords * scale for node, coords in pos.items()}

    # Assign positions to node objects
    for node_id, position in pos.items():
        self.id_node_map[node_id].position = glm.vec3(position)

    self.update_buffers()

# Update layout (iterative refinement)
def update_layout(self):
    """
    Refine existing layout (warm start from current positions).
    """
    pos = nx.spring_layout(
        self.graph,
        dim=3,
        pos={
            node_id: list(node.position)
            for node_id, node in self.id_node_map.items()
        },  # Warm start from current positions
        k=2.0,
        iterations=100,  # Additional refinement iterations
        weight=None,
    )

    # Update positions
    for node_id, position in pos.items():
        self.id_node_map[node_id].position = glm.vec3(position)

    self.update_buffers()
```

### Algorithm (High-Level)

**Fruchterman-Reingold Force Model**:

```python
# Repulsive force (nodes repel)
f_repulsive(u, v) = k^2 / distance(u, v)

# Attractive force (edges attract)
f_attractive(u, v) = distance(u, v)^2 / k

# Net force on node u
force(u) = Σ f_attractive(u, v) for neighbors v
         - Σ f_repulsive(u, w) for all nodes w

# Update positions (gradient descent)
for iteration in range(max_iter):
    for node in nodes:
        force_vector = compute_forces(node)
        node.position += force_vector * cooling_factor

    cooling_factor *= 0.99  # Simulated annealing
```

### Parameters

```python
k: float = 2.0           # Optimal distance between nodes
iterations: int = 100    # Number of refinement iterations
dim: int = 3             # Dimensions (2D or 3D)
weight: str = None       # Edge attribute for spring strength
seed: int = None         # Random seed for reproducibility
```

### Complexity

- **Time**: O(V^2) per iteration (naive), O(V log V) with Barnes-Hut optimization
- **Space**: O(V)
- **Total**: O(iterations * V^2) ≈ O(100 * V^2)

### Use Case: Default Visualization

```python
# Load knowledge graph
graph = nx.read_graphml("knowledge_graph.graphml")

# Calculate spring layout
pos = nx.spring_layout(graph, dim=3, iterations=100)

# Render 3D visualization
for node, (x, y, z) in pos.items():
    render_sphere(position=(x, y, z), label=node)

for (u, v) in graph.edges():
    render_line(pos[u], pos[v])
```

**Result**: Related entities (high edge density) cluster spatially → visual semantic clusters.

### Связь с Star-Attractor Pattern

Spring layout reveals **star patterns** naturally (spec/research/02-star-attractor-pattern.md):

```
         ●  (Tim Cook)
        ╱
   ●───●───●  (Apple Inc = central hub)
        ╲
         ●  (iPhone)

Hub entities (high degree) → center of cluster
Peripheral entities → outskirts
```

---

## Алгоритм 2: Circular Layout

### Описание

**Circular layout** размещает entities равномерно по окружности.

```
        ●
    ●       ●
  ●           ●
    ●       ●
        ●
```

**Интуиция**: Simple symmetric layout, good for small graphs (< 50 nodes).

### Реализация

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py:551`

```python
import networkx as nx

def calculate_circular_layout(self):
    """
    Circular layout: entities arranged in circle.
    """
    # NetworkX circular layout (2D)
    pos_2d = nx.circular_layout(self.graph)
    # pos_2d = {node_id: [x, y], ...}

    # Convert to 3D (set z=0)
    pos = {
        node: np.array([x, 0.0, y])  # (x, 0, y)
        for node, (x, y) in pos_2d.items()
    }

    # Scale and assign positions
    positions = np.array(list(pos.values()))
    scale = 10.0 / max(1.0, np.max(np.abs(positions)))
    pos = {node: coords * scale for node, coords in pos.items()}

    for node_id, position in pos.items():
        self.id_node_map[node_id].position = glm.vec3(position)
```

### Complexity

- **Time**: O(V) (evenly space nodes on circle)
- **Space**: O(V)

### Parameters

```python
scale: float = 1.0       # Circle radius
center: tuple = (0, 0)   # Circle center
```

### Use Case

```python
# Small graph overview
if graph.number_of_nodes() < 50:
    pos = nx.circular_layout(graph, scale=10)
else:
    pos = nx.spring_layout(graph)  # Use spring for large graphs
```

---

## Алгоритм 3: Shell Layout (Community-Based)

### Описание

**Shell layout** размещает entities в **concentric shells** based on communities.

```
        Outer Shell (Community 3)
    ●       ●       ●
  ╱     Middle Shell (Community 2)   ╲
 ●       ●       ●       ●       ●
  ╲      Inner Shell (Community 1)   ╱
    ●       ●       ●
        Center
```

**Интуиция**: Community-based visualization — same community = same shell.

### Реализация

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py:554`

```python
import networkx as nx
import community  # python-louvain

def calculate_shell_layout(self):
    """
    Shell layout: communities in concentric shells.
    """
    # Detect communities
    self.communities = community.best_partition(self.graph)
    num_communities = len(set(self.communities.values()))

    # Group nodes by community
    comm_lists = [[] for _ in range(num_communities)]
    for node, comm_id in self.communities.items():
        comm_lists[comm_id].append(node)

    # Shell layout (2D)
    pos_2d = nx.shell_layout(
        self.graph,
        nlist=comm_lists  # Each community = one shell
    )

    # Convert to 3D (z=0)
    pos = {
        node: np.array([x, 0.0, y])
        for node, (x, y) in pos_2d.items()
    }

    # Scale and assign
    positions = np.array(list(pos.values()))
    scale = 10.0 / max(1.0, np.max(np.abs(positions)))
    pos = {node: coords * scale for node, coords in pos.items()}

    for node_id, position in pos.items():
        self.id_node_map[node_id].position = glm.vec3(position)
```

### Complexity

- **Time**: O(V) + O(community_detection)
- **Space**: O(V)

### Use Case: Community Visualization

```python
# Detect communities
communities = community.best_partition(graph)
num_comms = len(set(communities.values()))

# Group by community
comm_groups = [[] for _ in range(num_comms)]
for node, comm_id in communities.items():
    comm_groups[comm_id].append(node)

# Shell layout
pos = nx.shell_layout(graph, nlist=comm_groups)

# Result: Visual separation of semantic clusters
```

### Связь с Community Detection

Shell layout visualizes **community structure** (spec/graph/02-community-detection.md):

```
Shell 1 (inner): Community 0 (Technology)
  [Apple, Samsung, Microsoft, ...]

Shell 2 (middle): Community 1 (People)
  [Tim Cook, Steve Jobs, Elon Musk, ...]

Shell 3 (outer): Community 2 (Locations)
  [California, Silicon Valley, ...]
```

---

## Алгоритм 4: Random Layout

### Описание

**Random layout** — baseline для сравнения, entities randomly placed.

### Реализация

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py:561`

```python
import numpy as np

def calculate_random_layout(self):
    """
    Random layout: nodes placed randomly in 3D space.
    """
    pos = {
        node: np.random.rand(3) * 2 - 1  # Random in [-1, 1]^3
        for node in self.graph.nodes()
    }

    # Scale
    positions = np.array(list(pos.values()))
    scale = 10.0 / max(1.0, np.max(np.abs(positions)))
    pos = {node: coords * scale for node, coords in pos.items()}

    for node_id, position in pos.items():
        self.id_node_map[node_id].position = glm.vec3(position)
```

### Complexity

- **Time**: O(V)
- **Space**: O(V)

### Use Case

Baseline для оценки quality of other layouts:

```python
# Compare layout quality
random_pos = nx.random_layout(graph, dim=3)
spring_pos = nx.spring_layout(graph, dim=3)

# Measure edge crossings (fewer = better)
random_crossings = count_edge_crossings(random_pos, graph.edges())
spring_crossings = count_edge_crossings(spring_pos, graph.edges())

# Spring should have fewer crossings
assert spring_crossings < random_crossings
```

---

## Layout Quality Metrics

### 1. Edge Length Variance

```python
def edge_length_variance(pos, edges):
    """
    Measure variance in edge lengths.
    Low variance = uniform spacing (good).
    """
    lengths = []
    for (u, v) in edges:
        length = np.linalg.norm(pos[u] - pos[v])
        lengths.append(length)

    return np.var(lengths)

# Lower variance = better layout
```

### 2. Node Overlap

```python
def count_node_overlaps(pos, min_distance=0.5):
    """
    Count pairs of nodes too close together.
    No overlaps = clear visualization.
    """
    overlaps = 0
    nodes = list(pos.keys())

    for i, u in enumerate(nodes):
        for v in nodes[i+1:]:
            distance = np.linalg.norm(pos[u] - pos[v])
            if distance < min_distance:
                overlaps += 1

    return overlaps

# Zero overlaps = good layout
```

### 3. Community Separation

```python
def community_separation(pos, communities):
    """
    Measure spatial separation between communities.
    High separation = clear community structure.
    """
    # Calculate community centroids
    centroids = {}
    for comm_id in set(communities.values()):
        comm_nodes = [n for n, c in communities.items() if c == comm_id]
        centroid = np.mean([pos[n] for n in comm_nodes], axis=0)
        centroids[comm_id] = centroid

    # Measure inter-community distances
    dists = []
    comm_ids = list(centroids.keys())
    for i, c1 in enumerate(comm_ids):
        for c2 in comm_ids[i+1:]:
            dist = np.linalg.norm(centroids[c1] - centroids[c2])
            dists.append(dist)

    return np.mean(dists)  # Higher = better separation
```

---

## Связь с Dependencies

### NetworkX (spec/dependencies/03-graph-processing.md)

```python
import networkx as nx

# All layout algorithms via NetworkX
pos_spring = nx.spring_layout(graph, dim=3, iterations=100)
pos_circular = nx.circular_layout(graph)
pos_shell = nx.shell_layout(graph, nlist=community_groups)
pos_random = nx.random_layout(graph, dim=3)

# Custom layouts (not in NetworkX)
# Hierarchical layout for tree structures
pos_hierarchy = nx.kamada_kawai_layout(graph)

# Spectral layout (eigenvalue-based)
pos_spectral = nx.spectral_layout(graph, dim=3)
```

### 3D Visualization (spec/dependencies/03-graph-processing.md)

LightRAG visualizer uses:
- **ModernGL**: OpenGL rendering
- **imgui_bundle**: UI controls
- **NetworkX**: Layout computation

```python
# Compute layout
pos = nx.spring_layout(graph, dim=3)

# Render with OpenGL
for node, (x, y, z) in pos.items():
    render_sphere(position=(x, y, z), color=node_color)

for (u, v) in graph.edges():
    render_line(pos[u], pos[v], color=edge_color)
```

---

## Interactive Visualization Features

**Файл**: `lightrag/tools/lightrag_visualizer/graph_visualizer.py`

### 1. Layout Switching

```python
# Runtime layout switching
if imgui.combo("Layout", current_layout, available_layouts):
    if selected_layout == "Spring":
        self.calculate_spring_layout()
    elif selected_layout == "Circular":
        self.calculate_circular_layout()
    elif selected_layout == "Shell":
        self.calculate_shell_layout()
    # ...
    self.update_buffers()
```

### 2. Iterative Refinement

```python
# Button: "Update Layout" (refine current positions)
if imgui.button("Update Layout"):
    self.update_layout()  # Warm start from current pos

# Runs 100 more spring iterations → smoother layout
```

### 3. Community-Based Coloring

```python
# Detect communities
communities = community.best_partition(graph)

# Assign colors by community
for node, comm_id in communities.items():
    node.color = community_colors[comm_id]

# Render: same community = same color
# → Visual clustering in 3D space
```

---

## Производительность

### Benchmark (Graph: 1K nodes, 5K edges)

| Layout | Computation Time | Quality | Use Case |
|--------|------------------|---------|----------|
| **Spring (100 iter)** | 2.5s | High | Default (< 10K nodes) |
| **Spring (50 iter)** | 1.2s | Medium | Fast preview |
| **Circular** | 10ms | Low | Small graphs |
| **Shell** | 150ms | Medium | Community viz |
| **Random** | 5ms | Very Low | Baseline |

### Large Graphs (10K+ nodes)

For large graphs, spring layout becomes slow (O(V^2)):

```python
# Optimization: Multi-level layout
if graph.number_of_nodes() > 10000:
    # 1. Detect communities
    communities = community.best_partition(graph)

    # 2. Coarse layout: position community centroids
    coarse_graph = build_community_graph(graph, communities)
    coarse_pos = nx.spring_layout(coarse_graph, iterations=50)

    # 3. Fine layout: position nodes within communities
    pos = {}
    for comm_id in set(communities.values()):
        comm_nodes = [n for n, c in communities.items() if c == comm_id]
        subgraph = graph.subgraph(comm_nodes)

        # Layout subgraph around community centroid
        sub_pos = nx.spring_layout(subgraph, iterations=20, scale=2.0)

        # Offset by community centroid
        centroid = coarse_pos[comm_id]
        for node, pos_local in sub_pos.items():
            pos[node] = pos_local + centroid

    # Result: Fast hierarchical layout
```

---

## Example: Complete Visualization Pipeline

```python
async def visualize_knowledge_graph(
    graph_file: str,
    layout_type: str = "spring",
    show_communities: bool = True
):
    """
    Complete pipeline: Load → Layout → Visualize.
    """
    import networkx as nx
    import community

    # 1. Load graph
    graph = nx.read_graphml(graph_file)
    print(f"Loaded: {graph.number_of_nodes()} nodes, {graph.number_of_edges()} edges")

    # 2. Detect communities (for coloring)
    communities_dict = None
    if show_communities:
        communities_dict = community.best_partition(graph)
        num_comms = len(set(communities_dict.values()))
        print(f"Detected {num_comms} communities")

    # 3. Calculate layout
    if layout_type == "spring":
        pos = nx.spring_layout(graph, dim=3, iterations=100, seed=42)
    elif layout_type == "circular":
        pos_2d = nx.circular_layout(graph)
        pos = {n: np.array([x, 0, y]) for n, (x, y) in pos_2d.items()}
    elif layout_type == "shell" and communities_dict:
        comm_groups = group_by_community(communities_dict)
        pos_2d = nx.shell_layout(graph, nlist=comm_groups)
        pos = {n: np.array([x, 0, y]) for n, (x, y) in pos_2d.items()}
    else:
        pos = nx.random_layout(graph, dim=3)

    # 4. Scale to viewport
    positions = np.array(list(pos.values()))
    scale = 10.0 / max(1.0, np.max(np.abs(positions)))
    pos = {node: coords * scale for node, coords in pos.items()}

    # 5. Assign colors by community
    if communities_dict:
        colors = generate_community_colors(communities_dict)
    else:
        colors = {n: (0.5, 0.5, 0.5) for n in graph.nodes()}

    # 6. Render 3D visualization
    render_3d_graph(graph, pos, colors)

# Usage
await visualize_knowledge_graph(
    "lightrag_kg.graphml",
    layout_type="spring",
    show_communities=True
)
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
