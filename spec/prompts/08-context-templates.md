# P8: Context Templates

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P8 |
| **Имя** | `kg_query_context` / `naive_query_context` |
| **Тип** | Template |
| **Трансформация** | T9 (Context Assembly) |
| **Format** | JSON + Plain Text |
| **Расположение** | `lightrag/prompt.py:322-358` |

---

## Назначение

Templates для форматирования context data перед передачей в P6/P7 для генерации ответа. Структурируют KG entities, relations, chunks, и references.

---

## KG Query Context (для P6)

```python
PROMPTS["kg_query_context"] = """
Entities Data From Knowledge Graph(KG):

```json
{entities_str}
```

Relationships Data From Knowledge Graph(KG):

```json
{relations_str}
```

Original Texts From Document Chunks(DC):

```json
{text_chunks_str}
```

Document Chunks (DC) Reference Document List: (Each entry begins with [reference_id])

{reference_list_str}

"""
```

### Parameters

| Параметр | Описание | Format |
|----------|----------|--------|
| `{entities_str}` | KG entities | JSONL (one per line) |
| `{relations_str}` | KG relationships | JSONL |
| `{text_chunks_str}` | Document chunks | JSONL with reference_id |
| `{reference_list_str}` | Source documents | `[n] Title` format |

---

## Naive Query Context (для P7)

```python
PROMPTS["naive_query_context"] = """
Original Texts From Document Chunks(DC):

```json
{text_chunks_str}
```

Document Chunks (DC) Reference Document List: (Each entry begins with [reference_id])

{reference_list_str}

"""
```

Simpler - только chunks и references.

---

## Example: KG Context

```
Entities Data From Knowledge Graph(KG):

```json
{"entity_name": "Alice Smith", "entity_type": "person", "description": "Alice Smith is the CEO of TechCorp..."}
{"entity_name": "TechCorp", "entity_type": "organization", "description": "TechCorp is a technology company..."}
```

Relationships Data From Knowledge Graph(KG):

```json
{"source": "Alice Smith", "target": "TechCorp", "keywords": "leadership, employment", "description": "Alice Smith serves as CEO of TechCorp"}
```

Original Texts From Document Chunks(DC):

```json
{"content": "Alice joined TechCorp as CEO in 2020...", "reference_id": 1}
{"content": "Under Alice's leadership, TechCorp launched...", "reference_id": 2}
```

Document Chunks (DC) Reference Document List:

[1] TechCorp Annual Report 2020
[2] Leadership Announcements
```

---

## Role in Pipeline

```
T9: Context Assembly
════════════════════

Retrieved Data:
• Entities from entity_vdb
• Relationships from graph_db
• Chunks from chunks_vdb
• Reference documents metadata

    ↓
[P8: Context Templates] ← THIS
    ↓
Formatted Context String
    ↓
Insert into P6/P7 {context_data}
    ↓
T10: Answer Generation
```

---

## Design Rationale

### 1. JSONL Format

```python
# Why JSONL (not JSON array)?
benefits = {
    "streaming": "Can process line by line",
    "truncation": "Easy to limit by line count",
    "readability": "LLM can parse each object separately"
}
```

### 2. Explicit Labeling

```
"Entities Data From Knowledge Graph(KG):"
"Relationships Data From Knowledge Graph(KG):"
"Original Texts From Document Chunks(DC):"
```

**Purpose**: Clear semantic separation for LLM understanding

### 3. Reference List Format

```
[1] Document Title
[2] Another Document
```

**Purpose**: Easy for LLM to cite as `[1]`, `[2]` etc.

---

## External Dependencies

### JSONL Generation

```python
import json

def format_entities_jsonl(entities: list[dict]) -> str:
    """Format entities as JSONL"""
    return "\n".join(json.dumps(e) for e in entities)
```

### Reference List Generation

```python
def format_reference_list(chunks: list[dict]) -> str:
    """Format reference list"""
    references = {}
    for chunk in chunks:
        ref_id = chunk["reference_id"]
        doc_title = chunk.get("file_path", "Unknown Source")
        references[ref_id] = doc_title
    
    return "\n".join(f"[{i}] {title}" for i, title in sorted(references.items()))
```

---

## Best Practices

### 1. Token Management

```python
# Limit context size
max_context_tokens = 100000  # For large context models

# Truncate if needed
if len(tokenizer.encode(context)) > max_context_tokens:
    truncate_context(entities, relations, chunks)
```

### 2. Deduplication

```python
# Remove duplicate entities/relations
entities = deduplicate_by_name(entities)
relations = deduplicate_by_source_target(relations)
```

### 3. Relevance Filtering

```python
# Only include relevant items
filtered_entities = [e for e in entities if e["similarity"] > 0.7]
```

---

## Связь с Промптами

```
P8 (Templates) format context for:
    ├─ P6 (RAG Response)
    └─ P7 (Naive Response)
```

**См. также**:
- [P6: RAG Response](06-rag-response.md)
- [P7: Naive Response](07-naive-rag-response.md)
- [T9: Context Assembly](../transform/02-query-transforms.md#t9)

