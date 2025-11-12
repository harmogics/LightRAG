# LightRAG Graph Structure: Типы Узлов и Связей

## Обзор

Эта директория содержит детальную документацию о структуре графа знаний в LightRAG, включая типы узлов (nodes), типы связей (relationships), топологию графа и их влияние на поиск и формулировку ответов.

## Архитектура Графа

```
┌─────────────────────────────────────────────────────────────────┐
│                    LIGHTRAG GRAPH STRUCTURE                      │
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │  Document    │────>│    Chunk     │────>│   Entity     │   │
│  │   Nodes      │     │    Nodes     │     │   Nodes      │   │
│  └──────────────┘     └──────────────┘     └──────┬───────┘   │
│        │                     │                     │            │
│        │                     │                     │            │
│        v                     v                     v            │
│  [Metadata Layer]      [Content Layer]      [Knowledge Layer]  │
│                                                    │            │
│                                              ┌─────v──────┐    │
│                                              │Relationship│    │
│                                              │   Edges    │    │
│                                              └────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Содержание Документации

### [01-node-types.md](01-node-types.md)
**Типы узлов и их атрибуты**

Описание всех типов узлов в LightRAG:
- **Document Nodes**: Исходные документы
- **Chunk Nodes**: Текстовые фрагменты
- **Entity Nodes**: Извлеченные сущности
  - Person (Человек)
  - Organization (Организация)
  - Location (Место)
  - Event (Событие)
  - Product (Продукт)
  - Concept (Концепция)
  - Category (Категория)
  - Other (Прочее)

Каждый тип описан с:
- Атрибутами и схемой
- Примерами данных
- Use cases
- Vector representations

---

### [02-relationship-types.md](02-relationship-types.md)
**Типы отношений между узлами**

Детальное описание всех типов связей:
- **Structural Relations**: Document→Chunk, Chunk→Entity
- **Semantic Relations**: Entity↔Entity relationships
- **Implicit Relations**: Co-occurrence, citation links
- **Temporal Relations**: Event sequences
- **Hierarchical Relations**: Part-of, belongs-to

Для каждого типа:
- Структура данных
- Направленность (directed/undirected)
- Веса и метрики
- Примеры и паттерны

---

### [03-graph-topology.md](03-graph-topology.md)
**Топология и структура графа**

Анализ структуры Knowledge Graph:
- **Graph Properties**:
  - Scale-free distribution
  - Small-world properties
  - Hub nodes (highly connected entities)
  - Community structure
- **Centrality Metrics**:
  - Degree centrality
  - PageRank
  - Betweenness centrality
  - Closeness centrality
- **Path Analysis**:
  - Average path length
  - Diameter
  - Clustering coefficient
- **Subgraph Patterns**:
  - Star patterns (hub entities)
  - Chain patterns (event sequences)
  - Clique patterns (strongly connected groups)

---

### [04-search-patterns.md](04-search-patterns.md)
**Паттерны поиска по графу**

Как структура графа влияет на поиск:
- **Traversal Strategies**:
  - Breadth-First Search (BFS)
  - Depth-First Search (DFS)
  - Bidirectional Search
  - A* pathfinding
- **Query Modes**:
  - Naive Mode (chunk-only)
  - Local Mode (1-2 hop traversal)
  - Global Mode (full graph exploration)
  - Hybrid Mode (adaptive)
- **Ranking Algorithms**:
  - PageRank-based scoring
  - Personalized PageRank
  - HITS algorithm
  - Graph Neural Network scoring
- **Optimization Techniques**:
  - Index structures
  - Pruning strategies
  - Caching mechanisms

---

### [05-answer-generation.md](05-answer-generation.md)
**Формулировка ответов на основе графа**

Как граф влияет на генерацию ответов:
- **Context Assembly**:
  - Subgraph extraction
  - Entity aggregation
  - Relationship chain building
  - Multi-hop reasoning
- **Evidence Collection**:
  - Source chunk retrieval
  - Citation tracking
  - Confidence scoring
- **Answer Composition**:
  - Template-based generation
  - LLM-based synthesis
  - Structured output formatting
- **Explainability**:
  - Path visualization
  - Source attribution
  - Reasoning chain presentation

---

## Ключевые Концепции

### Node Types Hierarchy

```
All Nodes
├── Metadata Nodes (Infrastructure)
│   ├── Document Node
│   └── Chunk Node
│
└── Knowledge Nodes (Content)
    └── Entity Node
        ├── Person
        ├── Organization
        ├── Location
        ├── Event
        ├── Product
        ├── Concept
        ├── Category
        └── Other
```

### Relationship Types Hierarchy

```
All Relations
├── Structural (System-generated)
│   ├── contains (Document → Chunk)
│   ├── extracted_from (Entity → Chunk)
│   └── cited_in (Entity → Document)
│
├── Semantic (LLM-extracted)
│   ├── Direct Relations
│   │   ├── works_for (Person → Organization)
│   │   ├── located_in (Entity → Location)
│   │   ├── participates_in (Entity → Event)
│   │   └── produces (Organization → Product)
│   │
│   └── Indirect Relations
│       ├── similar_to (Entity ↔ Entity)
│       ├── related_to (Entity ↔ Entity)
│       └── associated_with (Entity ↔ Entity)
│
└── Derived (Computed)
    ├── co-occurs_with (Entity ↔ Entity)
    ├── shares_community (Entity ↔ Entity)
    └── connected_through (Entity ⇝ Entity)
```

## Graph Metrics Overview

| Metric | Typical Value | Description |
|--------|---------------|-------------|
| **Nodes** | 1K-100K | Total entities in graph |
| **Edges** | 5K-500K | Total relationships |
| **Avg Degree** | 5-15 | Average connections per node |
| **Diameter** | 6-10 | Maximum distance between nodes |
| **Clustering** | 0.3-0.6 | Tendency to form clusters |
| **Communities** | 10-100 | Number of detected communities |

## Query Flow Examples

### Example 1: Simple Factual Query

```
Query: "Who is the CEO of Apple Inc?"

Graph Traversal:
1. Entity Search: "Apple Inc" (Organization)
2. Local Traversal: Find connected Person entities
3. Relationship Filter: type contains "CEO" or "leadership"
4. Result: Tim Cook (Person) -[works_as: CEO]-> Apple Inc

Answer: "Tim Cook is the CEO of Apple Inc"
```

### Example 2: Complex Multi-Hop Query

```
Query: "What products has Apple Inc developed using technology from Stanford?"

Graph Traversal:
1. Entity Search: "Apple Inc" (Organization)
2. Find Products: Apple Inc -[produces]-> {iPhone, iPad, Mac, ...}
3. Multi-hop: Products -[uses_technology]-> Technologies -[developed_at]-> Stanford
4. Path Assembly: Apple Inc → Products → Technologies → Stanford
5. Chunk Retrieval: Get source chunks for all entities in path

Answer: "Apple Inc has developed several products using technology from
Stanford, including [specific products with details from chunks]..."
```

## Performance Characteristics

### Search Complexity

| Query Mode | Time Complexity | Space Complexity | Use Case |
|------------|----------------|------------------|----------|
| **Naive** | O(log N) | O(K) | Simple facts |
| **Local** | O(D × E) | O(D × N) | Entity-focused |
| **Global** | O(N + E) | O(N) | Complex reasoning |
| **Hybrid** | Adaptive | Adaptive | General purpose |

где:
- N = количество nodes
- E = количество edges
- D = depth (глубина traversal)
- K = top_k results

## Integration with Pipeline

```
Document Ingestion → Chunking → Entity Extraction → Graph Construction
                                                            ↓
                                                    [Knowledge Graph]
                                                            ↓
                                            Query → Search → Answer
                                                    ↑
                                            [Graph Structure]
                                            influences both
                                            search & answer
```

## Best Practices

### Node Design
- ✅ Normalize entity names (Title Case)
- ✅ Store rich metadata for each node
- ✅ Maintain source attribution (chunk_id, file_path)
- ✅ Update descriptions incrementally

### Relationship Design
- ✅ Use undirected edges for symmetric relations
- ✅ Store relationship keywords for fast filtering
- ✅ Include confidence scores
- ✅ Track relationship provenance

### Search Optimization
- ✅ Index frequently queried entity types
- ✅ Precompute PageRank for ranking
- ✅ Cache subgraphs for common queries
- ✅ Use approximate search for large graphs

### Answer Quality
- ✅ Combine multiple evidence sources
- ✅ Include source citations
- ✅ Show reasoning paths
- ✅ Rank by confidence scores

## Порядок Чтения

### Для понимания структуры:
1. [01-node-types.md](01-node-types.md) - типы узлов
2. [02-relationship-types.md](02-relationship-types.md) - типы связей
3. [03-graph-topology.md](03-graph-topology.md) - топология

### Для реализации поиска:
1. [03-graph-topology.md](03-graph-topology.md) - структура
2. [04-search-patterns.md](04-search-patterns.md) - паттерны поиска

### Для генерации ответов:
1. [04-search-patterns.md](04-search-patterns.md) - поиск
2. [05-answer-generation.md](05-answer-generation.md) - генерация ответов

## Code References

- **Node Operations**: `lightrag/operate.py` - `merge_nodes_and_edges()`
- **Graph Storage**: `lightrag/base.py` - `BaseGraphStorage`
- **Search Algorithms**: `lightrag/lightrag.py` - `aquery()`
- **Entity Extraction**: `lightrag/operate.py` - `extract_entities()`

## Визуализация

Для визуализации графа используйте:
- **NetworkX**: Для анализа и визуализации (Python)
- **Neo4j Browser**: Для интерактивного исследования
- **Gephi**: Для продвинутой визуализации
- **Cytoscape**: Для биологических/научных графов

## Расширение

Для добавления новых типов узлов или связей:
1. Обновите `entity_types` в конфигурации
2. Добавьте специфичные промпты для extraction
3. Обновите graph schema в storage backend
4. Добавьте специфичную логику в search patterns

---

**Версия**: 1.0
**Дата**: 2025-01-12
**Связано**: [Main Specification](../README.md)
