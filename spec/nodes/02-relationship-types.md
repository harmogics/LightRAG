# Relationship Types: Типы Отношений в Knowledge Graph

## Обзор

Отношения (relationships/edges) в LightRAG Knowledge Graph представляют связи между узлами. Они делятся на три основные категории: **Structural** (структурные, системные), **Semantic** (семантические, извлеченные LLM) и **Derived** (производные, вычисляемые).

## Иерархия Типов Отношений

```
┌────────────────────────────────────────────────────────────┐
│                  ALL RELATIONSHIPS                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  STRUCTURAL (System-Generated)                      │ │
│  │  • contains (Document → Chunk)                      │ │
│  │  • extracted_from (Entity → Chunk)                  │ │
│  │  • cited_in (Entity → Document)                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  SEMANTIC (LLM-Extracted)                           │ │
│  │  ┌───────────────────────────────────────────────┐ │ │
│  │  │  Domain-Specific Relations                    │ │ │
│  │  │  • works_for, founded, manages                │ │ │
│  │  │  • located_in, headquarters_of                │ │ │
│  │  │  • produces, manufactures                     │ │ │
│  │  │  • participates_in, organized_by              │ │ │
│  │  └───────────────────────────────────────────────┘ │ │
│  │  ┌───────────────────────────────────────────────┐ │ │
│  │  │  Generic Relations                            │ │ │
│  │  │  • related_to                                 │ │ │
│  │  │  • associated_with                            │ │ │
│  │  │  • similar_to                                 │ │ │
│  │  └───────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  DERIVED (Computed)                                 │ │
│  │  • co_occurs_with (computed from chunks)            │ │
│  │  • shares_community (computed from clustering)      │ │
│  │  • connected_through (multi-hop path)               │ │
│  │  • temporally_related (time-based)                  │ │
│  └─────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

## 1. Structural Relationships

### 1.1 Contains (Document → Chunk)

**Назначение**: Связывает документ с его фрагментами

#### Schema

```python
ContainsRelation = {
    "source": "doc_id",         # Document node ID
    "target": "chunk_id",       # Chunk node ID
    "type": "contains",
    "attributes": {
        "order": int,           # Порядковый номер chunk в документе
        "start_pos": int,       # Начальная позиция в документе
        "end_pos": int          # Конечная позиция
    }
}
```

#### Properties

- **Направленность**: Directed (Document → Chunk)
- **Кардинальность**: One-to-Many (один документ → много chunks)
- **Создание**: Автоматически при chunking
- **Обновление**: Immutable (не изменяется после создания)

#### Example

```python
{
    "source": "doc-a3f2b9c1e5d8",
    "target": "chunk-b7e4d2f8a1c3",
    "type": "contains",
    "attributes": {
        "order": 0,
        "start_pos": 0,
        "end_pos": 1024
    }
}
```

#### Use Cases

- Навигация от документа к chunks
- Реконструкция полного документа
- Boundary определение для citations

---

### 1.2 Extracted_from (Entity → Chunk)

**Назначение**: Связывает entity с chunk, из которого она была извлечена

#### Schema

```python
ExtractedFromRelation = {
    "source": "entity_id",      # Entity node ID
    "target": "chunk_id",       # Chunk node ID
    "type": "extracted_from",
    "attributes": {
        "extraction_timestamp": int,
        "confidence": float,    # Confidence score (0-1)
        "method": str           # "llm_extraction", "manual", etc.
    }
}
```

#### Properties

- **Направленность**: Directed (Entity → Chunk)
- **Кардинальность**: Many-to-Many (entity может быть в многих chunks)
- **Создание**: Автоматически при entity extraction
- **Обновление**: Append-only (добавляются новые mentions)

#### Example

```python
{
    "source": "ent-c9f1e3a7b5d2",  # Tim Cook
    "target": "chunk-b7e4d2f8a1c3",
    "type": "extracted_from",
    "attributes": {
        "extraction_timestamp": 1705012346,
        "confidence": 0.95,
        "method": "llm_extraction"
    }
}
```

#### Use Cases

- Source attribution для entities
- Citation generation
- Evidence retrieval
- Provenance tracking

---

### 1.3 Cited_in (Entity → Document)

**Назначение**: Связывает entity с документом (высокоуровневая атрибуция)

#### Schema

```python
CitedInRelation = {
    "source": "entity_id",      # Entity node ID
    "target": "doc_id",         # Document node ID
    "type": "cited_in",
    "attributes": {
        "mention_count": int,   # Количество упоминаний в документе
        "first_mention": int,   # Позиция первого упоминания
        "importance": float     # Важность entity в документе (TF-IDF)
    }
}
```

#### Properties

- **Направленность**: Directed (Entity → Document)
- **Кардинальность**: Many-to-Many
- **Создание**: Автоматически при aggregation
- **Обновление**: Incremental (обновляется при новых mentions)

#### Use Cases

- Document-level entity index
- Document relevance scoring
- Entity importance ranking

---

## 2. Semantic Relationships

### 2.1 LLM-Extracted Relations (General)

**Назначение**: Отношения извлеченные LLM из текста

#### Base Schema

```python
SemanticRelation = {
    "source": "entity_id",          # Source entity
    "target": "entity_id",          # Target entity
    "type": str,                    # Relationship type (user-defined)
    "keywords": str,                # Comma-separated keywords
    "description": str,             # Relationship description
    "source_id": str,               # Chunk IDs where mentioned
    "file_path": str,               # Source file
    "created_at": int,              # First extraction timestamp
    "updated_at": int,              # Last update timestamp
    "attributes": {
        "confidence": float,        # Confidence score
        "bidirectional": bool,      # True for symmetric relations
        "weight": float,            # Strength of relationship (0-1)
        "temporal": dict            # Temporal info if applicable
    }
}
```

#### Properties

- **Направленность**: Mixed (configured per type)
- **Кардинальность**: Many-to-Many
- **Создание**: LLM extraction
- **Обновление**: LLM-based description merging

---

### 2.2 Domain-Specific Relations

#### 2.2.1 Works_for (Person → Organization)

```python
{
    "source": "ent-tim-cook",
    "target": "ent-apple-inc",
    "type": "works_for",
    "keywords": "employment, leadership, CEO",
    "description": "Tim Cook serves as the Chief Executive Officer of Apple Inc",
    "attributes": {
        "role": "CEO",
        "start_date": "2011-08-24",
        "employment_type": "full-time",
        "bidirectional": false
    }
}
```

**Common Patterns**:
- Person → Organization
- Variations: `employed_by`, `leads`, `manages`

---

#### 2.2.2 Located_in (Entity → Location)

```python
{
    "source": "ent-apple-inc",
    "target": "ent-cupertino",
    "type": "located_in",
    "keywords": "headquarters, location, based",
    "description": "Apple Inc is headquartered in Cupertino, California",
    "attributes": {
        "location_type": "headquarters",
        "since": "1997",
        "bidirectional": false
    }
}
```

**Common Patterns**:
- Organization → Location (headquarters)
- Person → Location (residence)
- Event → Location (venue)

---

#### 2.2.3 Produces (Organization → Product)

```python
{
    "source": "ent-apple-inc",
    "target": "ent-iphone",
    "type": "produces",
    "keywords": "manufacturing, product, design",
    "description": "Apple Inc designs and manufactures the iPhone smartphone line",
    "attributes": {
        "since": "2007",
        "production_status": "active",
        "bidirectional": false
    }
}
```

**Common Patterns**:
- Organization → Product
- Variations: `manufactures`, `develops`, `designs`

---

#### 2.2.4 Founded (Person → Organization)

```python
{
    "source": "ent-steve-jobs",
    "target": "ent-apple-inc",
    "type": "founded",
    "keywords": "founder, establishment, co-founder",
    "description": "Steve Jobs co-founded Apple Inc in 1976",
    "attributes": {
        "founding_date": "1976-04-01",
        "role": "co-founder",
        "with": ["Steve Wozniak", "Ronald Wayne"],
        "bidirectional": false
    }
}
```

---

#### 2.2.5 Participates_in (Entity → Event)

```python
{
    "source": "ent-tim-cook",
    "target": "ent-iphone15-launch",
    "type": "participates_in",
    "keywords": "speaker, presenter, keynote",
    "description": "Tim Cook presented the iPhone 15 at the launch event",
    "attributes": {
        "role": "keynote_speaker",
        "date": "2023-09-12",
        "bidirectional": false
    }
}
```

---

#### 2.2.6 Competes_with (Organization ↔ Organization)

```python
{
    "source": "ent-apple-inc",
    "target": "ent-samsung",
    "type": "competes_with",
    "keywords": "competition, rival, market",
    "description": "Apple Inc and Samsung compete in the smartphone market",
    "attributes": {
        "market": "smartphones",
        "intensity": "high",
        "bidirectional": true  # Symmetric relation
    }
}
```

---

#### 2.2.7 Uses_technology (Product → Concept)

```python
{
    "source": "ent-iphone",
    "target": "ent-machine-learning",
    "type": "uses_technology",
    "keywords": "technology, implementation, feature",
    "description": "iPhone utilizes machine learning for face recognition and photography",
    "attributes": {
        "features": ["Face ID", "computational photography"],
        "bidirectional": false
    }
}
```

---

### 2.3 Generic Relations

#### 2.3.1 Related_to (Entity ↔ Entity)

**Назначение**: Общая связь между entities без специфичной семантики

```python
{
    "source": "ent-entity1",
    "target": "ent-entity2",
    "type": "related_to",
    "keywords": "connection, association",
    "description": "General relationship between entities",
    "attributes": {
        "specificity": "low",
        "bidirectional": true
    }
}
```

**Use Cases**:
- Fallback для неклассифицированных relationships
- Слабые или неопределенные связи
- Placeholder для будущей классификации

---

#### 2.3.2 Similar_to (Entity ↔ Entity)

**Назначение**: Семантическое сходство между entities

```python
{
    "source": "ent-iphone",
    "target": "ent-galaxy",
    "type": "similar_to",
    "keywords": "similarity, comparable, equivalent",
    "description": "iPhone and Galaxy are similar smartphone products",
    "attributes": {
        "similarity_score": 0.85,
        "basis": "product_category",
        "bidirectional": true
    }
}
```

**Creation**:
- Вычисляется на основе embedding similarity
- LLM-detected similarities
- Clustering results

---

## 3. Derived Relationships

### 3.1 Co_occurs_with (Entity ↔ Entity)

**Назначение**: Entities часто встречающиеся вместе в тексте

#### Schema

```python
CoOccursRelation = {
    "source": "entity_id",
    "target": "entity_id",
    "type": "co_occurs_with",
    "attributes": {
        "co_occurrence_count": int,     # Количество совместных упоминаний
        "shared_chunks": list[str],     # Chunk IDs
        "pmi_score": float,             # Pointwise Mutual Information
        "window_size": int,             # Размер окна для подсчета
        "bidirectional": true
    }
}
```

#### Computation

```python
def compute_co_occurrence(entity1, entity2, window_size=3):
    """
    Подсчет co-occurrence в пределах окна chunks

    PMI = log(P(entity1, entity2) / (P(entity1) * P(entity2)))
    """

    # Получить chunks для обеих entities
    chunks1 = get_chunks_for_entity(entity1)
    chunks2 = get_chunks_for_entity(entity2)

    # Найти пересечения в пределах window
    co_occurrences = find_co_occurrences_in_window(
        chunks1, chunks2, window_size
    )

    # Вычислить PMI
    pmi = compute_pmi(entity1, entity2, co_occurrences)

    return {
        "co_occurrence_count": len(co_occurrences),
        "shared_chunks": co_occurrences,
        "pmi_score": pmi
    }
```

#### Use Cases

- Обнаружение implicit relationships
- Entity clustering
- Context expansion
- Recommendation systems

---

### 3.2 Shares_community (Entity ↔ Entity)

**Назначение**: Entities принадлежащие одному community в графе

#### Schema

```python
SharesCommunityRelation = {
    "source": "entity_id",
    "target": "entity_id",
    "type": "shares_community",
    "attributes": {
        "community_id": str,            # ID community
        "community_name": str,          # Название (опционально)
        "algorithm": str,               # "louvain", "label_propagation"
        "modularity": float,            # Modularity score
        "bidirectional": true
    }
}
```

#### Computation

```python
# Using Louvain algorithm
import community as community_louvain

communities = community_louvain.best_partition(graph)

# Create relations for entities in same community
for entity1, comm1 in communities.items():
    for entity2, comm2 in communities.items():
        if entity1 < entity2 and comm1 == comm2:
            create_relation(entity1, entity2, "shares_community", {
                "community_id": comm1,
                "algorithm": "louvain"
            })
```

#### Use Cases

- Thematic clustering
- Community-based search
- Entity grouping
- Topic modeling

---

### 3.3 Connected_through (Entity ⇝ Entity)

**Назначение**: Multi-hop connection через промежуточные entities

#### Schema

```python
ConnectedThroughRelation = {
    "source": "entity_id",
    "target": "entity_id",
    "type": "connected_through",
    "attributes": {
        "path": list[str],              # List of entity IDs in path
        "path_length": int,             # Hop count
        "path_type": list[str],         # Relation types in path
        "strength": float,              # Path strength (product of weights)
        "bidirectional": false
    }
}
```

#### Example

```python
# Path: Tim Cook → Apple Inc → iPhone
{
    "source": "ent-tim-cook",
    "target": "ent-iphone",
    "type": "connected_through",
    "attributes": {
        "path": ["ent-tim-cook", "ent-apple-inc", "ent-iphone"],
        "path_length": 2,
        "path_type": ["works_for", "produces"],
        "strength": 0.8
    }
}
```

#### Use Cases

- Multi-hop reasoning
- Relationship explanation
- Path-based search
- Inference generation

---

### 3.4 Temporally_related (Entity → Entity)

**Назначение**: Временная связь между entities (последовательность событий)

#### Schema

```python
TemporallyRelatedRelation = {
    "source": "entity_id",
    "target": "entity_id",
    "type": "temporally_related",
    "attributes": {
        "temporal_type": str,           # "before", "after", "during"
        "time_diff": int,               # Временная дельта (секунды)
        "confidence": float,
        "bidirectional": false
    }
}
```

#### Example

```python
{
    "source": "ent-iphone14-launch",
    "target": "ent-iphone15-launch",
    "type": "temporally_related",
    "attributes": {
        "temporal_type": "before",
        "time_diff": 31536000,  # 1 year in seconds
        "confidence": 1.0
    }
}
```

---

## Relationship Directionality

### Undirected Relations (Bidirectional)

```python
UNDIRECTED_TYPES = {
    "related_to",
    "similar_to",
    "competes_with",
    "partners_with",
    "co_occurs_with",
    "shares_community"
}
```

**Storage**: Сортированные edge keys для предотвращения дубликатов

```python
# Всегда сортируем для undirected relations
edge_key = tuple(sorted([source_entity, target_entity]))
```

### Directed Relations

```python
DIRECTED_TYPES = {
    "contains",
    "extracted_from",
    "works_for",
    "located_in",
    "produces",
    "founded",
    "uses_technology",
    "connected_through"
}
```

---

## Relationship Weights

### Weight Calculation

```python
def calculate_relationship_weight(relation):
    """
    Вычисление веса отношения на основе:
    1. Frequency: количество упоминаний
    2. Recency: свежесть данных
    3. Confidence: уверенность extraction
    4. Context: качество контекста
    """

    frequency_score = min(relation["mention_count"] / 10, 1.0)

    time_decay = math.exp(-(time.time() - relation["created_at"]) / (365 * 24 * 3600))

    confidence_score = relation.get("confidence", 0.5)

    # Weighted combination
    weight = (
        0.4 * frequency_score +
        0.3 * time_decay +
        0.3 * confidence_score
    )

    return weight
```

### Use Cases for Weights

- **Ranking**: Приоритизация relationships в search results
- **Pruning**: Удаление weak relationships для оптимизации
- **Scoring**: Path strength в multi-hop reasoning
- **Visualization**: Толщина ребер в graph visualization

---

## Relationship Operations

### Creation

```python
async def create_relationship(
    source_id: str,
    target_id: str,
    rel_type: str,
    attributes: dict
):
    """Create new relationship"""

    # Ensure both entities exist
    await ensure_entities_exist([source_id, target_id])

    # Check if undirected
    if rel_type in UNDIRECTED_TYPES:
        edge_key = tuple(sorted([source_id, target_id]))
    else:
        edge_key = (source_id, target_id)

    # Create relation data
    relation_data = {
        "source": source_id,
        "target": target_id,
        "type": rel_type,
        "created_at": int(time.time()),
        **attributes
    }

    # Store in graph DB
    await knowledge_graph_inst.upsert_edge(
        edge_key[0], edge_key[1], relation_data
    )

    # Store in vector DB for semantic search
    await relationships_vdb.upsert({
        compute_mdhash_id(f"{source_id}-{target_id}"): relation_data
    })
```

### Update (Merging)

```python
async def merge_relationship(
    source_id: str,
    target_id: str,
    new_attributes: dict
):
    """Merge new information into existing relationship"""

    # Get existing
    existing = await knowledge_graph_inst.get_edge(source_id, target_id)

    if not existing:
        return await create_relationship(source_id, target_id, new_attributes)

    # Merge descriptions using LLM
    all_descriptions = [
        existing["description"],
        new_attributes["description"]
    ]

    merged_description = await llm_summarize(all_descriptions)

    # Update attributes
    updated_data = {
        **existing,
        "description": merged_description,
        "updated_at": int(time.time()),
        "mention_count": existing.get("mention_count", 0) + 1
    }

    # Upsert
    await knowledge_graph_inst.upsert_edge(source_id, target_id, updated_data)
```

### Querying

```python
async def get_relationships_by_type(
    entity_id: str,
    rel_type: str,
    direction: str = "both"  # "outgoing", "incoming", "both"
):
    """Get all relationships of specific type for entity"""

    relations = []

    if direction in ["outgoing", "both"]:
        outgoing = await knowledge_graph_inst.get_outgoing_edges(
            entity_id, edge_type=rel_type
        )
        relations.extend(outgoing)

    if direction in ["incoming", "both"]:
        incoming = await knowledge_graph_inst.get_incoming_edges(
            entity_id, edge_type=rel_type
        )
        relations.extend(incoming)

    return relations
```

---

## Relationship Patterns

### Pattern 1: Hub Pattern (Star)

```
          ┌─────────┐
      ┌───│Entity 2 │
      │   └─────────┘
┌─────▼────┐
│  Hub     │───────┐
│  Entity  │       │
└─────┬────┘   ┌───▼─────┐
      │        │Entity 3 │
      │        └─────────┘
  ┌───▼─────┐
  │Entity 4 │
  └─────────┘
```

**Example**: Apple Inc → {iPhone, iPad, Mac, Tim Cook, Cupertino}

**Search Strategy**: Direct neighbors retrieval

---

### Pattern 2: Chain Pattern (Path)

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│Entity 1 │────>│Entity 2 │────>│Entity 3 │
└─────────┘     └─────────┘     └─────────┘
```

**Example**: Steve Jobs → Apple Inc → iPhone

**Search Strategy**: Path traversal (BFS/DFS)

---

### Pattern 3: Clique Pattern (Fully Connected)

```
    ┌─────────┐
 ┌──│Entity 1 │──┐
 │  └─────────┘  │
 │               │
 │  ┌─────────┐ │
 └──│Entity 2 │─┤
    └─────────┘ │
         │      │
    ┌────▼──────▼┐
    │  Entity 3  │
    └────────────┘
```

**Example**: {Tim Cook, Apple Inc, Cupertino} все связаны

**Search Strategy**: Community detection

---

## Performance Considerations

### Storage

| Relation Type | Avg Size | Vector | Total |
|---------------|----------|--------|-------|
| Structural | 0.2 KB | No | 0.2 KB |
| Semantic | 1-2 KB | Yes (1536d) | 7-8 KB |
| Derived | 0.5 KB | No | 0.5 KB |

### Query Performance

| Operation | Complexity | Time |
|-----------|-----------|------|
| Get Edge | O(1) | <1ms |
| Get Neighbors | O(D) | 1-10ms |
| Path Finding | O(E × D) | 10-100ms |
| Subgraph | O(N + E) | 50-500ms |

---

## Best Practices

### ✅ DO

1. **Use specific relation types** when possible
2. **Store provenance** (source_id, confidence)
3. **Include temporal info** when available
4. **Compute weights** for ranking
5. **Normalize directionality** for undirected relations

### ❌ DON'T

1. **Don't create duplicate relations** (check before insert)
2. **Don't ignore bidirectionality** semantics
3. **Don't store weak relations** (<0.3 weight)
4. **Don't skip description merging** on updates

---

**Следующий раздел**: [03-graph-topology.md](03-graph-topology.md) - Топология и структура графа
