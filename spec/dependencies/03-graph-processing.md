# Graph Processing: Обработка Графов Знаний

## Обзор

Библиотеки для работы с графовыми структурами данных. Эти зависимости обеспечивают **graph algorithms**, **traversal operations**, и **community detection** для knowledge graph processing.

## Core Library: networkx

**Версия**: Latest
**Лицензия**: BSD
**Сайт**: https://networkx.org

### Назначение

Comprehensive library для создания, манипуляции и изучения complex networks/graphs. В LightRAG это **default graph storage backend**.

### Использование в LightRAG

**Файл**: `lightrag/kg/networkx_impl.py`

```python
import networkx as nx

class NetworkXStorage(BaseGraphStorage):
    """
    NetworkX-based graph storage.

    Graph Type: nx.Graph (undirected) or nx.DiGraph (directed)
    Storage Format: GraphML (XML-based)
    """

    def __init__(self, namespace, global_config, ...):
        self.graph = nx.Graph()  # or load from file
        self.file_path = f"{working_dir}/{namespace}.graphml"

    async def upsert_node(self, node_id: str, node_data: dict):
        """Add/update node in graph"""
        self.graph.add_node(
            node_id,
            entity_type=node_data["entity_type"],
            description=node_data["description"],
            source_id=node_data["source_id"],
            ...
        )

    async def upsert_edge(self, src: str, tgt: str, edge_data: dict):
        """Add/update edge in graph"""
        self.graph.add_edge(
            src,
            tgt,
            description=edge_data["description"],
            keywords=edge_data["keywords"],
            weight=edge_data.get("weight", 1.0),
            ...
        )

    async def get_node(self, node_id: str) -> dict | None:
        """Retrieve node data"""
        if self.graph.has_node(node_id):
            return dict(self.graph.nodes[node_id])
        return None

    async def get_neighbors(self, node_id: str) -> list[str]:
        """Get all neighbors of a node"""
        if self.graph.has_node(node_id):
            return list(self.graph.neighbors(node_id))
        return []

    def save(self):
        """Persist graph to disk"""
        nx.write_graphml(self.graph, self.file_path)

    @staticmethod
    def load(file_path):
        """Load graph from disk"""
        if os.path.exists(file_path):
            return nx.read_graphml(file_path)
        return nx.Graph()
```

## Graph Algorithms

### 1. Traversal (BFS/DFS)

```python
def extract_subgraph_bfs(graph: nx.Graph, seed_nodes: list, depth: int = 2):
    """
    Breadth-First Search subgraph extraction.

    Used in: Local/Global mode query processing
    """
    visited = set()
    queue = [(node, 0) for node in seed_nodes]
    subgraph_nodes = []
    subgraph_edges = []

    while queue:
        current, current_depth = queue.pop(0)

        if current in visited or current_depth > depth:
            continue

        visited.add(current)
        subgraph_nodes.append(current)

        if current_depth < depth:
            for neighbor in graph.neighbors(current):
                if neighbor not in visited:
                    queue.append((neighbor, current_depth + 1))
                    subgraph_edges.append((current, neighbor))

    # Extract induced subgraph
    subgraph = graph.subgraph(subgraph_nodes).copy()
    return subgraph
```

**Use Case**: Local mode (1-2 hops), Global mode (3+ hops) from seed entities.

### 2. Centrality Measures

```python
# Degree centrality (hub identification)
degree_centrality = nx.degree_centrality(graph)
hubs = sorted(degree_centrality.items(), key=lambda x: x[1], reverse=True)[:10]

# Betweenness centrality (bridge identification)
betweenness = nx.betweenness_centrality(graph)
bridges = sorted(betweenness.items(), key=lambda x: x[1], reverse=True)[:10]

# PageRank (importance ranking)
pagerank = nx.pagerank(graph)
important_nodes = sorted(pagerank.items(), key=lambda x: x[1], reverse=True)[:10]
```

**Use Case**: Identify major attractors (hubs), critical connectors (bridges).

### 3. Community Detection

```python
# Using Louvain algorithm (requires python-louvain)
import community as community_louvain

def detect_communities(graph: nx.Graph):
    """
    Detect thematic communities in knowledge graph.

    Algorithm: Louvain method (modularity optimization)
    """
    # Partition graph into communities
    partition = community_louvain.best_partition(graph)

    # Group nodes by community ID
    communities = {}
    for node, comm_id in partition.items():
        if comm_id not in communities:
            communities[comm_id] = []
        communities[comm_id].append(node)

    return communities

# communities = {
#     0: ["Apple Inc", "iPhone", "iPad", "Mac", "Tim Cook"],  # Tech cluster
#     1: ["Samsung", "Galaxy", "Android", "Lee Jae-yong"],    # Competitor cluster
#     2: ["AI", "Machine Learning", "Neural Networks"],       # AI cluster
# }
```

**Use Case**: Global mode - identify thematic clusters for comprehensive reasoning.

### 4. Shortest Path

```python
def find_relationship_path(graph, entity1, entity2):
    """
    Find shortest path between two entities.

    Use Case: Explain how two entities are related.
    """
    try:
        path = nx.shortest_path(graph, entity1, entity2)
        # path = ["Apple Inc", "produces", "iPhone"]
        return path
    except nx.NetworkXNoPath:
        return None  # No connection exists
```

### 5. Connected Components

```python
# Find all connected components
components = list(nx.connected_components(graph))

# Largest component (main knowledge graph)
largest_component = max(components, key=len)

# Isolated entities (orphans)
isolated = [node for node in graph.nodes if graph.degree(node) == 0]
```

**Use Case**: Identify disconnected knowledge islands, quality assurance.

---

## Graph Metrics

### Global Metrics

```python
# Number of nodes and edges
num_nodes = graph.number_of_nodes()
num_edges = graph.number_of_edges()

# Average degree (connectivity)
avg_degree = sum(dict(graph.degree()).values()) / num_nodes

# Density (how connected)
density = nx.density(graph)  # edges / max_possible_edges

# Clustering coefficient (local connectivity)
clustering = nx.average_clustering(graph)
```

### Path Metrics

```python
# Average shortest path length (for connected graph)
if nx.is_connected(graph):
    avg_path_length = nx.average_shortest_path_length(graph)
else:
    # For disconnected graph, use largest component
    largest_cc = max(nx.connected_components(graph), key=len)
    subgraph = graph.subgraph(largest_cc)
    avg_path_length = nx.average_shortest_path_length(subgraph)

# Diameter (longest shortest path)
diameter = nx.diameter(subgraph)
```

**Use Case**: Understand graph structure, optimize traversal depth.

---

## Graph Visualization

```python
import matplotlib.pyplot as plt

def visualize_subgraph(graph, entities, depth=2):
    """Visualize subgraph around seed entities"""
    subgraph = extract_subgraph_bfs(graph, entities, depth)

    # Layout
    pos = nx.spring_layout(subgraph, k=0.5, iterations=50)

    # Node colors by type
    node_colors = []
    for node in subgraph.nodes():
        entity_type = subgraph.nodes[node].get("entity_type", "unknown")
        color_map = {
            "person": "lightblue",
            "organization": "lightgreen",
            "location": "lightyellow",
            "concept": "lightcoral"
        }
        node_colors.append(color_map.get(entity_type, "lightgray"))

    # Node sizes by degree
    node_sizes = [graph.degree(node) * 100 for node in subgraph.nodes()]

    # Draw
    nx.draw(
        subgraph,
        pos,
        node_color=node_colors,
        node_size=node_sizes,
        with_labels=True,
        font_size=8,
        edge_color="gray",
        alpha=0.7
    )

    plt.title(f"Subgraph around {entities} (depth={depth})")
    plt.show()
```

---

## Graph Formats

### GraphML (Default)

```python
# Save
nx.write_graphml(graph, "knowledge_graph.graphml")

# Load
graph = nx.read_graphml("knowledge_graph.graphml")
```

**Format**: XML-based, preserves all attributes.

**Advantages**:
- Human-readable (XML)
- Preserves data types
- Standard format

**Disadvantages**:
- Large file size
- Slower I/O

### Alternative Formats

```python
# GEXF (Gephi format)
nx.write_gexf(graph, "graph.gexf")

# JSON
import json
data = nx.node_link_data(graph)
with open("graph.json", "w") as f:
    json.dump(data, f)

# Pickle (fastest, Python-only)
nx.write_gpickle(graph, "graph.pkl")

# Edge list (simple, no attributes)
nx.write_edgelist(graph, "graph.edgelist")
```

---

## Performance Characteristics

| Operation | Complexity | Time (10K nodes) | Time (100K nodes) |
|-----------|-----------|------------------|-------------------|
| **Add node** | O(1) | < 1ms | < 1ms |
| **Add edge** | O(1) | < 1ms | < 1ms |
| **Get neighbors** | O(1) | < 1ms | < 1ms |
| **BFS (depth 2)** | O(V+E) | 10-50ms | 100-500ms |
| **Community detection** | O(V log V) | 1-5s | 10-60s |
| **Shortest path** | O(V+E) | 10-100ms | 100-1000ms |
| **Save GraphML** | O(V+E) | 100-500ms | 1-5s |

---

## Optimization Strategies

### 1. Lazy Loading

```python
class LazyGraph:
    """Load graph only when needed"""

    def __init__(self, file_path):
        self.file_path = file_path
        self._graph = None

    @property
    def graph(self):
        if self._graph is None:
            self._graph = nx.read_graphml(self.file_path)
        return self._graph
```

### 2. Subgraph Caching

```python
# Cache frequently accessed subgraphs
subgraph_cache = {}

def get_subgraph_cached(seed_entities, depth):
    cache_key = f"{sorted(seed_entities)}:{depth}"

    if cache_key in subgraph_cache:
        return subgraph_cache[cache_key]

    subgraph = extract_subgraph_bfs(graph, seed_entities, depth)
    subgraph_cache[cache_key] = subgraph

    return subgraph
```

### 3. Incremental Updates

```python
# Instead of rewriting entire graph
def incremental_save(graph, new_nodes, new_edges, file_path):
    """Append-only updates"""
    # Save only delta
    delta_graph = nx.Graph()
    delta_graph.add_nodes_from(new_nodes)
    delta_graph.add_edges_from(new_edges)

    # Merge with main graph (in-memory)
    graph.update(delta_graph)

    # Periodic full save (e.g., every 100 updates)
    if update_counter % 100 == 0:
        nx.write_graphml(graph, file_path)
```

---

## Alternative Graph Databases

While NetworkX is default, LightRAG supports dedicated graph databases:

### Neo4j

```python
from neo4j import AsyncGraphDatabase

driver = AsyncGraphDatabase.driver(uri, auth=(user, password))

# Cypher query
async with driver.session() as session:
    result = await session.run(
        "MATCH (a)-[r]->(b) WHERE a.name = $name RETURN b",
        name="Apple Inc"
    )
```

**Use Case**: Production-grade, ACID, billions of nodes/edges.

### Memgraph

```python
from memgraph import Memgraph

memgraph = Memgraph(host="localhost", port=7687)
memgraph.execute(
    "MATCH (a)-[r]->(b) WHERE a.name = 'Apple Inc' RETURN b"
)
```

**Use Case**: In-memory, high-performance, real-time analytics.

---

## Related Documentation

- **[Graph Topology](../nodes/03-graph-topology.md)** - Graph structure analysis
- **[Star Attractor Pattern](../research/02-star-attractor-pattern.md)** - Hub-spoke topology
- **[Semantic Traversal](../research/04-semantic-traversal-methods.md)** - BFS traversal methods

---

**Версия**: 1.0
**Дата**: 2025-01-13
