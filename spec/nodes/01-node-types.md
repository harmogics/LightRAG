# Node Types: Типы Узлов в Knowledge Graph

## Обзор

LightRAG Knowledge Graph содержит три основных категории узлов: **Metadata Nodes** (инфраструктурные), **Content Nodes** (контентные) и **Knowledge Nodes** (семантические). Каждый тип узла имеет свою роль в системе и специфичные атрибуты.

## Иерархия Типов

```
┌─────────────────────────────────────────────────────────┐
│                   ALL NODES                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌────────────────────────────────────────────────┐   │
│  │  METADATA NODES (Infrastructure Layer)         │   │
│  │  ┌──────────────┐      ┌──────────────┐       │   │
│  │  │  Document    │      │    Chunk     │       │   │
│  │  │    Node      │      │    Node      │       │   │
│  │  └──────────────┘      └──────────────┘       │   │
│  └────────────────────────────────────────────────┘   │
│                          │                             │
│                          v                             │
│  ┌────────────────────────────────────────────────┐   │
│  │  KNOWLEDGE NODES (Semantic Layer)              │   │
│  │  ┌──────────────────────────────────────────┐ │   │
│  │  │         Entity Node                      │ │   │
│  │  │  ┌────────────────────────────────────┐ │ │   │
│  │  │  │ • Person                           │ │ │   │
│  │  │  │ • Organization                     │ │ │   │
│  │  │  │ • Location                         │ │ │   │
│  │  │  │ • Event                            │ │ │   │
│  │  │  │ • Product                          │ │ │   │
│  │  │  │ • Concept                          │ │ │   │
│  │  │  │ • Category                         │ │ │   │
│  │  │  │ • Other                            │ │ │   │
│  │  │  └────────────────────────────────────┘ │ │   │
│  │  └──────────────────────────────────────────┘ │   │
│  └────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 1. Metadata Nodes

### 1.1 Document Node

**Назначение**: Представление исходного документа в системе

#### Schema

```python
DocumentNode = {
    "doc_id": str,              # Уникальный ID: "doc-{md5_hash}"
    "content": str,             # Полное содержимое документа
    "file_path": str,           # Путь к исходному файлу
    "created_at": int,          # Unix timestamp создания
    "metadata": {
        "title": str,           # Название документа (опционально)
        "author": str,          # Автор (опционально)
        "date": str,            # Дата документа (опционально)
        "tags": list[str],      # Теги (опционально)
        "language": str,        # Язык документа
        "source": str           # Источник (URL, путь)
    }
}
```

#### Пример

```json
{
    "doc_id": "doc-a3f2b9c1e5d8",
    "content": "Apple Inc. is a multinational technology company...",
    "file_path": "/documents/apple_overview.pdf",
    "created_at": 1705012345,
    "metadata": {
        "title": "Apple Inc: Company Overview",
        "author": "Tech Analyst Team",
        "date": "2024-01-10",
        "tags": ["technology", "company profile"],
        "language": "English",
        "source": "internal_research"
    }
}
```

#### Storage Locations

- **Primary**: `full_docs` (Key-Value Storage)
- **Index**: `doc_status` (Document Status Storage)

#### Relations

- **Outgoing**: `contains` → Chunk Nodes
- **Incoming**: `cited_in` ← Entity Nodes (reverse lookup)

#### Use Cases

- Трассировка к исходному документу
- Версионирование и обновление
- Access control и permissions
- Audit trail

---

### 1.2 Chunk Node

**Назначение**: Представление текстового фрагмента документа

#### Schema

```python
ChunkNode = {
    "chunk_id": str,            # Уникальный ID: "chunk-{md5_hash}"
    "content": str,             # Текст фрагмента
    "tokens": int,              # Количество токенов
    "chunk_order_index": int,   # Порядковый номер в документе (0-based)
    "full_doc_id": str,         # ID родительского документа
    "file_path": str,           # Путь к исходному файлу
    "created_at": int,          # Unix timestamp
    "embedding": list[float],   # Vector representation (хранится в VDB)
    "metadata": {
        "start_char": int,      # Начальная позиция в документе
        "end_char": int,        # Конечная позиция
        "section": str          # Секция документа (опционально)
    }
}
```

#### Пример

```json
{
    "chunk_id": "chunk-b7e4d2f8a1c3",
    "content": "Apple Inc is headquartered in Cupertino, California. The company was founded in 1976 by Steve Jobs, Steve Wozniak, and Ronald Wayne.",
    "tokens": 32,
    "chunk_order_index": 0,
    "full_doc_id": "doc-a3f2b9c1e5d8",
    "file_path": "/documents/apple_overview.pdf",
    "created_at": 1705012346,
    "metadata": {
        "start_char": 0,
        "end_char": 145,
        "section": "Introduction"
    }
}
```

#### Storage Locations

- **Primary**: `text_chunks` (Key-Value Storage)
- **Vector**: `chunks_vdb` (Vector Database)

#### Relations

- **Incoming**: `contains` ← Document Node
- **Outgoing**: `extracted_from` → Entity Nodes

#### Vector Representation

```python
# Embedding content = chunk text
embedding = embedding_func(chunk["content"])

# Используется для:
# - Semantic search по chunks (Naive RAG)
# - Context expansion
# - Similar chunk retrieval
```

#### Use Cases

- Semantic search (Naive mode)
- Citation и source attribution
- Context window для LLM
- Evidence retrieval

---

## 2. Knowledge Nodes

### 2.1 Entity Node (Base)

**Назначение**: Представление извлеченной сущности (entity)

#### Base Schema

```python
EntityNode = {
    "entity_id": str,           # Уникальный ID: "ent-{md5_hash}"
    "entity_name": str,         # Нормализованное имя (Title Case)
    "entity_type": str,         # Тип сущности
    "description": str,         # Comprehensive описание
    "source_id": str,           # CSV chunk IDs: "chunk-1,chunk-2,..."
    "file_path": str,           # Путь к исходному файлу
    "created_at": int,          # Unix timestamp первого упоминания
    "updated_at": int,          # Unix timestamp последнего обновления
    "embedding": list[float],   # Vector representation (хранится в VDB)
    "metadata": {
        "aliases": list[str],   # Альтернативные имена
        "confidence": float,    # Confidence score (0-1)
        "mention_count": int,   # Количество упоминаний
        "importance": float     # Importance score (PageRank, etc.)
    }
}
```

#### Storage Locations

- **Primary**: `full_entities_storage` (Key-Value Storage)
- **Graph**: `knowledge_graph_inst` (Graph Database)
- **Vector**: `entity_vdb` (Vector Database)

#### Vector Representation

```python
# Embedding content = entity_name + description
content = f"{entity_name}\n{description}"
embedding = embedding_func(content)

# Используется для:
# - Semantic entity search
# - Entity disambiguation
# - Concept similarity
```

---

### 2.2 Person Node

**Назначение**: Представление человека (личности)

#### Extended Schema

```python
PersonNode = EntityNode + {
    "entity_type": "person",
    "person_metadata": {
        "full_name": str,           # Полное имя
        "roles": list[str],         # Роли: ["CEO", "Founder", ...]
        "organizations": list[str], # Связанные организации
        "locations": list[str],     # Связанные локации
        "birth_year": int,          # Год рождения (если известен)
        "nationality": str          # Национальность
    }
}
```

#### Пример

```json
{
    "entity_id": "ent-c9f1e3a7b5d2",
    "entity_name": "Tim Cook",
    "entity_type": "person",
    "description": "Tim Cook is the Chief Executive Officer of Apple Inc, leading the company since August 2011. He previously served as Chief Operating Officer and has been instrumental in Apple's supply chain management and operational excellence.",
    "source_id": "chunk-b7e4d2f8a1c3,chunk-d3e8f1a9c2b5",
    "file_path": "/documents/apple_overview.pdf",
    "created_at": 1705012346,
    "person_metadata": {
        "full_name": "Timothy Donald Cook",
        "roles": ["CEO", "Board Member"],
        "organizations": ["Apple Inc"],
        "locations": ["Cupertino"],
        "nationality": "American"
    }
}
```

#### Common Relations

- `works_for` → Organization
- `founded` → Organization
- `located_in` → Location
- `participated_in` → Event
- `educated_at` → Organization (university)
- `knows` ↔ Person

#### Search Patterns

```python
# Finding people by role
query = "Who is the CEO of Apple?"
→ Search: entity_type="person" AND description contains "CEO" AND related to "Apple"

# Finding people in organization
query = "Who works at Google?"
→ Traverse: Organization("Google") -[works_for]- Person

# Finding people in location
query = "Who are the tech leaders in Silicon Valley?"
→ Traverse: Location("Silicon Valley") -[located_in]- Person
           Filter: roles contains "CEO" or "Founder"
```

---

### 2.3 Organization Node

**Назначение**: Представление организации (компании, учреждения)

#### Extended Schema

```python
OrganizationNode = EntityNode + {
    "entity_type": "organization",
    "org_metadata": {
        "org_type": str,            # "company", "university", "government"
        "industry": str,            # Индустрия
        "founded_year": int,        # Год основания
        "headquarters": str,        # Расположение штаб-квартиры
        "products": list[str],      # Продукты/услуги
        "key_people": list[str],    # Ключевые люди
        "website": str              # Веб-сайт
    }
}
```

#### Пример

```json
{
    "entity_id": "ent-e2b5c8d1f3a6",
    "entity_name": "Apple Inc",
    "entity_type": "organization",
    "description": "Apple Inc is a multinational technology company that designs, develops, and sells consumer electronics, computer software, and online services. Founded in 1976, Apple is known for innovative products like iPhone, iPad, and Mac computers.",
    "source_id": "chunk-b7e4d2f8a1c3,chunk-a1b2c3d4e5f6",
    "org_metadata": {
        "org_type": "company",
        "industry": "Technology",
        "founded_year": 1976,
        "headquarters": "Cupertino, California",
        "products": ["iPhone", "iPad", "Mac", "Apple Watch"],
        "key_people": ["Tim Cook", "Steve Jobs"],
        "website": "https://www.apple.com"
    }
}
```

#### Common Relations

- `located_in` → Location
- `produces` → Product
- `founded_by` ← Person
- `employs` ← Person
- `competes_with` ↔ Organization
- `acquired` → Organization
- `partners_with` ↔ Organization

#### Search Patterns

```python
# Finding organizations by industry
query = "What are the major tech companies?"
→ Search: entity_type="organization" AND industry="Technology"

# Finding products of organization
query = "What products does Apple make?"
→ Traverse: Organization("Apple Inc") -[produces]-> Product

# Finding competitors
query = "Who are Apple's competitors?"
→ Traverse: Organization("Apple Inc") -[competes_with]- Organization
```

---

### 2.4 Location Node

**Назначение**: Представление географического места

#### Extended Schema

```python
LocationNode = EntityNode + {
    "entity_type": "location",
    "location_metadata": {
        "location_type": str,       # "city", "country", "region", "address"
        "country": str,             # Страна
        "coordinates": {            # GPS координаты
            "lat": float,
            "lon": float
        },
        "population": int,          # Население (для городов/стран)
        "area": float               # Площадь (кв. км)
    }
}
```

#### Пример

```json
{
    "entity_id": "ent-f3a6b8c1d2e5",
    "entity_name": "Cupertino",
    "entity_type": "location",
    "description": "Cupertino is a city in Santa Clara County, California, known as the headquarters location of Apple Inc. The city is part of Silicon Valley and has become synonymous with the tech industry.",
    "location_metadata": {
        "location_type": "city",
        "country": "United States",
        "coordinates": {
            "lat": 37.3229,
            "lon": -122.0322
        },
        "population": 60000
    }
}
```

#### Common Relations

- `contains` → Location (nested locations)
- `headquarters_of` ← Organization
- `located_in` ← Person
- `located_in` ← Event
- `part_of` → Location (region)

---

### 2.5 Event Node

**Назначение**: Представление события или происшествия

#### Extended Schema

```python
EventNode = EntityNode + {
    "entity_type": "event",
    "event_metadata": {
        "event_type": str,          # "conference", "launch", "announcement"
        "date": str,                # Дата события (ISO format)
        "location": str,            # Место проведения
        "participants": list[str],  # Участники
        "outcome": str              # Результат/итог
    }
}
```

#### Пример

```json
{
    "entity_id": "ent-a5d2e8f1b3c6",
    "entity_name": "iPhone 15 Launch Event",
    "entity_type": "event",
    "description": "Apple's annual product launch event where iPhone 15 was announced with new features including USB-C connectivity and upgraded camera system.",
    "event_metadata": {
        "event_type": "product_launch",
        "date": "2023-09-12",
        "location": "Apple Park, Cupertino",
        "participants": ["Tim Cook", "Apple Inc"],
        "outcome": "iPhone 15 announced"
    }
}
```

#### Common Relations

- `organized_by` → Organization
- `participated_in` ← Person
- `located_in` → Location
- `resulted_in` → Product/Outcome
- `preceded_by` ← Event (temporal)
- `followed_by` → Event (temporal)

---

### 2.6 Product Node

**Назначение**: Представление продукта или услуги

#### Extended Schema

```python
ProductNode = EntityNode + {
    "entity_type": "product",
    "product_metadata": {
        "product_type": str,        # "hardware", "software", "service"
        "manufacturer": str,        # Производитель
        "launch_date": str,         # Дата запуска
        "category": str,            # Категория продукта
        "features": list[str],      # Ключевые особенности
        "price_range": str          # Ценовой диапазон
    }
}
```

#### Пример

```json
{
    "entity_id": "ent-b6c3d9e2f4a7",
    "entity_name": "iPhone",
    "entity_type": "product",
    "description": "iPhone is a line of smartphones designed and marketed by Apple Inc. First introduced in 2007, iPhone revolutionized the smartphone industry with its touchscreen interface and App Store ecosystem.",
    "product_metadata": {
        "product_type": "hardware",
        "manufacturer": "Apple Inc",
        "launch_date": "2007-06-29",
        "category": "smartphone",
        "features": ["touchscreen", "iOS", "App Store", "Face ID"],
        "price_range": "$699-$1199"
    }
}
```

#### Common Relations

- `produced_by` → Organization
- `uses_technology` → Concept/Technology
- `competes_with` ↔ Product
- `component_of` → Product (nested)
- `announced_at` → Event

---

### 2.7 Concept Node

**Назначение**: Представление абстрактной концепции, технологии, идеи

#### Extended Schema

```python
ConceptNode = EntityNode + {
    "entity_type": "concept",
    "concept_metadata": {
        "concept_type": str,        # "technology", "theory", "methodology"
        "domain": str,              # Предметная область
        "related_concepts": list[str],
        "applications": list[str]   # Области применения
    }
}
```

#### Пример

```json
{
    "entity_id": "ent-c7d4e1f5b8a9",
    "entity_name": "Machine Learning",
    "entity_type": "concept",
    "description": "Machine Learning is a subset of artificial intelligence that enables systems to learn and improve from experience without being explicitly programmed. It uses statistical techniques to give computers the ability to learn from data.",
    "concept_metadata": {
        "concept_type": "technology",
        "domain": "Artificial Intelligence",
        "related_concepts": ["Deep Learning", "Neural Networks", "AI"],
        "applications": ["image recognition", "natural language processing"]
    }
}
```

#### Common Relations

- `related_to` ↔ Concept
- `part_of` → Concept (broader concept)
- `applied_in` → Product/Organization
- `researched_by` ← Organization/Person
- `enables` → Technology/Product

---

### 2.8 Category Node

**Назначение**: Представление категории, класса, группы

#### Extended Schema

```python
CategoryNode = EntityNode + {
    "entity_type": "category",
    "category_metadata": {
        "parent_category": str,     # Родительская категория
        "subcategories": list[str], # Подкатегории
        "member_count": int         # Количество членов
    }
}
```

#### Пример

```json
{
    "entity_id": "ent-d8e5f2a6c9b1",
    "entity_name": "Consumer Electronics",
    "entity_type": "category",
    "description": "Consumer Electronics encompasses electronic equipment intended for everyday use by individuals. This includes devices for entertainment, communications, and home office productivity.",
    "category_metadata": {
        "parent_category": "Technology",
        "subcategories": ["Smartphones", "Laptops", "Tablets"],
        "member_count": 150
    }
}
```

#### Common Relations

- `belongs_to` ← Entity (any type)
- `subcategory_of` → Category
- `contains` → Entity (members)

---

### 2.9 Other Node

**Назначение**: Представление сущности, не подходящей под другие типы

#### Schema

```python
OtherNode = EntityNode + {
    "entity_type": "other",
    "other_metadata": {
        "suggested_type": str,      # Предполагаемый тип (для будущей классификации)
        "characteristics": list[str]
    }
}
```

#### Use Cases

- Редкие или специфичные типы сущностей
- Временная категория для неклассифицированных entities
- Entities требующие custom handling

---

## Node Lifecycle

### Creation Flow

```
Text Chunk → Entity Extraction (LLM) → Parsing → Validation → Node Creation
                                                                     ↓
                                                          [Knowledge Graph]
                                                                     ↓
                                                     Graph DB + Vector DB + KV Storage
```

### Update Flow

```
New Mention → Entity Detected → Check Existing → Merge Descriptions (LLM)
                                                         ↓
                                                  Update Node Attributes
                                                         ↓
                                                  Recompute Embedding
```

### Deletion Flow

```
Delete Request → Remove from Graph DB → Remove from Vector DB → Remove from KV Storage
                                                                        ↓
                                                            Orphaned Relations Cleanup
```

## Performance Considerations

### Storage Overhead

| Node Type | Avg Size | Vector Dim | Total Footprint |
|-----------|----------|-----------|----------------|
| Document | 50-500 KB | N/A | 50-500 KB |
| Chunk | 2-8 KB | 1536 | 8-14 KB |
| Entity | 0.5-2 KB | 1536 | 6-8 KB |

### Query Performance

| Operation | Complexity | Typical Time |
|-----------|-----------|--------------|
| Get Node by ID | O(1) | <1ms |
| Search by Type | O(log N) | 1-10ms |
| Vector Search | O(log N) | 10-50ms |
| Subgraph Extract | O(E × D) | 50-500ms |

## Best Practices

### Node Naming
- ✅ Use Title Case for entity names
- ✅ Normalize variations (e.g., "Apple Inc" vs "Apple Inc.")
- ✅ Store aliases in metadata

### Metadata Management
- ✅ Store all source attribution
- ✅ Include confidence scores
- ✅ Track creation and update timestamps

### Type Selection
- ✅ Use most specific type available
- ✅ Prefer established types over "Other"
- ✅ Document custom type extensions

---

**Следующий раздел**: [02-relationship-types.md](02-relationship-types.md) - Типы отношений между узлами
