# LightRAG: Обзор Pipeline Обработки Документов

## Введение

LightRAG - это система для построения графов знаний (Knowledge Graph) с использованием языковых моделей и векторного поиска. Система преобразует неструктурированные текстовые документы в структурированный граф знаний, который поддерживает семантический поиск и навигацию по связям между сущностями.

## Архитектурный Обзор

```
┌─────────────────────────────────────────────────────────────────┐
│                    DOCUMENT INPUT LAYER                          │
│  (Текстовые документы, файлы, строки)                           │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PHASE 1: ENQUEUE PHASE                          │
│  • Валидация и санитизация текста                               │
│  • Генерация/проверка ID документов                             │
│  • Дедупликация документов                                      │
│  • Создание статуса обработки (PENDING)                         │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              PHASE 2: PROCESSING PHASE                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  STAGE 1: CHUNKING LAYER                                  │ │
│  │  • Token-based chunking с overlap                         │ │
│  │  • Character-based splitting (опционально)                │ │
│  │  • Semantic boundary preservation                         │ │
│  └───────────────────┬───────────────────────────────────────┘ │
│                      │                                           │
│                      ▼                                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  STAGE 2: ENTITY EXTRACTION LAYER                         │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  LLM Agent 1: Initial Extraction                    │ │ │
│  │  │  • Идентификация сущностей                          │ │ │
│  │  │  • Извлечение отношений                             │ │ │
│  │  │  • Категоризация типов                              │ │ │
│  │  └──────────────────┬──────────────────────────────────┘ │ │
│  │                     │                                      │ │
│  │                     ▼                                      │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  LLM Agent 2: Gleaning Pass (опционально)          │ │ │
│  │  │  • Выявление пропущенных сущностей                  │ │ │
│  │  │  • Улучшение описаний                               │ │ │
│  │  │  • Исправление форматирования                       │ │ │
│  │  └──────────────────┬──────────────────────────────────┘ │ │
│  │                     │                                      │ │
│  │                     ▼                                      │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Parsing & Validation                               │ │ │
│  │  │  • Парсинг LLM output                               │ │ │
│  │  │  • Валидация структуры                              │ │ │
│  │  │  • Санитизация данных                               │ │ │
│  │  └──────────────────┬──────────────────────────────────┘ │ │
│  └────────────────────┼──────────────────────────────────────┘ │
│                       │                                          │
│                       ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  STAGE 3: GRAPH CONSTRUCTION LAYER                        │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Phase 1: Entity Processing                         │ │ │
│  │  │  • Проверка существования сущности                  │ │ │
│  │  │  • LLM Agent 3: Description Merging                 │ │ │
│  │  │  • Update Graph DB                                  │ │ │
│  │  │  • Update Vector DB                                 │ │ │
│  │  └──────────────────┬──────────────────────────────────┘ │ │
│  │                     │                                      │ │
│  │                     ▼                                      │ │
│  │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │  Phase 2: Relationship Processing                   │ │ │
│  │  │  • Валидация сущностей отношения                    │ │ │
│  │  │  • LLM Agent 3: Relationship Merging                │ │ │
│  │  │  • Update Graph DB                                  │ │ │
│  │  │  • Update Vector DB                                 │ │ │
│  │  └──────────────────┬──────────────────────────────────┘ │ │
│  └────────────────────┼──────────────────────────────────────┘ │
└───────────────────────┼──────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE LAYER                                 │
│  • Graph Database (NetworkX/Neo4j/MongoDB/PostgreSQL)           │
│  • Vector Database (Nano/Milvus/Qdrant)                         │
│  • Key-Value Storage (JSON/MongoDB/PostgreSQL)                  │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                INDEXED KNOWLEDGE GRAPH                           │
│  • Queryable entities and relationships                          │
│  • Semantic search capabilities                                  │
│  • Citation tracking to source documents                         │
└─────────────────────────────────────────────────────────────────┘
```

## Ключевые Компоненты

### 1. Document Ingestion Layer
- **Функция**: `ainsert()` в `lightrag.py:901`
- **Назначение**: Прием документов и управление процессом обработки
- **Режимы работы**: синхронный и асинхронный

### 2. Chunking Layer
- **Функция**: `chunking_by_token_size()` в `operate.py:66`
- **Назначение**: Разбиение документов на семантически значимые фрагменты
- **Стратегии**: token-based с overlap, character-based splitting

### 3. Entity Extraction Layer
- **Функция**: `extract_entities()` в `operate.py:2010`
- **Назначение**: Извлечение сущностей и отношений с помощью LLM
- **Режимы**: initial extraction + gleaning pass

### 4. Graph Construction Layer
- **Функция**: `merge_nodes_and_edges()` в `operate.py:1579`
- **Назначение**: Слияние извлеченных данных в единый граф знаний
- **Фазы**: entity processing → relationship processing

### 5. Storage Layer
- **Назначение**: Персистентное хранение данных
- **Типы**: Graph DB, Vector DB, Key-Value Storage

## Типы Данных

### TextChunk
```python
{
    "content": str,           # Текст фрагмента
    "tokens": int,            # Количество токенов
    "chunk_order_index": int, # Порядковый номер в документе
    "full_doc_id": str,       # ID родительского документа
    "file_path": str          # Путь к исходному файлу
}
```

### Entity (Node)
```python
{
    "entity_name": str,        # Нормализованное имя (Title Case)
    "entity_type": str,        # Тип (person, organization, location, ...)
    "description": str,        # Комплексное описание
    "source_id": str,          # ID фрагмента-источника
    "file_path": str,          # Путь к исходному файлу
    "created_at": int          # Timestamp создания
}
```

### Relationship (Edge)
```python
{
    "source_entity": str,            # Исходная сущность
    "target_entity": str,            # Целевая сущность
    "keywords": str,                 # High-level ключевые слова
    "description": str,              # Описание отношения
    "source_id": str,                # ID фрагмента-источника
    "file_path": str,                # Путь к исходному файлу
    "created_at": int,               # Timestamp создания
    "weight": float                  # Вес отношения (опционально)
}
```

## Языковые Агенты и Их Роли

LightRAG использует три типа LLM-агентов на разных этапах pipeline:

### Agent 1: Entity Extraction Agent
- **Промпт**: `entity_extraction_system_prompt`
- **Вход**: текстовый chunk
- **Выход**: entities + relationships
- **Приоритет**: 0-7 (зависит от порядка chunk)

### Agent 2: Gleaning Agent
- **Промпт**: `entity_continue_extraction_user_prompt`
- **Вход**: previous extraction + original text
- **Выход**: missed/corrected entities + relationships
- **Приоритет**: 0-7

### Agent 3: Summary Agent
- **Промпт**: `summarize_entity_descriptions`
- **Вход**: list of descriptions
- **Выход**: merged summary
- **Приоритет**: 8 (высокий)
- **Стратегия**: map-reduce для больших списков

## Семантический/Концептуальный Слой

### Векторные Представления
1. **Chunk Embeddings**: семантическое представление текстовых фрагментов
2. **Entity Embeddings**: векторы сущностей (`entity_name + description`)
3. **Relationship Embeddings**: векторы отношений (`source + target + keywords + description`)

### Graph-Based Semantics
1. **Entity Co-occurrence**: сущности связанные через общие chunks
2. **Relationship Paths**: многоуровневые связи между сущностями
3. **Community Detection**: кластеризация связанных сущностей

### Hybrid Search
- **Dense Retrieval**: векторный поиск по embeddings
- **Graph Traversal**: навигация по структуре графа
- **Ranking Fusion**: комбинированная оценка релевантности

## Ключевые Особенности

### 1. Конкурентная Обработка
- Параллельная обработка chunks
- Асинхронное извлечение сущностей
- Keyed locks для предотвращения race conditions

### 2. Кеширование
- LLM response cache (MD5-based)
- Предотвращение дублирующих запросов
- Chunk-level tracking

### 3. Обработка Ошибок
- Retry logic с exponential backoff
- First-exception cancellation
- Detailed error prefixing

### 4. Масштабируемость
- Configurable async limits
- Semaphore-based concurrency control
- Map-reduce для больших описаний

## Workflow: От Документа До Графа

```
Input Document
    ↓
[Validation & Deduplication]
    ↓
[Token-based Chunking]
    ↓
[Parallel Entity Extraction]
    ↓ (для каждого chunk)
LLM Call: Initial Extraction → Entities + Relationships
    ↓
[Optional: Gleaning Pass]
    ↓
LLM Call: Find Missed Entities → Additional Entities + Relationships
    ↓
[Parsing & Validation]
    ↓
[Graph Construction: Phase 1]
    ↓ (для каждой entity)
Check if exists → LLM Call: Merge Descriptions → Update Graph & Vector DB
    ↓
[Graph Construction: Phase 2]
    ↓ (для каждого relationship)
Validate entities → LLM Call: Merge Descriptions → Update Graph & Vector DB
    ↓
[Storage Finalization]
    ↓
Indexed Knowledge Graph
```

## Метрики и Мониторинг

### Pipeline Status Tracking
- **Track ID**: уникальный идентификатор обработки
- **Document Status**: PENDING → PROCESSING → PROCESSED/FAILED
- **Progress Logging**: количество обработанных entities/relationships
- **Error Tracking**: детальные логи ошибок с контекстом

### Performance Metrics
- **Chunk processing rate**: chunks/second
- **Entity extraction rate**: entities/second
- **LLM call latency**: средняя задержка запросов
- **Storage operation latency**: время записи в БД

## Конфигурируемые Параметры

### Chunking Parameters
- `chunk_token_size`: максимальный размер chunk (default: 1024)
- `chunk_overlap_token_size`: размер overlap (default: 128)
- `split_by_character`: символ для разделения (default: None)

### Extraction Parameters
- `entity_extract_max_gleaning`: количество gleaning passes (default: 1)
- `llm_model_max_async`: максимум параллельных LLM вызовов
- `entity_types`: список типов сущностей для извлечения

### Summary Parameters
- `summary_context_size`: максимум токенов для одного summary call
- `summary_max_tokens`: максимум токенов в output
- `force_llm_summary_on_merge`: порог для принудительного summarization

## Следующие Разделы

1. **[Document Ingestion](01-document-ingestion.md)** - детали приема документов
2. **[Chunking Strategy](02-chunking-strategy.md)** - стратегии разбиения
3. **[Entity Extraction](03-entity-extraction.md)** - извлечение сущностей и отношений
4. **[Graph Construction](04-graph-construction.md)** - построение графа знаний
5. **[Storage Architecture](05-storage-architecture.md)** - архитектура хранения
6. **[LLM Agents](06-llm-agents.md)** - роли и функции агентов
7. **[Semantic Layer](07-semantic-layer.md)** - концептуальный слой
