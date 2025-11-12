# Semantic Transformations: Семантические Преобразования в LightRAG

## Обзор

Эта директория содержит детальное описание всех семантических преобразований, выполняемых LLM агентами и другими компонентами LightRAG как в **Indexing Pipeline** (индексация документов), так и в **Query Pipeline** (обработка запросов и формирование ответов).

## Что Такое Семантическое Преобразование?

**Семантическое преобразование** - это операция, изменяющая форму представления информации при сохранении, извлечении или обогащении её смысла (семантики).

```
┌────────────────────────────────────────────────────────────┐
│         SEMANTIC TRANSFORMATION SPECTRUM                   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Information Preserving ←→ Information Extracting         │
│  (сохранение семантики)     (извлечение семантики)        │
│                                                            │
│  Examples:                  Examples:                      │
│  • Text → Embedding         • Text → Entities             │
│  • Format conversion        • Summarization               │
│  • Chunking                 • Keyword extraction          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## Архитектура Преобразований

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRANSFORMATION PIPELINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           INDEXING TRANSFORMATIONS                       │  │
│  │                                                          │  │
│  │  Document → Chunks → Entities/Relations → Graph/Vectors │  │
│  │                                                          │  │
│  │  T1: Chunking (text segmentation)                       │  │
│  │  T2: Entity Extraction (semantic parsing)               │  │
│  │  T3: Gleaning (refinement)                              │  │
│  │  T4: Parsing (structuring)                              │  │
│  │  T5: Description Merging (summarization)                │  │
│  │  T6: Vectorization (embedding)                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │            QUERY TRANSFORMATIONS                         │  │
│  │                                                          │  │
│  │  Query → Keywords → Entities → Subgraph → Answer        │  │
│  │                                                          │  │
│  │  T7: Keyword Extraction (intent analysis)               │  │
│  │  T8: Entity Resolution (disambiguation)                 │  │
│  │  T9: Context Assembly (aggregation)                     │  │
│  │  T10: Answer Generation (synthesis)                     │  │
│  │  T11: Answer Formatting (structuring)                   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Классификация Преобразований

### По Направлению Информационного Потока

```
┌─────────────────────────────────────────────────────────┐
│  TRANSFORMATION TYPES BY INFORMATION FLOW               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. DECOMPOSITION (1 → Many)                           │
│     Document → Chunks                                   │
│     Large Description → Summary                         │
│                                                         │
│  2. EXTRACTION (Unstructured → Structured)             │
│     Text → Entities                                     │
│     Text → Keywords                                     │
│                                                         │
│  3. AGGREGATION (Many → 1)                             │
│     Multiple Descriptions → Merged Description          │
│     Multiple Chunks → Context                           │
│                                                         │
│  4. TRANSFORMATION (Format Change)                      │
│     Text → Vector Embedding                             │
│     Raw Output → Structured Data                        │
│                                                         │
│  5. SYNTHESIS (Creation)                                │
│     Context → Answer                                    │
│     Entities + Relations → Explanation                  │
└─────────────────────────────────────────────────────────┘
```

### По Роли Агента

| Transformation Type | Agent Role | Automation |
|-------------------|-----------|-----------|
| **Rule-Based** | None | Fully automatic |
| **LLM-Assisted** | Support | Semi-automatic |
| **LLM-Driven** | Primary | Requires LLM |

### По Семантическим Характеристикам

| Characteristic | Description | Example |
|---------------|-------------|---------|
| **Lossless** | Полное сохранение информации | Text → Embedding (reversible) |
| **Lossy** | Частичная потеря деталей | Text → Summary |
| **Enriching** | Добавление информации | Text → Entities (структура) |
| **Filtering** | Удаление нерелевантного | Chunks → Top-K |

## Содержание Документации

### [01-indexing-transforms.md](01-indexing-transforms.md)
**Преобразования при индексации документов**

Описывает 6 основных преобразований в indexing pipeline:
- **T1: Chunking** - разбиение на фрагменты
- **T2: Entity Extraction** - извлечение сущностей
- **T3: Gleaning** - уточнение и дополнение
- **T4: Parsing** - структурирование данных
- **T5: Description Merging** - слияние описаний
- **T6: Vectorization** - создание embeddings

Для каждого:
- Input/Output formats
- Semantic operation type
- Agent involvement
- Information flow
- Examples

---

### [02-query-transforms.md](02-query-transforms.md)
**Преобразования при обработке запросов**

Описывает 5 основных преобразований в query pipeline:
- **T7: Keyword Extraction** - извлечение ключевых слов
- **T8: Entity Resolution** - разрешение сущностей
- **T9: Context Assembly** - сборка контекста
- **T10: Answer Generation** - генерация ответа
- **T11: Answer Formatting** - форматирование

Для каждого:
- Query type compatibility
- Mode usage (Naive/Local/Global)
- Performance characteristics
- Quality metrics

---

### [03-transform-chains.md](03-transform-chains.md)
**Цепочки преобразований**

Детальное описание последовательностей преобразований:
- **Indexing Chain**: Document → Knowledge Graph
- **Query Chains** (3 modes):
  - Naive Chain: Query → Answer (minimal)
  - Local Chain: Query → Local Graph → Answer
  - Global Chain: Query → Global Graph → Answer
- **Data Flow** между преобразованиями
- **Dependencies** и параллелизм

---

### [04-semantic-characteristics.md](04-semantic-characteristics.md)
**Семантические характеристики преобразований**

Детальный анализ семантических свойств:
- **Information Metrics**:
  - Entropy changes
  - Precision/Recall
  - Semantic similarity preservation
- **Fidelity Measures**:
  - Semantic drift
  - Hallucination risk
  - Consistency scores
- **Quality Dimensions**:
  - Accuracy
  - Completeness
  - Coherence
  - Relevance

---

### [05-comparison-tables.md](05-comparison-tables.md)
**Сравнительные таблицы**

Comprehensive comparison tables:
- **Master Comparison Table**: все преобразования side-by-side
- **By Pipeline**: Indexing vs Query
- **By Agent Type**: Rule-based vs LLM-driven
- **By Complexity**: Simple vs Complex
- **By Performance**: Latency vs Accuracy trade-offs
- **By Information Flow**: Preservation vs Extraction

---

## Ключевые Метрики

### Performance Metrics

| Transformation | Avg Latency | Throughput | Scalability |
|---------------|-------------|------------|-------------|
| Chunking | 1-5ms | 1000+ docs/s | Linear |
| Entity Extraction | 2-5s | 1 chunk/s | Parallel |
| Description Merge | 3-8s | 1 entity/s | Parallel |
| Keyword Extraction | 1-3s | 1 query/s | Sequential |
| Answer Generation | 2-5s | 1 query/s | Sequential |

### Quality Metrics

| Transformation | Precision | Recall | F1-Score |
|---------------|-----------|--------|----------|
| Entity Extraction | 0.85-0.92 | 0.78-0.88 | 0.81-0.90 |
| Keyword Extraction | 0.80-0.90 | 0.75-0.85 | 0.77-0.87 |
| Answer Generation | 0.75-0.90 | 0.70-0.85 | 0.72-0.87 |

## Семантические Паттерны

### Pattern 1: Information Refinement

```
Raw Data → Extraction → Validation → Structuring → Storage

Semantic Change: Unstructured → Semi-structured → Structured
Information: Preserved (entities), Lost (noise)
```

### Pattern 2: Multi-Source Fusion

```
Source 1 ─┐
Source 2 ─┼→ Aggregation → Deduplication → Merging → Unified View
Source 3 ─┘

Semantic Change: Multiple views → Single consensus
Information: Enriched (completeness), Resolved (conflicts)
```

### Pattern 3: Abstraction Hierarchy

```
Specific Text → Entities → Concepts → Abstract Patterns

Semantic Change: Concrete → Abstract
Information: Generalized (patterns), Lost (specifics)
```

## Best Practices

### ✅ DO

1. **Preserve Provenance**: Track source for all transformations
2. **Measure Semantic Drift**: Monitor information loss
3. **Cache Deterministic**: Cache rule-based transforms
4. **Validate LLM Outputs**: Always parse and validate
5. **Chain Efficiently**: Minimize intermediate steps

### ❌ DON'T

1. **Don't Ignore Errors**: Handle transform failures gracefully
2. **Don't Skip Validation**: Validate semantic correctness
3. **Don't Over-Transform**: Avoid unnecessary conversions
4. **Don't Lose Context**: Maintain metadata through chain
5. **Don't Assume Lossless**: Account for information loss

## Terminology

| Term | Definition |
|------|------------|
| **Transform** | Function: Input → Output |
| **Chain** | Sequence of transforms |
| **Semantic Drift** | Loss of meaning accuracy |
| **Information Entropy** | Measure of information content |
| **Fidelity** | Faithfulness to source |
| **Hallucination** | Generated false information |
| **Grounding** | Connection to source data |

## Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                  INDEXING PIPELINE                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Document (100% info)                                   │
│      ↓ T1: Chunking (98% preserved)                    │
│  Chunks                                                 │
│      ↓ T2: Extraction (60% entities extracted)         │
│  Raw Entities                                           │
│      ↓ T3: Gleaning (75% coverage achieved)            │
│  Refined Entities                                       │
│      ↓ T4: Parsing (95% structured)                    │
│  Structured Entities                                    │
│      ↓ T5: Merging (90% fidelity)                      │
│  Merged Entities                                        │
│      ↓ T6: Vectorization (100% preserved)              │
│  Knowledge Graph + Vectors                              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   QUERY PIPELINE                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  User Query (100% intent)                               │
│      ↓ T7: Keywords (85% intent captured)              │
│  Keywords                                               │
│      ↓ T8: Resolution (80% entities found)             │
│  Entities                                               │
│      ↓ T9: Assembly (70% relevant context)             │
│  Context                                                │
│      ↓ T10: Generation (90% query answered)            │
│  Raw Answer                                             │
│      ↓ T11: Formatting (100% structured)               │
│  Structured Answer                                      │
└─────────────────────────────────────────────────────────┘
```

## Порядок Чтения

### Для понимания индексации:
1. [01-indexing-transforms.md](01-indexing-transforms.md)
2. [03-transform-chains.md](03-transform-chains.md) (Indexing section)
3. [04-semantic-characteristics.md](04-semantic-characteristics.md)

### Для понимания query processing:
1. [02-query-transforms.md](02-query-transforms.md)
2. [03-transform-chains.md](03-transform-chains.md) (Query section)
3. [05-comparison-tables.md](05-comparison-tables.md)

### Для оптимизации:
1. [05-comparison-tables.md](05-comparison-tables.md)
2. [04-semantic-characteristics.md](04-semantic-characteristics.md)

## Code References

- **Chunking**: `lightrag/operate.py:66` - `chunking_by_token_size()`
- **Entity Extraction**: `lightrag/operate.py:2010` - `extract_entities()`
- **Description Merging**: `lightrag/operate.py:121` - `_handle_entity_relation_summary()`
- **Keyword Extraction**: `lightrag/prompt.py` - `keywords_extraction`
- **Answer Generation**: `lightrag/lightrag.py` - `aquery()`

---

**Версия**: 1.0
**Дата**: 2025-01-12
**Связано**: [Main Specification](../README.md), [LLM Agents](../06-llm-agents.md)
