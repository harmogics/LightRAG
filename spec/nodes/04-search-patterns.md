# Search Patterns: Паттерны Поиска по Графу

## Обзор

LightRAG использует разнообразные паттерны поиска по Knowledge Graph, каждый оптимизированный для специфичных типов queries. Выбор правильного паттерна critical для performance и качества результатов.

## Query Mode Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    QUERY PROCESSING                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Input Query                                                │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────┐                                       │
│  │ Query Classifier│                                       │
│  └────────┬────────┘                                       │
│           │                                                 │
│     ┌─────┴──────────┬─────────────┬──────────────┐      │
│     │                │             │              │      │
│     ▼                ▼             ▼              ▼      │
│ ┌────────┐     ┌─────────┐   ┌────────┐    ┌─────────┐ │
│ │ Naive  │     │  Local  │   │ Global │    │ Hybrid  │ │
│ │  Mode  │     │  Mode   │   │  Mode  │    │  Mode   │ │
│ └────────┘     └─────────┘   └────────┘    └─────────┘ │
│      │              │             │              │        │
│      └──────────────┴─────────────┴──────────────┘        │
│                         │                                  │
│                         ▼                                  │
│                  ┌────────────┐                           │
│                  │   Answer   │                           │
│                  │ Generation │                           │
│                  └────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

## Mode 1: Naive Mode (Chunk-Only)

### Strategy

**Подход**: Прямой semantic search по text chunks без использования graph structure

```
Query → Vector Embedding → Vector Search → Top-K Chunks → LLM → Answer
```

### Implementation

```python
async def naive_search(query: str, top_k: int = 10):
    """
    Naive RAG: direct chunk retrieval

    Steps:
    1. Generate query embedding
    2. Vector similarity search
    3. Retrieve top-k chunks
    4. Pass to LLM for answer generation
    """

    # Generate query embedding
    query_embedding = await embedding_func([query])[0]

    # Vector search in chunks VDB
    results = await chunks_vdb.query(
        query_text=query,
        top_k=top_k
    )

    # Format chunks for LLM
    context_chunks = [
        {
            "content": result["content"],
            "score": result["score"],
            "source": result["file_path"]
        }
        for result in results
    ]

    return context_chunks
```

### When to Use

✅ **Good For**:
- Simple factual queries
- Direct information lookup
- Quick responses needed
- Single-document questions

❌ **Not Good For**:
- Complex reasoning
- Multi-hop questions
- Entity-centric queries
- Relationship exploration

### Example

```
Query: "What is the capital of France?"

Process:
1. Vector search finds chunk: "Paris is the capital of France..."
2. Return chunk directly
3. LLM generates: "The capital of France is Paris"

Time: ~50ms
Hops: 0 (no graph traversal)
```

### Performance

| Metric | Value |
|--------|-------|
| **Latency** | 50-100ms |
| **Precision** | 0.6-0.7 |
| **Recall** | 0.5-0.6 |
| **Use Case** | Simple facts |

---

## Mode 2: Local Mode (Entity-Centric + 1-2 Hops)

### Strategy

**Подход**: Entity-focused search с локальным graph traversal

```
Query → Keyword Extraction → Entity Search → Local Subgraph (1-2 hops)
     → Related Chunks → LLM → Answer
```

### Implementation

```python
async def local_search(
    query: str,
    top_k_entities: int = 10,
    depth: int = 2,
    max_nodes: int = 50
):
    """
    Local mode: entity-centric with local graph context

    Steps:
    1. Extract keywords from query
    2. Find relevant entities (vector search)
    3. Extract local subgraph (1-2 hops)
    4. Retrieve chunks for entities
    5. Generate answer with entity context
    """

    # === KEYWORD EXTRACTION ===
    keywords = await extract_keywords_with_llm(query)
    # {
    #     "high_level": ["Apple Inc", "products"],
    #     "low_level": ["iPhone", "iPad", "innovation"]
    # }

    # === ENTITY SEARCH ===
    # Search by high-level keywords
    relevant_entities = await entity_vdb.query(
        query_text=" ".join(keywords["high_level"]),
        top_k=top_k_entities
    )

    entity_names = [e["entity_name"] for e in relevant_entities]

    # === LOCAL SUBGRAPH EXTRACTION ===
    subgraph = await knowledge_graph_inst.get_knowledge_graph(
        entity_names=entity_names,
        depth=depth,
        max_nodes=max_nodes
    )

    # subgraph = {
    #     "nodes": [list of entity nodes],
    #     "edges": [list of relationships]
    # }

    # === CHUNK RETRIEVAL ===
    all_chunks = []
    for entity in subgraph["nodes"]:
        # Get chunks where entity was mentioned
        entity_chunks = await get_chunks_for_entity(
            entity["source_id"]
        )
        all_chunks.extend(entity_chunks)

    # Deduplicate chunks
    unique_chunks = deduplicate_by_id(all_chunks)

    # === CONTEXT ASSEMBLY ===
    context = {
        "query": query,
        "entities": subgraph["nodes"],
        "relationships": subgraph["edges"],
        "chunks": unique_chunks[:20]  # Limit for LLM context
    }

    return context
```

### Graph Traversal (BFS)

```python
async def extract_local_subgraph(
    seed_entities: list[str],
    depth: int = 2,
    max_nodes: int = 50
):
    """
    BFS traversal from seed entities

    Algorithm:
    1. Start from seed entities (depth 0)
    2. Explore neighbors (depth 1)
    3. Explore neighbors of neighbors (depth 2)
    4. Stop at max_depth or max_nodes
    """

    visited = set()
    queue = [(entity, 0) for entity in seed_entities]  # (entity, depth)
    nodes = []
    edges = []

    while queue and len(visited) < max_nodes:
        current_entity, current_depth = queue.pop(0)

        if current_entity in visited:
            continue

        if current_depth > depth:
            continue

        visited.add(current_entity)

        # Get node data
        node_data = await knowledge_graph_inst.get_node(current_entity)
        if node_data:
            nodes.append(node_data)

        # Get neighbors if within depth
        if current_depth < depth:
            neighbors = await knowledge_graph_inst.get_neighbors(
                current_entity
            )

            for neighbor in neighbors:
                if neighbor not in visited:
                    queue.append((neighbor, current_depth + 1))

                    # Get edge data
                    edge_data = await knowledge_graph_inst.get_edge(
                        current_entity, neighbor
                    )
                    if edge_data:
                        edges.append(edge_data)

    return {"nodes": nodes, "edges": edges}
```

### When to Use

✅ **Good For**:
- Entity-specific questions
- "What/Who/Where" queries
- Relationship exploration (1-2 hops)
- Moderate complexity reasoning

❌ **Not Good For**:
- Very broad queries
- Deep multi-hop reasoning (>3 hops)
- Unstructured questions

### Example

```
Query: "What products does Apple Inc manufacture?"

Process:
1. Keywords: ["Apple Inc", "products", "manufacture"]
2. Entity search finds: "Apple Inc" (Organization)
3. Local subgraph (1 hop):
   - Apple Inc -[produces]-> iPhone
   - Apple Inc -[produces]-> iPad
   - Apple Inc -[produces]-> Mac
4. Retrieve chunks for these entities
5. LLM generates comprehensive answer

Time: ~200ms
Hops: 1-2
Entities: 5-15
```

### Performance

| Metric | Value |
|--------|-------|
| **Latency** | 150-300ms |
| **Precision** | 0.75-0.85 |
| **Recall** | 0.70-0.80 |
| **Use Case** | Entity queries |

---

## Mode 3: Global Mode (Full Graph Exploration)

### Strategy

**Подход**: Comprehensive graph traversal для complex reasoning

```
Query → Keywords (High+Low) → Seed Entities → Full Subgraph (3+ hops)
     → Community Detection → Comprehensive Chunks → LLM → Detailed Answer
```

### Implementation

```python
async def global_search(
    query: str,
    max_depth: int = 3,
    max_nodes: int = 200,
    top_k_entities: int = 5
):
    """
    Global mode: comprehensive graph reasoning

    Steps:
    1. Extract hierarchical keywords
    2. Find seed entities (high-level keywords)
    3. Extract large subgraph (3+ hops)
    4. Detect communities in subgraph
    5. Retrieve chunks from all relevant communities
    6. Generate comprehensive answer
    """

    # === HIERARCHICAL KEYWORD EXTRACTION ===
    keywords = await extract_keywords_with_llm(query)
    # {
    #     "high_level": ["technology", "innovation"],
    #     "low_level": ["smartphones", "AI", "user experience"]
    # }

    # === SEED ENTITY SEARCH ===
    seed_entities = await entity_vdb.query(
        query_text=" ".join(keywords["high_level"]),
        top_k=top_k_entities
    )

    seed_names = [e["entity_name"] for e in seed_entities]

    # === GLOBAL SUBGRAPH EXTRACTION ===
    subgraph = await knowledge_graph_inst.get_knowledge_graph(
        entity_names=seed_names,
        depth=max_depth,
        max_nodes=max_nodes
    )

    # === COMMUNITY DETECTION ===
    # Build NetworkX graph from subgraph
    nx_graph = build_networkx_graph(subgraph)

    # Detect communities using Louvain
    communities = detect_communities_louvain(nx_graph)
    # {
    #     0: ["Apple Inc", "iPhone", "Tim Cook", ...],
    #     1: ["AI", "Machine Learning", "Neural Networks", ...],
    #     2: ["Google", "Android", "Sundar Pichai", ...]
    # }

    # === COMMUNITY RANKING ===
    # Rank communities by relevance to query
    ranked_communities = await rank_communities_by_relevance(
        communities,
        query,
        keywords
    )

    # === COMPREHENSIVE CHUNK RETRIEVAL ===
    all_chunks = []
    for comm_id in ranked_communities[:3]:  # Top 3 communities
        community_entities = communities[comm_id]

        for entity in community_entities:
            entity_data = next(
                (n for n in subgraph["nodes"] if n["entity_name"] == entity),
                None
            )
            if entity_data:
                chunks = await get_chunks_for_entity(
                    entity_data["source_id"]
                )
                all_chunks.extend(chunks)

    # Deduplicate and rank chunks
    unique_chunks = deduplicate_and_rank(all_chunks, query)

    # === CONTEXT ASSEMBLY ===
    context = {
        "query": query,
        "subgraph": subgraph,
        "communities": {
            comm_id: communities[comm_id]
            for comm_id in ranked_communities[:3]
        },
        "chunks": unique_chunks[:30]  # More chunks for comprehensive answer
    }

    return context
```

### Community Ranking

```python
async def rank_communities_by_relevance(
    communities: dict,
    query: str,
    keywords: dict
):
    """
    Rank communities by relevance to query

    Scoring:
    1. Keyword overlap with community entities
    2. Average entity relevance (vector similarity)
    3. Community size (moderate penalty for very large)
    """

    community_scores = []

    for comm_id, members in communities.items():
        # Keyword overlap score
        keyword_overlap = len([
            kw for kw in keywords["low_level"]
            if any(kw.lower() in member.lower() for member in members)
        ]) / len(keywords["low_level"])

        # Average entity relevance
        member_scores = []
        for member in members[:10]:  # Sample for efficiency
            entity_data = await entity_vdb.get_by_id(member)
            if entity_data and "embedding" in entity_data:
                query_emb = await embedding_func([query])[0]
                similarity = cosine_similarity(query_emb, entity_data["embedding"])
                member_scores.append(similarity)

        avg_relevance = sum(member_scores) / len(member_scores) if member_scores else 0

        # Size penalty (prefer moderate-sized communities)
        size_score = 1.0 - abs(len(members) - 20) / 100  # Ideal size ~20

        # Combined score
        final_score = (
            0.4 * keyword_overlap +
            0.4 * avg_relevance +
            0.2 * size_score
        )

        community_scores.append((comm_id, final_score))

    # Sort by score descending
    community_scores.sort(key=lambda x: x[1], reverse=True)

    return [comm_id for comm_id, score in community_scores]
```

### When to Use

✅ **Good For**:
- Complex reasoning questions
- Multi-hop queries (>2 hops)
- Broad exploratory questions
- "How/Why" questions
- Comparative analysis

❌ **Not Good For**:
- Simple facts (overkill)
- Time-sensitive queries (slower)
- Resource-constrained environments

### Example

```
Query: "How has AI technology influenced smartphone development?"

Process:
1. Keywords:
   - High-level: ["AI", "technology", "smartphone", "development"]
   - Low-level: ["machine learning", "neural networks", "features"]
2. Seed entities: ["AI", "Machine Learning", "Smartphone"]
3. Global subgraph (3 hops):
   - AI → ML algorithms → Face recognition → iPhone
   - AI → Neural networks → Camera processing → Smartphone photography
   - AI → Natural language → Voice assistants → Siri/Google Assistant
4. Communities detected:
   - Community 0: AI/ML concepts
   - Community 1: Smartphone products
   - Community 2: Features/applications
5. Retrieve chunks from all communities
6. LLM generates comprehensive multi-paragraph answer

Time: ~800ms
Hops: 3-4
Entities: 50-100
Communities: 3-5
```

### Performance

| Metric | Value |
|--------|-------|
| **Latency** | 500-1000ms |
| **Precision** | 0.80-0.90 |
| **Recall** | 0.80-0.90 |
| **Use Case** | Complex reasoning |

---

## Mode 4: Hybrid Mode (Adaptive)

### Strategy

**Подход**: Автоматический выбор оптимальной стратегии

```
Query → Query Analysis → Mode Selection → Appropriate Strategy → Answer
```

### Query Classification

```python
async def classify_query_complexity(query: str):
    """
    Classify query to select appropriate mode

    Factors:
    1. Query length
    2. Question type (what/who/how/why)
    3. Entity count
    4. Relationship indicators
    5. Comparative/analytical keywords
    """

    # LLM-based classification
    classification_prompt = f"""
    Analyze this query and classify its complexity:

    Query: {query}

    Classifications:
    - simple: Direct fact lookup
    - entity_specific: Focus on specific entities
    - complex: Multi-hop reasoning, comparisons, analysis

    Output JSON:
    {{
        "complexity": "simple|entity_specific|complex",
        "reasoning": "brief explanation",
        "suggested_mode": "naive|local|global"
    }}
    """

    result = await llm_func(classification_prompt)
    classification = json.loads(result)

    return classification
```

### Mode Selection Logic

```python
async def hybrid_search(query: str):
    """
    Adaptive mode selection based on query analysis
    """

    # Classify query
    classification = await classify_query_complexity(query)

    mode = classification["suggested_mode"]

    if mode == "naive":
        logger.info(f"Using Naive mode for query: {query}")
        return await naive_search(query)

    elif mode == "local":
        logger.info(f"Using Local mode for query: {query}")
        return await local_search(query)

    elif mode == "global":
        logger.info(f"Using Global mode for query: {query}")
        return await global_search(query)

    else:
        # Default to local mode
        logger.warning(f"Unknown mode, defaulting to Local: {mode}")
        return await local_search(query)
```

### Fallback Strategy

```python
async def hybrid_search_with_fallback(query: str):
    """
    Try modes in order of complexity with fallback
    """

    # Try simple first
    try:
        result = await naive_search(query)
        if is_confident_answer(result):
            return result
    except Exception as e:
        logger.warning(f"Naive mode failed: {e}")

    # Fallback to local
    try:
        result = await local_search(query)
        if is_confident_answer(result):
            return result
    except Exception as e:
        logger.warning(f"Local mode failed: {e}")

    # Fallback to global
    return await global_search(query)
```

### Performance

| Metric | Value |
|--------|-------|
| **Latency** | Variable (50-1000ms) |
| **Precision** | 0.75-0.90 |
| **Recall** | 0.70-0.85 |
| **Use Case** | General purpose |

---

## Advanced Search Techniques

### 1. Personalized PageRank Search

```python
async def personalized_pagerank_search(
    query: str,
    seed_entities: list[str]
):
    """
    Use Personalized PageRank for entity ranking

    Advantages:
    - Query-specific entity importance
    - Accounts for graph structure
    - Balances local and global relevance
    """

    # Get full graph
    graph = await knowledge_graph_inst.get_full_graph()

    # Personalization vector (bias towards seed entities)
    personalization = {
        entity: 1.0 if entity in seed_entities else 0.0
        for entity in graph.nodes()
    }

    # Compute Personalized PageRank
    ppr_scores = nx.pagerank(
        graph,
        personalization=personalization,
        alpha=0.85
    )

    # Rank entities by PPR score
    ranked_entities = sorted(
        ppr_scores.items(),
        key=lambda x: x[1],
        reverse=True
    )

    return ranked_entities[:50]  # Top 50
```

### 2. HITS Algorithm (Hubs and Authorities)

```python
async def hits_search(query: str, seed_entities: list[str]):
    """
    HITS algorithm for finding authoritative entities

    Hub: Points to many authorities
    Authority: Pointed to by many hubs

    Good for: Finding influential entities
    """

    subgraph = await extract_subgraph(seed_entities, depth=2)
    nx_graph = build_networkx_graph(subgraph)

    # Compute HITS scores
    hubs, authorities = nx.hits(nx_graph)

    # Return top authorities (most referenced)
    top_authorities = sorted(
        authorities.items(),
        key=lambda x: x[1],
        reverse=True
    )

    return top_authorities[:20]
```

### 3. Multi-Path Reasoning

```python
async def multi_path_search(
    source_entity: str,
    target_entity: str,
    max_paths: int = 5
):
    """
    Find multiple paths between entities

    Use case: Explaining relationships
    """

    paths = await find_all_paths(
        source_entity,
        target_entity,
        max_depth=4,
        max_paths=max_paths
    )

    # Rank paths by:
    # 1. Length (shorter better)
    # 2. Relationship strength (weight)
    # 3. Entity importance (PageRank)

    ranked_paths = rank_paths_by_quality(paths)

    return ranked_paths
```

---

## Performance Optimization

### 1. Caching Strategies

```python
# Subgraph cache
subgraph_cache = TTLCache(maxsize=1000, ttl=3600)  # 1 hour

async def get_subgraph_cached(entity_names, depth):
    cache_key = f"{sorted(entity_names)}:{depth}"

    if cache_key in subgraph_cache:
        return subgraph_cache[cache_key]

    subgraph = await extract_subgraph(entity_names, depth)
    subgraph_cache[cache_key] = subgraph

    return subgraph
```

### 2. Parallel Execution

```python
async def parallel_search(query: str):
    """
    Execute multiple search strategies in parallel
    """

    # Run multiple modes concurrently
    results = await asyncio.gather(
        naive_search(query),
        local_search(query),
        # global_search(query),  # Skip for faster response
        return_exceptions=True
    )

    # Merge results
    merged = merge_search_results(results)

    return merged
```

### 3. Progressive Loading

```python
async def progressive_search(query: str):
    """
    Return quick results first, then refine
    """

    # Quick naive search
    quick_results = await naive_search(query, top_k=5)
    yield {"type": "quick", "results": quick_results}

    # Refined local search
    local_results = await local_search(query)
    yield {"type": "refined", "results": local_results}

    # Optional: Deep global search
    if needs_deep_search(quick_results):
        global_results = await global_search(query)
        yield {"type": "comprehensive", "results": global_results}
```

---

**Следующий раздел**: [05-answer-generation.md](05-answer-generation.md) - Формулировка ответов на основе графа
