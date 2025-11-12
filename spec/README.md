# LightRAG Document Processing Pipeline Specification

## Обзор

Эта директория содержит детальную техническую спецификацию pipeline обработки и индексации документов в LightRAG. Спецификация описывает все этапы преобразования неструктурированных текстовых документов в структурированный граф знаний с семантическими представлениями.

## Структура Документации

### [00-overview.md](00-overview.md)
**Общий обзор системы**

Содержит:
- Архитектурный обзор всего pipeline
- Диаграмму потока данных
- Ключевые компоненты системы
- Типы данных (TextChunk, Entity, Relationship)
- Роли языковых агентов (краткий обзор)
- Workflow от документа до графа
- Конфигурируемые параметры

**Аудитория**: Все, кто хочет получить общее представление о системе

---

### [01-document-ingestion.md](01-document-ingestion.md)
**Прием и валидация документов**

Содержит:
- Двухфазный процесс ingestion
  - Phase 1: Enqueue Phase (валидация, дедупликация)
  - Phase 2: Processing Phase (chunking, extraction)
- Детали санитизации текста
- Генерация Document IDs
- Tracking system для мониторинга обработки
- Document Status Storage (PENDING → PROCESSING → PROCESSED/FAILED)
- Стратегии дедупликации
- Обработка ошибок и retry logic

**Аудитория**: Разработчики, работающие с input layer и data ingestion

---

### [02-chunking-strategy.md](02-chunking-strategy.md)
**Разбиение документов на семантические фрагменты**

Содержит:
- Три стратегии chunking:
  1. Pure Token-Based Chunking
  2. Character-Based Pre-Splitting
  3. Character-Only Splitting
- Алгоритм overlap для сохранения контекста
- Semantic boundary detection
- Chunk metadata structure
- Интеграция с pipeline
- Advanced techniques (hierarchical, content-aware chunking)
- Best practices и метрики качества

**Аудитория**: ML-инженеры, исследователи, оптимизирующие retrieval quality

---

### [03-entity-extraction.md](03-entity-extraction.md)
**Извлечение сущностей и отношений с помощью LLM**

Содержит:
- Трехэтапный процесс extraction:
  1. Stage 1: Initial Extraction (LLM Agent 1)
  2. Stage 2: Gleaning Pass (LLM Agent 2)
  3. Stage 3: Parsing & Validation
- Детальные промпты для LLM агентов
- System prompts и инструкции
- Entity и Relationship extraction rules
- Параллельная обработка chunks
- Кеширование LLM responses
- Output structures (entities_dict, relationships_dict)
- Error handling

**Аудитория**: Prompt engineers, NLP специалисты, разработчики работающие с LLM

---

### [04-graph-construction.md](04-graph-construction.md)
**Построение графа знаний**

Содержит:
- Двухфазный процесс merging:
  - Phase 1: Entity Processing
  - Phase 2: Relationship Processing
- Keyed locks для предотвращения race conditions
- LLM Agent 3: Description Summarization
  - Map-Reduce strategy для больших описаний
  - Decision tree для summarization
- Graph DB и Vector DB синхронизация
- Consistency guarantees
- Storage finalization
- Performance optimizations

**Аудитория**: Backend разработчики, database engineers, distributed systems специалисты

---

### [05-storage-architecture.md](05-storage-architecture.md)
**Архитектура хранения данных**

Содержит:
- Трехкомпонентная архитектура:
  1. Key-Value Storage (documents, chunks, metadata)
  2. Vector Database Storage (embeddings для semantic search)
  3. Graph Database Storage (entities, relationships, traversal)
- Base classes и их методы
- Реализации для разных backends:
  - JSON, MongoDB, PostgreSQL (KV Storage)
  - NanoDB, Milvus, Qdrant (Vector DB)
  - NetworkX, Neo4j, MongoDB (Graph DB)
- Storage initialization
- Рекомендуемые комбинации для разных масштабов

**Аудитория**: DevOps, infrastructure engineers, system architects

---

### [06-llm-agents.md](06-llm-agents.md)
**Роли и функции языковых агентов**

Содержит:
- Детальное описание 4 типов LLM агентов:
  1. **Agent 1: Entity Extraction Agent**
     - Роль: Knowledge Graph Specialist
     - Ответственности: идентификация entities, extraction relationships
     - Промпт: entity_extraction_system_prompt
     - Priority: 0-7

  2. **Agent 2: Gleaning Agent**
     - Роль: Quality Assurance Specialist
     - Ответственности: поиск пропущенных/некорректных extractions
     - Промпт: entity_continue_extraction_user_prompt
     - Merging strategy

  3. **Agent 3: Description Summarization Agent**
     - Роль: Knowledge Curator
     - Ответственности: синтез множественных descriptions
     - Промпт: summarize_entity_descriptions
     - Map-Reduce implementation

  4. **Agent 4: Query Processing Agents**
     - Keyword Extraction, Context Generation, Response Generation
- LLM wrapper function с кешированием
- OpenAI implementation с retry logic
- Agent configuration и best practices

**Аудитория**: AI/ML engineers, prompt engineers, researchers

---

### [07-semantic-layer.md](07-semantic-layer.md)
**Концептуальный и семантический слой**

Содержит:
- Трехуровневая архитектура:
  1. **Level 1: Vector Representations**
     - Chunk Embeddings (dense retrieval)
     - Entity Embeddings (semantic entity search)
     - Relationship Embeddings (relationship patterns)

  2. **Level 2: Graph Structure Semantics**
     - Entity Co-occurrence (implicit relationships)
     - Relationship Paths (multi-hop reasoning)
     - Community Detection (thematic clustering)

  3. **Level 3: Hybrid Semantic Search**
     - Vector + Graph Fusion
     - Query Expansion
     - Ranking Fusion (RRF)
- Semantic query modes:
  - Naive Mode (direct chunk retrieval)
  - Local Mode (entity-centric with local graph)
  - Global Mode (full graph traversal)
  - Hybrid Mode (adaptive selection)
- Performance optimizations

**Аудитория**: ML researchers, information retrieval специалисты, semantic search engineers

---

## Порядок Чтения

### Для быстрого ознакомления:
1. [00-overview.md](00-overview.md) - получить общее представление

### Для понимания data flow:
1. [00-overview.md](00-overview.md)
2. [01-document-ingestion.md](01-document-ingestion.md)
3. [02-chunking-strategy.md](02-chunking-strategy.md)
4. [03-entity-extraction.md](03-entity-extraction.md)
5. [04-graph-construction.md](04-graph-construction.md)

### Для работы с LLM:
1. [03-entity-extraction.md](03-entity-extraction.md)
2. [06-llm-agents.md](06-llm-agents.md)

### Для оптимизации retrieval:
1. [02-chunking-strategy.md](02-chunking-strategy.md)
2. [07-semantic-layer.md](07-semantic-layer.md)

### Для настройки инфраструктуры:
1. [05-storage-architecture.md](05-storage-architecture.md)
2. [04-graph-construction.md](04-graph-construction.md)

## Ключевые Концепции

### Pipeline Flow
```
Documents → Validation → Chunking → Entity Extraction → Graph Construction → Storage → Indexed Knowledge Graph
```

### LLM Agents
```
Agent 1 (Extraction) → Agent 2 (Gleaning) → Agent 3 (Summarization)
                                              ↓
                                      Unified Graph
```

### Semantic Layer
```
Text → Embeddings → Vector Search
     ↓
     Entities & Relations → Graph → Graph Traversal
                             ↓
                       Hybrid Retrieval
```

## Code References

Все описанные компоненты соответствуют реальному коду LightRAG:

- **Main Pipeline**: `lightrag/lightrag.py`
- **Operations**: `lightrag/operate.py`
- **Prompts**: `lightrag/prompt.py`
- **Storage**: `lightrag/base.py`, `lightrag/storage/`
- **LLM**: `lightrag/llm/`, `lightrag/utils.py`

## Терминология

| Термин | Описание |
|--------|----------|
| **Chunk** | Текстовый фрагмент документа (обычно 512-1024 токена) |
| **Entity** | Сущность извлеченная из текста (person, organization, location, etc.) |
| **Relationship** | Связь между двумя entities |
| **Node** | Узел в графе знаний (entity) |
| **Edge** | Ребро в графе знаний (relationship) |
| **Embedding** | Векторное представление текста для semantic search |
| **Gleaning** | Процесс "дожима" информации, второй проход LLM для поиска пропущенного |
| **Summarization** | Слияние множественных descriptions в cohesive summary |
| **Co-occurrence** | Совместное появление entities в тексте |
| **Community** | Кластер связанных entities в графе |
| **Graph Traversal** | Навигация по графу (BFS, DFS, etc.) |
| **Hybrid Search** | Комбинирование vector и graph-based retrieval |

## Метрики и Performance

### Typical Processing Times (для документа ~10K tokens)
- **Document Ingestion**: <1s
- **Chunking**: <1s
- **Entity Extraction**: 2-5s per chunk (зависит от LLM)
- **Gleaning**: 2-5s per chunk (если enabled)
- **Graph Construction**: 1-3s per entity/relationship
- **Total**: ~30-60s для документа из 10 chunks

### Token Consumption (per chunk ~1024 tokens)
- **Extraction**: ~2300 tokens (prompt + chunk + output)
- **Gleaning**: ~4000 tokens (с history)
- **Summarization**: ~500-2000 tokens (зависит от количества descriptions)

### Cache Hit Rates
- **LLM Response Cache**: 70-90% при re-processing
- **Embedding Cache**: 80-95% для frequently accessed entities

## Contributing

При внесении изменений в документацию следуйте этим guidelines:

1. **Структура**: Сохраняйте иерархическую структуру (Overview → Details → Examples)
2. **Code References**: Указывайте точные locations в коде (file:line)
3. **Диаграммы**: Используйте ASCII art для визуализации
4. **Примеры**: Включайте практические примеры кода
5. **Термины**: Добавляйте новые термины в Терминологию секцию

## License

Эта документация является частью проекта LightRAG и распространяется под той же лицензией.

## Feedback

Для вопросов, предложений или исправлений создавайте issues в основном репозитории LightRAG.

---

**Версия**: 1.0
**Дата**: 2025-01-12
**Автор**: LightRAG Team
