# Graph Algorithms: Алгоритмы Работы с Графом Знаний

## Обзор

Документация концептуальных алгоритмов на графах, применяемых в LightRAG для навигации, анализа и извлечения информации из Knowledge Graph.

## Категории Алгоритмов

### 1. Traversal Algorithms (Алгоритмы Обхода)

**Файл**: [01-bfs-traversal.md](01-bfs-traversal.md)

Семейство BFS (Breadth-First Search) алгоритмов для извлечения подграфов:
- Standard BFS - базовый обход в ширину
- Bidirectional BFS - оптимизированный двунаправленный поиск
- In/Out-bound BFS - специализированный MongoDB `$graphLookup`

**Роль**: Реализация **Local и Global traversal methods** из семантической парадигмы.

### 2. Community Detection (Обнаружение Сообществ)

**Файл**: [02-community-detection.md](02-community-detection.md)

Louvain algorithm для кластеризации entities:
- Автоматическое обнаружение semantic communities
- Иерархическая структура групп entities
- Оптимизация modularity

**Роль**: Реализация **star-attractor pattern** с автоматической идентификацией semantic clusters.

### 3. Centrality Metrics (Метрики Центральности)

**Файл**: [03-centrality-algorithms.md](03-centrality-algorithms.md)

Алгоритмы для вычисления важности entities:
- Degree Centrality - локальная важность (число связей)
- Betweenness Centrality - посредническая важность (bridge nodes)
- PageRank - глобальная важность (recursive importance)
- Personalized PageRank - query-specific importance

**Роль**: Определение **semantic mass** аттракторов в звездной архитектуре.

### 4. Entity Merging (Слияние Сущностей)

**Файл**: [04-entity-merging.md](04-entity-merging.md)

Алгоритмы объединения дубликатных entities:
- Merge strategies: concatenate, keep_first, keep_last, join_unique
- Graph topology preservation
- Source tracking

**Роль**: Entity resolution и дедупликация в **concept-manifestation bridge**.

### 5. Degree-Based Operations (Операции на Основе Степени)

**Файл**: [05-degree-operations.md](05-degree-operations.md)

Алгоритмы, использующие степень узлов:
- Node degree calculation
- Edge degree calculation (sum of endpoint degrees)
- Popular labels ranking by degree

**Роль**: Быстрая оценка **semantic gravity** entities без полного centrality расчета.

### 6. Graph Layout Algorithms (Алгоритмы Визуализации)

**Файл**: [06-layout-algorithms.md](06-layout-algorithms.md)

Алгоритмы для визуального представления графа:
- Spring Layout (Fruchterman-Reingold) - force-directed
- Circular Layout - entities по окружности
- Shell Layout - community-based shells

**Роль**: Визуализация **star-attractor constellation** для человеческого анализа.

---

## Концептуальные Связи

### Связь с Research Concepts

| Алгоритм | Research Concept | Описание |
|----------|------------------|----------|
| **BFS Traversal** | [Semantic Traversal Methods](../research/04-semantic-traversal-methods.md) | Local/Global modes реализуются через BFS с разными depth limits |
| **Community Detection** | [Star Attractor Pattern](../research/02-star-attractor-pattern.md) | Автоматическая идентификация semantic clusters (созвездий аттракторов) |
| **PageRank** | [Star Attractor Pattern](../research/02-star-attractor-pattern.md) | Вычисление semantic mass entities через recursive importance |
| **Entity Merging** | [Concept-Manifestation Bridge](../research/05-concept-manifestation-bridge.md) | Entity resolution объединяет разные manifestations одного концепта |
| **Degree Operations** | [Dual-Space Architecture](../research/03-dual-space-architecture.md) | Bridge между graph space (degree) и vector space (similarity) |

### Связь с Dependencies

| Алгоритм | Library | Функция |
|----------|---------|---------|
| **BFS** | Custom (MongoDB/PostgreSQL/NetworkX) | `_bidirectional_bfs_nodes`, `_bfs_subgraph`, `nx.bfs_edges` |
| **Community Detection** | [python-louvain](../dependencies/03-graph-processing.md) | `community.best_partition(graph)` |
| **Centrality** | [networkx](../dependencies/03-graph-processing.md) | `nx.degree_centrality`, `nx.betweenness_centrality`, `nx.pagerank` |
| **Layout** | [networkx](../dependencies/03-graph-processing.md) | `nx.spring_layout`, `nx.circular_layout`, `nx.shell_layout` |
| **Merging** | Custom (lightrag.utils_graph) | `amerge_entities` |

---

## Архитектурный Паттерн

```
┌─────────────────────────────────────────────────────────────┐
│             QUERY (Semantic Key)                            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓ [Vector Search]
              ┌──────────────┐
              │Top-K Entities│ ← Degree/PageRank filter
              │ (Attractors) │
              └──────┬───────┘
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
     [BFS Depth=1] [BFS Depth=2] [Community Detection]
          │          │          │
          ↓          ↓          ↓
    ┌─────────────────────────────┐
    │   Subgraph (Local/Global)   │ ← Centrality ranking
    └─────────────────────────────┘
                     │
                     ↓ [Entity Merging]
              ┌──────────────┐
              │  Final Graph │
              │   + Chunks   │
              └──────────────┘
                     │
                     ↓ [LLM]
                  ANSWER
```

**Философия**: Алгоритмы реализуют концепцию **"query as semantic key"** → **attractor activation** → **ray expansion** (BFS) → **context assembly**.

---

## Производительность

| Алгоритм | Complexity | Scale | Use Case |
|----------|-----------|-------|----------|
| **BFS (depth=2)** | O(V + E) | < 100K nodes | Local mode (entity-centric) |
| **Bidirectional BFS** | O(V + E) | < 1M nodes | Optimization for deep queries |
| **Louvain** | O(E) | < 10M edges | Community detection offline |
| **Degree Centrality** | O(V + E) | < 10M nodes | Fast online ranking |
| **PageRank** | O(V * iterations) | < 1M nodes | Offline pre-computation |
| **Entity Merging** | O(E_merged) | < 10K merges | Insert-time deduplication |

---

**Версия**: 1.0
**Дата**: 2025-01-13
