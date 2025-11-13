# Query as Semantic Key: Вопрос как Ключ в Семантическом Пространстве

## Концептуальная Парадигма

В традиционных поисковых системах **query** - это текстовая строка для сопоставления (string matching). В LightRAG query функционирует как **многомерный семантический ключ** (multi-dimensional semantic key), который:

1. **Проецируется** в несколько семантических пространств одновременно
2. **Активирует** семантические аттракторы через similarity matching
3. **Инициирует** структурный траверс по графу знаний
4. **Связывает** абстрактные концепции с их конкретными проявлениями

## Архитектура Ключа

### Уровень 1: Текстовое Представление (Surface Form)

```python
query = "What innovative products does Apple make?"
# Это поверхностная форма - natural language expression намерения
```

### Уровень 2: Символическая Декомпозиция (Symbolic Decomposition)

Query декомпозируется на **иерархические ключевые компоненты**:

```python
# Transformation T7: Keyword Extraction
keywords = extract_keywords(query)  # Using P5 prompt

keywords = {
    "high_level_keywords": [
        "Apple Inc",           # Abstract concept (organization)
        "products",            # Category concept
        "innovation"           # Quality concept
    ],
    "low_level_keywords": [
        "iPhone",              # Specific instance
        "iPad",                # Specific instance
        "Mac",                 # Specific instance
        "user experience",     # Specific attribute
        "design"               # Specific attribute
    ]
}
```

**Semantic Hierarchy**:
```
High-Level Keywords (Аттракторы верхнего уровня)
    → Абстрактные концепции, широкие темы
    → Используются для GLOBAL search (relationships_vdb)
    → Активируют центральные узлы-аттракторы

Low-Level Keywords (Аттракторы нижнего уровня)
    → Конкретные сущности, специфические детали
    → Используются для LOCAL search (entities_vdb)
    → Активируют периферийные узлы
```

### Уровень 3: Векторное Представление (Vector Projection)

Keywords проецируются в **continuous semantic space**:

```python
# Entity Resolution (T8)
query_embedding = await embedding_func(
    " ".join(keywords["high_level_keywords"])
)

# Result: numpy array, shape=(768,) or (1536,) depending on model
# query_embedding = [0.234, -0.567, 0.123, ..., 0.891]
```

**Свойства векторного ключа**:
- **Dimensionality**: 768 (BERT), 1536 (OpenAI ada-002), 4096 (Voyage)
- **Metric**: Cosine similarity для semantic proximity
- **Semantic gradients**: Близкие векторы = семантически похожие концепции

### Уровень 4: Структурная Идентификация (Structural Identification)

Vector key маппится на **discrete graph nodes**:

```python
# Transformation T8: Entity Resolution
entity_results = await entities_vdb.query(
    query_text=" ".join(keywords["high_level_keywords"]),
    query_embedding=query_embedding,
    top_k=10
)

# Returns seed entities (structural identifiers)
seed_entities = [
    {
        "entity_name": "Apple Inc",
        "entity_type": "organization",
        "score": 0.92,            # Cosine similarity
        "node_id": "entity_42"    # Graph node ID
    },
    {
        "entity_name": "iPhone",
        "entity_type": "product",
        "score": 0.87,
        "node_id": "entity_157"
    },
    ...
]
```

## Концептуальные Стадии Работы Ключа

### Стадия 1: Intent Decomposition (Декомпозиция Намерения)

```
Natural Language Query
    ↓
[LLM Analysis via P5]
    ↓
Hierarchical Intent Structure:
    ┌─────────────────────────────┐
    │  High-Level Intent          │  ← What domains/themes?
    │  "Apple", "products"        │
    └─────────────────────────────┘
              ↓
    ┌─────────────────────────────┐
    │  Low-Level Intent           │  ← What specifics?
    │  "iPhone", "innovation"     │
    └─────────────────────────────┘
```

**Код реализации** (lightrag/operate.py:2478):
```python
async def extract_keywords_only(
    text: str,
    param: QueryParam,
    global_config: dict[str, str],
    hashing_kv: BaseKVStorage | None = None,
) -> tuple[list[str], list[str]]:
    """
    Extract high-level and low-level keywords using LLM.

    This is the INTENT DECOMPOSITION stage - breaking down
    the user's semantic intent into hierarchical components.
    """

    # Build keyword extraction prompt
    kw_prompt = PROMPTS["keywords_extraction"].format(
        query=text,
        examples=examples,
        language=language,
    )

    # LLM performs semantic decomposition
    result = await use_model_func(kw_prompt, keyword_extraction=True)
    keywords_data = json_repair.loads(result)

    hl_keywords = keywords_data.get("high_level_keywords", [])
    ll_keywords = keywords_data.get("low_level_keywords", [])

    return hl_keywords, ll_keywords
```

### Стадия 2: Semantic Projection (Семантическая Проекция)

```
Symbolic Keywords
    ↓
[Embedding Model]
    ↓
Vector Space Projection:
    ┌──────────────────────────────────────┐
    │  768-dimensional semantic space     │
    │                                      │
    │  query_vector = [0.23, -0.56, ...]  │
    │                                      │
    │  Encodes semantic meaning as        │
    │  distributed representation         │
    └──────────────────────────────────────┘
```

**Свойства проекции**:
- **Lossless for retrieval**: Векторное представление сохраняет семантику для similarity search
- **Context-aware**: Embeddings учитывают контекст keywords
- **Multi-lingual**: Работает для разных языков в едином semantic space

### Стадия 3: Attractor Activation (Активация Аттракторов)

```
Query Vector (probe в semantic space)
    ↓
[Vector Similarity Search]
    ↓
Activated Attractors:
    ┌─────────────────────────────────────┐
    │  Nearest Entities (by cosine sim)  │
    │                                     │
    │  1. "Apple Inc" (0.92)             │ ← Primary attractor
    │  2. "iPhone" (0.87)                │ ← Secondary attractor
    │  3. "Steve Jobs" (0.84)            │ ← Related attractor
    │  ...                                │
    │  10. "Innovation" (0.75)           │ ← Peripheral attractor
    └─────────────────────────────────────┘
```

**Код реализации** (lightrag/operate.py:3651):
```python
async def _get_node_data(
    keywords: list[str],
    knowledge_graph_inst: BaseGraphStorage,
    entities_vdb: BaseVectorStorage,
    query_param: QueryParam,
):
    """
    ATTRACTOR ACTIVATION: Use keywords as semantic probe
    to activate nearest entities in vector space.
    """

    # Search entities by semantic similarity
    results = await entities_vdb.query(
        keywords,
        top_k=query_param.top_k
    )

    # results = activated attractors (seed entities)
    # These will become centers for subgraph expansion

    return entities, relations
```

### Стадия 4: Structural Traversal (Структурный Траверс)

```
Activated Seed Entities (attractors)
    ↓
[Graph Traversal BFS]
    ↓
Expanded Subgraph:
    ┌────────────────────────────────────────┐
    │  Entity Graph around Attractors        │
    │                                        │
    │     [Steve Jobs]                       │
    │           ↓ founded                    │
    │     [Apple Inc] ← seed attractor       │
    │       ↙    ↓    ↘                      │
    │ produces produces produces             │
    │    ↓       ↓       ↓                   │
    │ [iPhone] [iPad] [Mac]                  │
    │                                        │
    │  Depth: 1-2 hops (local)               │
    │  or 3+ hops (global)                   │
    └────────────────────────────────────────┘
```

### Стадия 5: Context Materialization (Материализация Контекста)

```
Expanded Subgraph (abstract concepts)
    ↓
[Chunk Retrieval via source_id]
    ↓
Grounded Text Context:
    ┌────────────────────────────────────────┐
    │  Concrete Manifestations               │
    │                                        │
    │  Chunk 1: "Apple Inc, founded by      │
    │            Steve Jobs in 1976..."      │
    │                                        │
    │  Chunk 2: "The iPhone revolutionized  │
    │            mobile computing..."        │
    │                                        │
    │  Chunk 3: "iPad introduced in 2010..." │
    └────────────────────────────────────────┘
```

## Ключ как Мост между Пространствами

Query key функционирует как **bridge operator** между множественными семантическими пространствами:

### Bridge 1: Natural Language ↔ Symbolic

```
"What products does Apple make?"
    ↕ [Keywords Extraction]
["Apple Inc", "products"] + ["iPhone", "iPad"]
```

### Bridge 2: Symbolic ↔ Vector Space

```
["Apple Inc", "products"]
    ↕ [Embedding]
[0.234, -0.567, 0.123, ..., 0.891]  (768-dim vector)
```

### Bridge 3: Vector Space ↔ Graph Space

```
[0.234, -0.567, ...]  (continuous)
    ↕ [Similarity Search + Entity Resolution]
{entity_42, entity_157, entity_203}  (discrete node IDs)
```

### Bridge 4: Graph Space ↔ Text Space

```
{entity_42: "Apple Inc"}
    ↕ [Source ID Lookup]
{chunk_12, chunk_45, chunk_78}  (text manifestations)
```

## Математическая Формализация

### Query Key Function

```python
Q: ℝ^n → 𝒫(G)

где:
- ℝ^n = n-dimensional vector space (semantic embeddings)
- 𝒫(G) = power set of graph G (subsets of entities/relations)
- Q = query key operator
```

### Decomposition

```
Q(q) = (K_high(q), K_low(q), E(q), R(q), S(q))

где:
- q = natural language query
- K_high = high-level keywords extraction
- K_low = low-level keywords extraction
- E = embedding projection
- R = entity resolution (attractor activation)
- S = subgraph extraction (structural traversal)
```

### Semantic Distance

```
sim(q, e) = cosine(E(K(q)), embed(e))

где:
- q = query
- e = entity
- E(K(q)) = embedding of query keywords
- embed(e) = entity embedding
- sim ∈ [0, 1] = semantic similarity score
```

### Attractor Activation Threshold

```
A_θ(q) = {e ∈ Entities | sim(q, e) ≥ θ}

где:
- A_θ = set of activated attractors
- θ = cosine threshold (e.g., 0.2)
- Entities = all entities in graph
```

## Концептуальные Свойства Ключа

### Свойство 1: Multi-scale Hierarchy

Query key работает на множественных уровнях абстракции:

```
Abstraction Scale:
High ←─────────────────────────────────→ Low

[Themes]  [Concepts]  [Entities]  [Instances]  [Attributes]
   ↓          ↓           ↓            ↓            ↓
"products" "Apple"   "iPhone"     "iPhone 15"  "design"

High-level keywords ────────┘  └──── Low-level keywords
```

### Свойство 2: Soft Matching через Vector Space

В отличие от exact string matching, query key использует **soft semantic matching**:

```
Query: "What does Apple produce?"
         ↓ [Embedding]
      [vector_q]
         ↓ [Cosine similarity]

Matches:
• "Apple Inc" (0.92) ← Exact entity match
• "Apple products" (0.89) ← Semantic variation
• "Tim Cook" (0.78) ← Associated entity (CEO of Apple)
• "iPhone" (0.87) ← Products of Apple
```

Это позволяет находить:
- Synonyms: "produce" → "manufacture", "make", "create"
- Related concepts: "Apple" → "iPhone", "iPad", "Mac"
- Contextual associations: "Apple" → "Steve Jobs", "Tim Cook"

### Свойство 3: Adaptive Expansion Radius

Query key определяет **radius of expansion** в graph space:

```python
def get_expansion_depth(mode, keywords):
    """
    Query key адаптивно определяет глубину траверса
    в зависимости от mode и характера keywords.
    """
    if mode == "local":
        return 1-2  # Limited expansion from activated attractors
    elif mode == "global":
        return 3+   # Extensive expansion, community detection
    elif mode == "naive":
        return 0    # No graph traversal, vector-only
```

### Свойство 4: Concept-Instance Bridging

Query key **связывает уровни абстракции**:

```
Abstraction Hierarchy:

    [Concept Level]
    "What products does Apple make?"
         ↓ [High-level keywords]
    "Apple", "products" ← Abstract concepts
         ↓ [Entity resolution]
    Entity: "Apple Inc" (organization)
         ↓ [Graph traversal via "produces" relation]
    Entities: "iPhone", "iPad", "Mac"

    [Instance Level]
         ↓ [Chunk retrieval]
    Chunk: "The iPhone 15 features A17 Pro chip..."
    Chunk: "iPad Air with M2 processor..."
```

## Практические Импликации

### Оптимизация 1: Keyword Quality

**Качество keywords напрямую влияет на эффективность ключа**:

```python
# Good keywords (high semantic relevance)
keywords = {
    "high_level": ["Apple Inc", "products"],
    "low_level": ["iPhone", "iPad", "innovation"]
}
# → Activate relevant attractors (precision: 0.90)

# Poor keywords (low semantic relevance)
keywords = {
    "high_level": ["company", "items"],
    "low_level": ["things", "stuff"]
}
# → Activate irrelevant attractors (precision: 0.40)
```

**Улучшение**: Использовать better LLM prompts (P5), examples, fine-tuning.

### Оптимизация 2: Embedding Model Selection

**Размерность и качество embeddings критичны**:

```python
# High-quality embeddings (e.g., Voyage-2, 1024-dim)
cosine_sim("Apple products", "iPhone") = 0.87
# → Strong semantic connection

# Low-quality embeddings (e.g., old Word2Vec, 300-dim)
cosine_sim("Apple products", "iPhone") = 0.62
# → Weak semantic connection
```

**Улучшение**: Использовать modern embedding models (OpenAI ada-002, Voyage, Cohere).

### Оптимизация 3: Threshold Tuning

**Cosine threshold определяет precision/recall trade-off**:

```python
# High threshold (θ = 0.4)
activated_entities = 3  # High precision, low recall
# Only very relevant attractors activated

# Medium threshold (θ = 0.2) ← Default
activated_entities = 10  # Balanced

# Low threshold (θ = 0.1)
activated_entities = 50  # Low precision, high recall
# Many irrelevant attractors activated
```

### Оптимизация 4: Caching Query Keys

**Query embeddings можно кэшировать**:

```python
# Cache structure
query_key_cache = {
    "query_hash": {
        "keywords": ["Apple", "products"],
        "embedding": [0.234, -0.567, ...],
        "seed_entities": [42, 157, 203],
        "timestamp": 1704067200
    }
}

# On similar query, reuse cached key
if query_hash in cache:
    seed_entities = cache[query_hash]["seed_entities"]
    # Skip T7 (keywords) and T8 (resolution)
    # Directly proceed to T9 (context assembly)
```

## Связь с Трансформациями

Query as Semantic Key реализуется через последовательность трансформаций:

```
Query Text (natural language)
    ↓
[T7: Keywords Extraction] ← Creates symbolic key
    ↓
High/Low Keywords (hierarchical intent)
    ↓
[Embedding Function] ← Projects to vector space
    ↓
Query Embedding (vector key)
    ↓
[T8: Entity Resolution] ← Activates attractors
    ↓
Seed Entities (structural key)
    ↓
[T9: Context Assembly] ← Traverses graph
    ↓
Subgraph + Chunks (materialized context)
    ↓
[T10: Answer Generation] ← Synthesizes answer
```

**См. также**:
- [Semantic Transformations](../transform/README.md)
- [T7: Keywords Extraction](../transform/02-query-transforms.md#t7-keyword-extraction)
- [T8: Entity Resolution](../transform/02-query-transforms.md#t8-entity-resolution)

---

## Философское Осмысление

### Query как Семантический Зонд (Semantic Probe)

В физике **probe** - это инструмент для измерения состояния системы. Query функционирует как **semantic probe**, который:
- Вводится в семантическое пространство знаний
- Активирует ближайшие семантические аттракторы
- Измеряет "semantic field" вокруг аттракторов
- Возвращает materialized information

### Ключ как Оператор Проекции

Query key = projection operator между пространствами:

```
𝒬: Text → Keywords → Vectors → Entities → Subgraph → Answer

Каждая стрелка = projection/mapping между semantic spaces
```

### Иерархия Намерений

High/low keywords отражают **hierarchy of intentions**:
- **High-level** = "что я хочу узнать?" (epistemological level)
- **Low-level** = "о чем конкретно?" (ontological level)

---

**Версия**: 1.0
**Дата**: 2025-01-13
**См. также**: [Star Attractor Pattern](02-star-attractor-pattern.md), [Dual-Space Architecture](03-dual-space-architecture.md)
