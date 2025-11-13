# P5: Keywords Extraction Prompt

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P5 |
| **Имя** | `keywords_extraction` |
| **Тип** | Task Prompt |
| **Трансформация** | T7 (Keyword Extraction) |
| **Output Format** | JSON |
| **Temperature** | 0.0 (deterministic) |
| **Расположение** | `lightrag/prompt.py:360-381` |

---

## Назначение

Извлечение **high-level** и **low-level** keywords из user query для поиска entities в Knowledge Graph. Ключевой компонент **Local** и **Global** search modes.

---

## Роль в Query Chain

```
Query Pipeline - T7: Keyword Extraction
════════════════════════════════════════

User Query: "How does AI impact healthcare innovation?"
    ↓
[P5: Keywords Extraction] ← THIS PROMPT
    ↓
Output (JSON):
{
  "high_level_keywords": ["AI impact", "Healthcare innovation", "Technology in medicine"],
  "low_level_keywords": ["Artificial intelligence", "Medical AI", "Healthcare AI", "Innovation"]
}
    ↓
T8: Entity Resolution
    • Search entity_vdb with high_level for entity candidates
    • Search chunks_vdb with low_level for relevant chunks
```

---

## Полный Текст

```python
PROMPTS["keywords_extraction"] = """---Role---
You are an expert keyword extractor, specializing in analyzing user queries for a Retrieval-Augmented Generation (RAG) system. Your purpose is to identify both high-level and low-level keywords in the user's query that will be used for effective document retrieval.

---Goal---
Given a user query, your task is to extract two distinct types of keywords:
1. **high_level_keywords**: for overarching concepts or themes, capturing user's core intent, the subject area, or the type of question being asked.
2. **low_level_keywords**: for specific entities or details, identifying the specific entities, proper nouns, technical jargon, product names, or concrete items.

---Instructions & Constraints---
1. **Output Format**: Your output MUST be a valid JSON object and nothing else. Do not include any explanatory text, markdown code fences (like ```json), or any other text before or after the JSON. It will be parsed directly by a JSON parser.
2. **Source of Truth**: All keywords must be explicitly derived from the user query, with both high-level and low-level keyword categories are required to contain content.
3. **Concise & Meaningful**: Keywords should be concise words or meaningful phrases. Prioritize multi-word phrases when they represent a single concept. For example, from "latest financial report of Apple Inc.", you should extract "latest financial report" and "Apple Inc." rather than "latest", "financial", "report", and "Apple".
4. **Handle Edge Cases**: For queries that are too simple, vague, or nonsensical (e.g., "hello", "ok", "asdfghjkl"), you must return a JSON object with empty lists for both keyword types.

---Examples---
{examples}

---Real Data---
User Query: {query}

---Output---
Output:"""
```

---

## Параметры

| Параметр | Тип | Описание |
|----------|-----|----------|
| `{examples}` | str | Few-shot примеры (3) |
| `{query}` | str | User query |

---

## Output Format

### JSON Structure

```json
{
  "high_level_keywords": ["string", "string", ...],
  "low_level_keywords": ["string", "string", ...]
}
```

### Keyword Types

**High-Level** (conceptual, abstract):
- Core intent, subject area
- Type of question
- Overarching themes
- Examples: "International trade", "Environmental consequences", "Economic impact"

**Low-Level** (specific, concrete):
- Entities, proper nouns
- Technical jargon
- Product/service names
- Specific items
- Examples: "Apple Inc.", "Carbon emissions", "Tariffs"

---

## Examples

### Example 1: Complex Query

```
Query: "How does international trade influence global economic stability?"

Output:
{
  "high_level_keywords": ["International trade", "Global economic stability", "Economic impact"],
  "low_level_keywords": ["Trade agreements", "Tariffs", "Currency exchange", "Imports", "Exports"]
}
```

### Example 2: Technical Query

```
Query: "What are the environmental consequences of deforestation on biodiversity?"

Output:
{
  "high_level_keywords": ["Environmental consequences", "Deforestation", "Biodiversity loss"],
  "low_level_keywords": ["Species extinction", "Habitat destruction", "Carbon emissions", "Rainforest", "Ecosystem"]
}
```

### Example 3: Edge Case

```
Query: "hello"

Output:
{
  "high_level_keywords": [],
  "low_level_keywords": []
}
```

---

## Usage in Search

### Entity Resolution (T8)

```python
# Use high_level keywords for entity search
keywords = extract_keywords(query)  # P5

entity_candidates = await entity_vdb.query(
    keywords["high_level_keywords"],
    top_k=10
)

# Also use low_level for chunk search
relevant_chunks = await chunks_vdb.query(
    keywords["low_level_keywords"],
    top_k=20
)
```

### Mode-Specific Usage

```python
query_mode_usage = {
    "naive": "Skip P5 - direct chunk vector search",
    "local": "P5 → high_level for entities (depth 1-2)",
    "global": "P5 → high+low for entities + communities (depth 3+)",
    "hybrid": "P5 → adaptive based on query complexity"
}
```

---

## Performance

### Metrics

```python
extraction_quality = {
    "intent_accuracy": 0.90,   # Correct query understanding
    "coverage": 0.85,          # All aspects covered
    "specificity": 0.80,       # Appropriate granularity
    "json_compliance": 0.98    # Valid JSON output
}

latency = {
    "gpt-4o-mini": "0.5-1.5s",
    "gpt-4o": "1-3s"
}

cost_per_query = {
    "gpt-4o-mini": "$0.0005-0.002",
    "gpt-4o": "$0.002-0.01"
}
```

---

## External Dependencies

### JSON Parsing

```python
import json

def parse_keywords(llm_response: str) -> dict:
    """Parse LLM output as JSON"""
    try:
        keywords = json.loads(llm_response.strip())
        
        # Validate structure
        if "high_level_keywords" not in keywords or "low_level_keywords" not in keywords:
            raise ValueError("Missing required keys")
        
        return keywords
    
    except json.JSONDecodeError:
        # Fallback: extract JSON from text
        import re
        json_match = re.search(r'\{[^\}]+\}', llm_response, re.DOTALL)
        if json_match:
            return json.loads(json_match.group(0))
        
        # Return empty keywords
        return {"high_level_keywords": [], "low_level_keywords": []}
```

---

## Best Practices

### 1. Phrase Extraction

```python
# ✅ Good: Multi-word phrases
"latest financial report", "Apple Inc.", "AI impact"

# ❌ Bad: Single words
"latest", "financial", "report", "Apple", "Inc"
```

### 2. Edge Case Handling

```python
edge_cases = {
    "too_short": "hello" → empty lists,
    "nonsense": "asdfghjkl" → empty lists,
    "ambiguous": "it" → try to infer from context
}
```

### 3. Language Adaptation

```python
# Query language determines keyword language
query_languages = {
    "English": "Keywords in English",
    "Russian": "Keywords in Russian",
    "Chinese": "Keywords in Chinese"
}
```

---

## Связь с Трансформациями

```
T7: Keywords → T8: Entity Resolution → T9: Context Assembly → T10: Answer
```

**См. также**:
- [T7: Keyword Extraction](../transform/02-query-transforms.md#t7)
- [P6: RAG Response](06-rag-response.md)

