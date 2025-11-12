# Semantic Layer: Концептуальный и Семантический Слой

## Обзор

Semantic Layer в LightRAG представляет собой многоуровневую систему семантического представления знаний, объединяющую векторные embeddings, структуру графа знаний и гибридные методы поиска. Этот слой обеспечивает глубокое понимание концептуальных связей между entities и эффективный semantic retrieval.

## Архитектура Semantic Layer

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEMANTIC LAYER ARCHITECTURE                   │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           LEVEL 1: VECTOR REPRESENTATIONS                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │ │
│  │  │   Chunk      │  │   Entity     │  │  Relationship   │ │ │
│  │  │  Embeddings  │  │  Embeddings  │  │   Embeddings    │ │ │
│  │  └──────────────┘  └──────────────┘  └─────────────────┘ │ │
│  │          ↓                 ↓                   ↓           │ │
│  │       Dense Retrieval   Semantic Entity   Semantic Path   │ │
│  │       (Text Similarity) Search            Search          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              ↓                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           LEVEL 2: GRAPH STRUCTURE SEMANTICS               │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │ │
│  │  │   Entity     │  │ Relationship │  │   Community     │ │ │
│  │  │  Co-occur    │  │    Paths     │  │   Detection     │ │ │
│  │  └──────────────┘  └──────────────┘  └─────────────────┘ │ │
│  │          ↓                 ↓                   ↓           │ │
│  │    Graph Traversal   Multi-hop        Cluster-based      │ │
│  │    (BFS/DFS)        Reasoning          Analysis          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              ↓                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         LEVEL 3: HYBRID SEMANTIC SEARCH                    │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │ │
│  │  │  Vector +    │  │  Graph +     │  │    Ranking      │ │ │
│  │  │   Graph      │  │  Keywords    │  │    Fusion       │ │ │
│  │  └──────────────┘  └──────────────┘  └─────────────────┘ │ │
│  │          ↓                 ↓                   ↓           │ │
│  │   Contextual Retrieval  Query Expansion   Score Fusion   │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Level 1: Vector Representations

### 1.1 Chunk Embeddings

**Назначение**: Семантическое представление текстовых фрагментов

#### Embedding Generation

```python
# В процессе chunking → vector DB upsert
async def upsert_chunk_embeddings(chunks: dict):
    """
    Генерация embeddings для chunks

    Input Format:
    {
        "chunk-abc123": {
            "content": "Text content of the chunk...",
            "tokens": 512,
            "full_doc_id": "doc-xyz789",
            "file_path": "document.pdf"
        }
    }
    """

    # Извлечение текстов
    texts = [chunk["content"] for chunk in chunks.values()]

    # Batch embedding generation
    embeddings = await embedding_func(texts)

    # Upsert в vector DB
    data_for_vdb = {
        chunk_id: {
            "content": chunk["content"],
            "embedding": embedding,
            **chunk  # Metadata
        }
        for (chunk_id, chunk), embedding in zip(chunks.items(), embeddings)
    }

    await chunks_vdb.upsert(data_for_vdb)
```

#### Semantic Search по Chunks

```python
async def semantic_chunk_search(query: str, top_k: int = 10):
    """
    Dense retrieval - поиск семантически похожих chunks

    Process:
    1. Generate query embedding
    2. Cosine similarity search в vector DB
    3. Return top-k chunks с metadata
    """

    results = await chunks_vdb.query(
        query_text=query,
        top_k=top_k
    )

    # Results format:
    # [
    #     {
    #         "id": "chunk-abc123",
    #         "score": 0.87,
    #         "content": "...",
    #         "full_doc_id": "doc-xyz789",
    #         "file_path": "document.pdf"
    #     }
    # ]

    return results
```

#### Use Cases

- **Naive RAG**: прямой retrieval chunks для query answering
- **Context Expansion**: расширение контекста вокруг entity
- **Document Similarity**: поиск похожих документов
- **Citation Tracking**: трассировка к исходным документам

### 1.2 Entity Embeddings

**Назначение**: Семантическое представление сущностей

#### Embedding Content Format

```python
# Content для embedding генерируется как:
content = f"{entity_name}\n{entity_description}"

# Пример:
content = """Apple Inc
Apple Inc is a multinational technology company that designs, develops,
and sells consumer electronics, computer software, and online services.
Founded in 1976 by Steve Jobs, Steve Wozniak, and Ronald Wayne..."""
```

**Обоснование**:
- Entity name обеспечивает lexical matching
- Description обеспечивает semantic context
- Комбинация улучшает retrieval accuracy

#### Entity Search

```python
async def semantic_entity_search(query: str, top_k: int = 10):
    """
    Semantic search по entities

    Use Cases:
    - Entity disambiguation
    - Concept discovery
    - Knowledge exploration
    """

    results = await entity_vdb.query(
        query_text=query,
        top_k=top_k
    )

    # Results:
    # [
    #     {
    #         "id": "ent-abc123",
    #         "score": 0.92,
    #         "entity_name": "Apple Inc",
    #         "entity_type": "organization",
    #         "description": "...",
    #         "source_id": "chunk-xyz789",
    #         "file_path": "document.pdf"
    #     }
    # ]

    return results
```

#### Entity Similarity

```python
def compute_entity_similarity(entity1: str, entity2: str):
    """
    Вычисление semantic similarity между entities

    Applications:
    - Entity clustering
    - Synonym detection
    - Concept relationships
    """

    # Получение embeddings
    emb1 = await entity_vdb.get_embedding(entity1)
    emb2 = await entity_vdb.get_embedding(entity2)

    # Cosine similarity
    similarity = cosine_similarity(emb1, emb2)

    return similarity
```

### 1.3 Relationship Embeddings

**Назначение**: Семантическое представление отношений

#### Embedding Content Format

```python
# Content для relationship embedding:
content = f"{source_entity} -> {target_entity}\n"
content += f"Keywords: {keywords}\n"
content += f"{description}"

# Пример:
content = """Tim Cook -> Apple Inc
Keywords: leadership, management, CEO
Tim Cook serves as the Chief Executive Officer of Apple Inc, leading
the company's strategic direction and operations since 2011."""
```

**Структура**:
- **Directional context**: source → target
- **Keyword summary**: high-level themes
- **Detailed description**: relationship nature

#### Relationship Search

```python
async def semantic_relationship_search(query: str, top_k: int = 10):
    """
    Semantic search по relationships

    Use Cases:
    - Finding specific types of relationships
    - Discovering implicit connections
    - Pattern matching in relationships
    """

    results = await relationships_vdb.query(
        query_text=query,
        top_k=top_k
    )

    # Results:
    # [
    #     {
    #         "id": "rel-abc123",
    #         "score": 0.89,
    #         "source_entity": "Tim Cook",
    #         "target_entity": "Apple Inc",
    #         "keywords": "leadership, management, CEO",
    #         "description": "...",
    #         "source_id": "chunk-xyz789"
    #     }
    # ]

    return results
```

#### Relationship Clustering

```python
async def cluster_relationships_by_type():
    """
    Кластеризация relationships по semantic similarity

    Applications:
    - Automatic relationship type discovery
    - Pattern extraction
    - Relationship taxonomy building
    """

    # Получение всех relationship embeddings
    all_relationships = await relationships_vdb.get_all()

    # Clustering (например, K-Means)
    from sklearn.cluster import KMeans

    embeddings = [rel["embedding"] for rel in all_relationships]
    kmeans = KMeans(n_clusters=10)
    clusters = kmeans.fit_predict(embeddings)

    # Группировка по кластерам
    clustered_rels = defaultdict(list)
    for rel, cluster_id in zip(all_relationships, clusters):
        clustered_rels[cluster_id].append(rel)

    return clustered_rels
```

## Level 2: Graph Structure Semantics

### 2.1 Entity Co-occurrence

**Концепция**: Entities связанные через общие chunks или relationships

#### Co-occurrence Matrix

```python
def build_entity_cooccurrence_matrix(window_size: int = 3):
    """
    Построение матрицы co-occurrence для entities

    Window:
    - Entities в одном chunk
    - Entities в соседних chunks (в пределах window_size)
    - Entities в одном документе

    Matrix:
    - M[i, j] = количество co-occurrences entity_i и entity_j
    """

    cooccurrence = defaultdict(lambda: defaultdict(int))

    # Для каждого документа
    for doc_id, chunks in documents.items():
        chunk_entities = []

        for chunk in chunks:
            # Извлечение entities из chunk
            entities = extract_entities_from_chunk(chunk)
            chunk_entities.append(entities)

        # Подсчет co-occurrences в window
        for i in range(len(chunk_entities)):
            for j in range(i, min(i + window_size, len(chunk_entities))):
                entities_i = chunk_entities[i]
                entities_j = chunk_entities[j]

                for ent1 in entities_i:
                    for ent2 in entities_j:
                        if ent1 != ent2:
                            key = tuple(sorted([ent1, ent2]))
                            cooccurrence[key[0]][key[1]] += 1

    return cooccurrence
```

#### Applications

- **Implicit relationship discovery**: entities часто встречающиеся вместе
- **Entity clustering**: группировка related entities
- **Context expansion**: finding related entities для query

### 2.2 Relationship Paths

**Концепция**: Multi-hop reasoning через граф знаний

#### Path Finding

```python
async def find_paths(
    source_entity: str,
    target_entity: str,
    max_depth: int = 3,
    max_paths: int = 10
) -> list[list[str]]:
    """
    Поиск путей между entities в графе

    Methods:
    - BFS: shortest paths
    - DFS: all paths (with depth limit)
    - Dijkstra: weighted paths (by relationship strength)

    Returns:
    [
        ["Apple Inc", "Tim Cook", "Stanford University"],
        ["Apple Inc", "iPhone", "Technology Industry"],
        ...
    ]
    """

    paths = []
    queue = [(source_entity, [source_entity])]
    visited = set()

    while queue and len(paths) < max_paths:
        current, path = queue.pop(0)

        if len(path) > max_depth:
            continue

        if current == target_entity:
            paths.append(path)
            continue

        if current in visited:
            continue

        visited.add(current)

        # Получение neighbors из graph DB
        neighbors = await knowledge_graph_inst.get_neighbors(current)

        for neighbor in neighbors:
            new_path = path + [neighbor]
            queue.append((neighbor, new_path))

    return paths
```

#### Path-Based Reasoning

```python
async def explain_relationship(entity1: str, entity2: str):
    """
    Объяснение связи между entities через paths

    Example:
    Query: "How is Tim Cook related to Stanford University?"

    Paths found:
    1. Tim Cook -> Apple Inc -> Stanford University (recruiting)
    2. Tim Cook -> Education -> Stanford University (alumnus)

    Explanation:
    Tim Cook is connected to Stanford University through multiple paths:
    - As CEO of Apple Inc, which has recruiting relationships with Stanford
    - As an alumnus who completed graduate studies at Stanford
    """

    paths = await find_paths(entity1, entity2, max_depth=3)

    # Для каждого path, извлечь relationship descriptions
    explanations = []
    for path in paths:
        path_description = []

        for i in range(len(path) - 1):
            source = path[i]
            target = path[i + 1]

            # Получить relationship
            edge = await knowledge_graph_inst.get_edge(source, target)
            if edge:
                path_description.append(
                    f"{source} -> {target}: {edge['description']}"
                )

        explanations.append("\n".join(path_description))

    return explanations
```

### 2.3 Community Detection

**Концепция**: Кластеризация entities в тематические группы

#### Community Algorithms

```python
async def detect_communities(algorithm: str = "louvain"):
    """
    Обнаружение communities в knowledge graph

    Algorithms:
    - Louvain: modularity optimization
    - Label Propagation: fast, scalable
    - Girvan-Newman: hierarchical
    - Infomap: information flow

    Returns:
    {
        community_0: ["Apple Inc", "Tim Cook", "iPhone", ...],
        community_1: ["Microsoft", "Satya Nadella", "Windows", ...],
        ...
    }
    """

    # Для NetworkX implementation
    if algorithm == "louvain":
        import community as community_louvain

        communities = community_louvain.best_partition(
            knowledge_graph_inst.graph
        )

    elif algorithm == "label_propagation":
        import networkx as nx

        communities_gen = nx.algorithms.community.label_propagation_communities(
            knowledge_graph_inst.graph
        )
        communities = {
            i: list(community)
            for i, community in enumerate(communities_gen)
        }

    return communities
```

#### Community-Based Search

```python
async def community_search(query: str):
    """
    Поиск с использованием community structure

    Process:
    1. Find relevant entities для query
    2. Identify их communities
    3. Expand search to entire community
    4. Return contextualized results
    """

    # Semantic entity search
    relevant_entities = await semantic_entity_search(query, top_k=5)

    # Получение communities
    communities = await detect_communities()

    # Найти communities для relevant entities
    relevant_communities = set()
    for entity in relevant_entities:
        for comm_id, members in communities.items():
            if entity["entity_name"] in members:
                relevant_communities.add(comm_id)

    # Извлечь все entities из relevant communities
    expanded_entities = []
    for comm_id in relevant_communities:
        expanded_entities.extend(communities[comm_id])

    # Получить subgraph для expanded entities
    subgraph = await knowledge_graph_inst.get_knowledge_graph(
        entity_names=expanded_entities,
        depth=2
    )

    return subgraph
```

## Level 3: Hybrid Semantic Search

### 3.1 Vector + Graph Fusion

**Концепция**: Комбинирование vector similarity и graph structure

#### Hybrid Retrieval

```python
async def hybrid_search(
    query: str,
    top_k: int = 10,
    vector_weight: float = 0.5,
    graph_weight: float = 0.5
):
    """
    Hybrid search combining vector and graph signals

    Scoring:
    final_score = vector_weight * vector_score +
                  graph_weight * graph_score

    Graph Score Components:
    - PageRank centrality
    - Degree centrality
    - Community membership
    """

    # === VECTOR RETRIEVAL ===
    vector_results = await entity_vdb.query(query, top_k=top_k * 2)

    # === GRAPH SCORING ===
    # Compute graph metrics
    pagerank = await knowledge_graph_inst.compute_pagerank()
    degree_centrality = await knowledge_graph_inst.compute_degree_centrality()

    # Normalize scores
    max_pr = max(pagerank.values())
    max_dc = max(degree_centrality.values())

    # === FUSION ===
    scored_results = []
    for result in vector_results:
        entity_name = result["entity_name"]

        # Vector score
        vector_score = result["score"]

        # Graph score
        pr_score = pagerank.get(entity_name, 0) / max_pr
        dc_score = degree_centrality.get(entity_name, 0) / max_dc
        graph_score = (pr_score + dc_score) / 2

        # Final score
        final_score = (
            vector_weight * vector_score +
            graph_weight * graph_score
        )

        scored_results.append({
            **result,
            "final_score": final_score,
            "vector_score": vector_score,
            "graph_score": graph_score
        })

    # Sort by final score
    scored_results.sort(key=lambda x: x["final_score"], reverse=True)

    return scored_results[:top_k]
```

### 3.2 Query Expansion

**Концепция**: Расширение query с использованием graph и embeddings

#### Expansion Strategies

```python
async def expand_query(query: str, expansion_method: str = "hybrid"):
    """
    Query expansion для improved retrieval

    Methods:
    1. Synonym expansion: semantically similar terms
    2. Graph expansion: related entities via graph
    3. Hybrid: combination of both
    """

    if expansion_method == "synonym":
        # === SYNONYM EXPANSION ===
        # Поиск семантически похожих entities
        similar_entities = await entity_vdb.query(query, top_k=5)

        expanded_terms = [query] + [
            ent["entity_name"] for ent in similar_entities
        ]

    elif expansion_method == "graph":
        # === GRAPH EXPANSION ===
        # Найти seed entities
        seed_entities = await semantic_entity_search(query, top_k=3)

        # Получить 1-hop neighbors
        expanded_entities = []
        for entity in seed_entities:
            neighbors = await knowledge_graph_inst.get_neighbors(
                entity["entity_name"]
            )
            expanded_entities.extend(neighbors)

        expanded_terms = [query] + expanded_entities

    elif expansion_method == "hybrid":
        # === HYBRID EXPANSION ===
        synonym_terms = await expand_query(query, "synonym")
        graph_terms = await expand_query(query, "graph")

        expanded_terms = list(set(synonym_terms + graph_terms))

    return expanded_terms
```

### 3.3 Ranking Fusion

**Концепция**: Объединение множественных ranking signals

#### Reciprocal Rank Fusion (RRF)

```python
def reciprocal_rank_fusion(
    rankings: list[list[dict]],
    k: int = 60
) -> list[dict]:
    """
    RRF для комбинирования rankings из разных sources

    Formula:
    RRF_score(d) = Σ 1 / (k + rank_i(d))

    где rank_i(d) - ранг документа d в ranking i

    Parameters:
    - rankings: List of ranked results from different retrievers
    - k: Constant (default: 60)

    Returns:
    - Fused ranking
    """

    # Accumulate scores
    scores = defaultdict(float)
    doc_data = {}

    for ranking in rankings:
        for rank, doc in enumerate(ranking, start=1):
            doc_id = doc["id"]
            scores[doc_id] += 1.0 / (k + rank)
            doc_data[doc_id] = doc

    # Sort by RRF score
    fused_ranking = sorted(
        scores.items(),
        key=lambda x: x[1],
        reverse=True
    )

    # Format results
    results = [
        {
            **doc_data[doc_id],
            "rrf_score": score
        }
        for doc_id, score in fused_ranking
    ]

    return results
```

#### Multi-Signal Fusion

```python
async def multi_signal_search(query: str, top_k: int = 10):
    """
    Combining multiple retrieval signals:
    1. Dense retrieval (embeddings)
    2. Sparse retrieval (BM25/keywords)
    3. Graph retrieval (PageRank)
    4. Recency (timestamp)
    """

    # === DENSE RETRIEVAL ===
    dense_results = await entity_vdb.query(query, top_k=50)

    # === SPARSE RETRIEVAL ===
    # Keyword extraction and BM25
    keywords = extract_keywords(query)
    sparse_results = await keyword_search(keywords, top_k=50)

    # === GRAPH RETRIEVAL ===
    # PageRank-weighted search
    graph_results = await pagerank_search(query, top_k=50)

    # === RECENCY ===
    # Boost recent entities
    recency_results = await recency_search(query, top_k=50)

    # === FUSION ===
    fused_results = reciprocal_rank_fusion([
        dense_results,
        sparse_results,
        graph_results,
        recency_results
    ])

    return fused_results[:top_k]
```

## Semantic Query Modes

### Mode 1: Naive Mode

**Описание**: Простой vector search по chunks

```python
async def naive_query(query: str, top_k: int = 10):
    """
    Naive RAG mode: direct chunk retrieval

    Process:
    1. Generate query embedding
    2. Search similar chunks
    3. Return chunks as context
    """

    chunks = await chunks_vdb.query(query, top_k=top_k)

    # Generate response using chunks
    context = "\n\n".join([chunk["content"] for chunk in chunks])

    response = await llm_func(
        f"Query: {query}\n\nContext:\n{context}\n\nAnswer:",
        system_prompt=PROMPTS["naive_rag_response"]
    )

    return response
```

### Mode 2: Local Mode

**Описание**: Entity-focused search с local graph context

```python
async def local_query(query: str, top_k: int = 10):
    """
    Local mode: entity-centric retrieval

    Process:
    1. Extract keywords from query
    2. Find relevant entities
    3. Get local subgraph (1-2 hops)
    4. Retrieve related chunks
    5. Generate response
    """

    # Keyword extraction
    keywords = await extract_keywords_with_llm(query)

    # Entity search
    entities = await entity_vdb.query(
        " ".join(keywords["high_level"]),
        top_k=10
    )

    # Local subgraph
    subgraph = await knowledge_graph_inst.get_knowledge_graph(
        entity_names=[e["entity_name"] for e in entities],
        depth=1,
        max_nodes=50
    )

    # Retrieve chunks for entities
    chunks = []
    for entity in entities:
        entity_chunks = await get_chunks_for_entity(entity["source_id"])
        chunks.extend(entity_chunks)

    # Generate response
    context = format_context(subgraph, chunks)

    response = await llm_func(
        f"Query: {query}\n\nContext:\n{context}\n\nAnswer:",
        system_prompt=PROMPTS["rag_response"]
    )

    return response
```

### Mode 3: Global Mode

**Описание**: Full graph traversal для complex reasoning

```python
async def global_query(query: str):
    """
    Global mode: comprehensive graph reasoning

    Process:
    1. Extract high-level and low-level keywords
    2. Find seed entities (high-level keywords)
    3. Expand to global graph (multi-hop)
    4. Community detection on subgraph
    5. Retrieve chunks from all relevant communities
    6. Generate comprehensive response
    """

    # Keyword extraction
    keywords = await extract_keywords_with_llm(query)

    # Seed entities
    seed_entities = await entity_vdb.query(
        " ".join(keywords["high_level"]),
        top_k=5
    )

    # Global subgraph
    subgraph = await knowledge_graph_inst.get_knowledge_graph(
        entity_names=[e["entity_name"] for e in seed_entities],
        depth=3,
        max_nodes=200
    )

    # Community detection
    communities = detect_communities_in_subgraph(subgraph)

    # Retrieve chunks for all relevant entities
    all_chunks = []
    for community in communities:
        community_chunks = await get_chunks_for_entities(community)
        all_chunks.extend(community_chunks)

    # Generate comprehensive response
    context = format_global_context(subgraph, communities, all_chunks)

    response = await llm_func(
        f"Query: {query}\n\nContext:\n{context}\n\nAnswer:",
        system_prompt=PROMPTS["rag_response"]
    )

    return response
```

### Mode 4: Hybrid Mode

**Описание**: Adaptive mode selection

```python
async def hybrid_query(query: str):
    """
    Hybrid mode: автоматический выбор strategy

    Decision Logic:
    - Simple factual query → Naive mode
    - Entity-specific query → Local mode
    - Complex reasoning query → Global mode
    """

    # Classify query complexity
    query_type = await classify_query(query)

    if query_type == "simple":
        return await naive_query(query)
    elif query_type == "entity_specific":
        return await local_query(query)
    elif query_type == "complex":
        return await global_query(query)
    else:
        # Default to local mode
        return await local_query(query)
```

## Performance Optimizations

### 1. Embedding Caching

```python
# Cache embeddings для frequently accessed entities
embedding_cache = {}

async def get_entity_embedding(entity_name: str):
    if entity_name in embedding_cache:
        return embedding_cache[entity_name]

    embedding = await entity_vdb.get_embedding(entity_name)
    embedding_cache[entity_name] = embedding

    return embedding
```

### 2. Graph Precomputation

```python
# Precompute graph metrics
async def precompute_graph_metrics():
    """
    Precompute expensive graph metrics:
    - PageRank
    - Betweenness centrality
    - Community structure
    """

    pagerank = await knowledge_graph_inst.compute_pagerank()
    betweenness = await knowledge_graph_inst.compute_betweenness()
    communities = await detect_communities()

    # Store in cache
    graph_metrics_cache = {
        "pagerank": pagerank,
        "betweenness": betweenness,
        "communities": communities,
        "timestamp": time.time()
    }

    return graph_metrics_cache
```

### 3. Approximate Search

```python
# Use approximate nearest neighbor for large-scale retrieval
async def approximate_search(query: str, top_k: int = 10):
    """
    HNSW/IVF for fast approximate search

    Trade-off: speed vs accuracy
    """

    # Configure vector DB for approximate search
    results = await entity_vdb.query(
        query,
        top_k=top_k,
        search_params={
            "metric_type": "COSINE",
            "params": {"ef": 64}  # HNSW parameter
        }
    )

    return results
```

## Conclusion

Semantic Layer в LightRAG объединяет:

1. **Vector Representations**: Dense semantic embeddings для chunks, entities, relationships
2. **Graph Structure**: Explicit relationships и multi-hop reasoning
3. **Hybrid Methods**: Fusion векторных и структурных signals

Эта архитектура обеспечивает:
- ✅ **Semantic Understanding**: глубокое понимание концептов
- ✅ **Flexible Retrieval**: multiple query modes
- ✅ **Scalable Performance**: efficient indexing и search
- ✅ **Explainable Results**: трассируемость через graph paths

---

**Это завершает документацию LightRAG Document Processing Pipeline.**

Для дальнейшего изучения см.:
- [Overview](00-overview.md) - общий обзор системы
- [Document Ingestion](01-document-ingestion.md) - детали приема документов
- [Chunking Strategy](02-chunking-strategy.md) - стратегии разбиения
- [Entity Extraction](03-entity-extraction.md) - извлечение сущностей
- [Graph Construction](04-graph-construction.md) - построение графа
- [Storage Architecture](05-storage-architecture.md) - архитектура хранения
- [LLM Agents](06-llm-agents.md) - роли агентов
