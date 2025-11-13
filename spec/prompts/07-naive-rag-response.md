# P7: Naive RAG Response Prompt

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P7 |
| **Имя** | `naive_rag_response` |
| **Тип** | System Prompt |
| **Трансформация** | T10 (Answer Generation) |
| **Mode** | Naive (chunk-only) |
| **Temperature** | 0.1 |
| **Расположение** | `lightrag/prompt.py:266-320` |

---

## Назначение

System prompt для генерации ответов в **Naive mode** - используя **только document chunks** без Knowledge Graph. Более простой и быстрый вариант P6.

---

## Отличия от P6

| Аспект | P6 (KG + Chunks) | P7 (Chunks Only) |
|--------|------------------|------------------|
| **Context** | KG entities + relations + chunks | Only chunks |
| **Complexity** | Higher | Lower |
| **Latency** | 5-15s | 2-5s |
| **Quality** | Higher (structured knowledge) | Lower (no relationships) |
| **Use Case** | Entity-focused queries | Simple factual queries |

---

## Текст Промпта (Key Sections)

```python
PROMPTS["naive_rag_response"] = """---Role---

You are an expert AI assistant specializing in synthesizing information from a provided knowledge base. Your primary function is to answer user queries accurately by ONLY using the information within the provided `Source Data`.

---Goal---

Generate a comprehensive, well-structured answer to the user query.
The answer must integrate relevant facts from the Document Chunks found in the `Source Data`.

---Instructions---

**1. Think Step-by-Step:**
  - Carefully determine the user's query intent
  - Scrutinize the `Source Data` (Document Chunks)
  - Weave the extracted facts into a coherent response
  - Track reference_id for citations
  - Generate a reference section at the end

**2. Content & Grounding:**
  - Strictly adhere to the provided context
  - DO NOT invent, assume, or infer any information
  - If the answer cannot be found, state that you do not have enough information

**3-5. Formatting, References, Additional Instructions**
  [Same as P6]

---Source Data---

Document Chunks:

{content_data}

"""
```

---

## Context Format

Simpler than P6 - only chunks:

```json
Original Texts From Document Chunks(DC):

{
  "content": "Alice joined TechCorp as CEO in 2020...",
  "reference_id": 1
}
{
  "content": "TechCorp develops AI platforms...",
  "reference_id": 2
}

Document Chunks (DC) Reference Document List:

[1] TechCorp Annual Report
[2] Product Documentation
```

---

## When to Use

### Naive Mode Scenarios

```python
use_naive_mode = {
    "simple_factual_queries": "What is X?",
    "speed_critical": "Need fast response",
    "no_kg_available": "Graph not built yet",
    "exploratory_queries": "Tell me about X"
}

avoid_naive_mode = {
    "entity_relationships": "How are X and Y related?",
    "multi_hop_reasoning": "X → Y → Z connections",
    "comprehensive_analysis": "Detailed entity analysis"
}
```

---

## Performance

```python
naive_mode_performance = {
    "latency": "2-5s",              # Faster than P6
    "context_size": "5K-20K",       # Smaller than P6
    "quality": 0.70,                # Lower than P6 (0.85)
    "use_case": "Simple queries",
    "cost": "$0.002-0.02"          # Lower cost
}
```

---

## Example

### Input

```
Query: "When did Alice join TechCorp?"

Context:
Chunks:
- [1] "Alice Smith joined TechCorp as CEO in January 2020"
- [2] "She brought 15 years of experience to the role"
```

### Output

```markdown
Alice Smith joined TechCorp in January 2020, when she assumed the position of CEO. She brought 15 years of experience to this leadership role.

### References
* [1] TechCorp Leadership History
```

---

## Связь с P6

```
Query Complexity
     Low ←────────────────→ High
     
     P7 (Naive)          P6 (KG)
     • Chunks only       • KG + chunks
     • Fast (2-5s)       • Slower (5-15s)
     • Simple answers    • Complex analysis
     • Lower quality     • Higher quality
```

**См. также**:
- [P6: RAG Response with KG](06-rag-response.md)
- [T10: Answer Generation](../transform/02-query-transforms.md#t10)

