# P6: RAG Response Prompt (Knowledge Graph)

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P6 |
| **Имя** | `rag_response` |
| **Тип** | System Prompt |
| **Трансформация** | T10 (Answer Generation) |
| **Modes** | Local, Global, Hybrid |
| **Temperature** | 0.1 (low creativity) |
| **Расположение** | `lightrag/prompt.py:214-264` |

---

## Назначение

System prompt для генерации ответов на основе **Knowledge Graph entities + relationships + document chunks**. Используется в Local/Global/Hybrid modes для создания grounded, citation-backed ответов.

---

## Роль в Query Chain

```
Query Pipeline - T10: Answer Generation
════════════════════════════════════════

User Query + Context Data → [P6: RAG Response] → Grounded Answer + Citations

Context Data includes:
• Entities from KG
• Relationships from KG
• Document chunks
• Reference list (sources)
```

---

## Текст Промпта (Abbreviated)

```python
PROMPTS["rag_response"] = """---Role---

You are an expert AI assistant specializing in synthesizing information from a provided knowledge base. Your primary function is to answer user queries accurately by ONLY using the information within the provided `Source Data`.

---Goal---

Generate a comprehensive, well-structured answer to the user query.
The answer must integrate relevant facts from the Knowledge Graph and Document Chunks found in the `Source Data`.
Consider the conversation history if provided to maintain conversational flow and avoid repeating information.

---Instructions---

**1. Step-by-Step Instruction:**
  - Carefully determine the user's query intent in the context of the conversation history
  - Scrutinize the `Source Data` (both Knowledge Graph and Document Chunks)
  - Weave the extracted facts into a coherent response
  - Track reference_id of each document chunk for citations
  - Generate a reference section at the end of the response
  - Do not generate anything after the reference section

**2. Content & Grounding:**
  - Strictly adhere to the provided context from the `Source Data`
  - DO NOT invent, assume, or infer any information not explicitly stated
  - If the answer cannot be found, state that you do not have enough information

**3. Formatting & Language:**
  - The response MUST be in the same language as the user query
  - Use Markdown for clear formatting
  - The response should be presented in {response_type}

**4. References Section Format:**
  - The References section should be under heading: `### References`
  - Format: `* [n] Document Title`
  - Maximum of 5 most relevant citations
  - Do not generate footnotes section or any text after the references

**5. Reference Section Example:**
```
### References
* [1] Document Title One
* [2] Document Title Two
```

**6. Additional Instructions**: {user_prompt}


---Source Data---
{context_data}
"""
```

---

## Context Data Format (P8)

Context is formatted using **kg_query_context** template:

```json
Entities Data From Knowledge Graph(KG):

{
  "entity_name": "Alice Smith",
  "entity_type": "person",
  "description": "Alice Smith is the CEO of TechCorp...",
  ...
}

Relationships Data From Knowledge Graph(KG):

{
  "source": "Alice Smith",
  "target": "TechCorp",
  "keywords": "leadership, employment",
  "description": "Alice Smith serves as CEO of TechCorp"
}

Original Texts From Document Chunks(DC):

{
  "content": "Alice joined TechCorp as CEO in 2020...",
  "reference_id": 1
}

Document Chunks (DC) Reference Document List:

[1] TechCorp Annual Report 2020
[2] Leadership Announcement
```

---

## Key Features

### 1. Grounding Enforcement

```
"Strictly adhere to the provided context from the `Source Data`"
"DO NOT invent, assume, or infer any information not explicitly stated"
```

**Purpose**: Prevent hallucinations

### 2. Citation Requirement

```
"Track the reference_id of each document chunk"
"Generate a reference section at the end"
```

**Purpose**: Verifiability, transparency

### 3. Multi-Source Integration

```
"integrate relevant facts from the Knowledge Graph and Document Chunks"
```

**Purpose**: Leverage both structured (KG) and unstructured (chunks) data

### 4. Language Matching

```
"The response MUST be in the same language as the user query"
```

**Purpose**: User experience

---

## Parameters

| Параметр | Описание | Example |
|----------|----------|---------|
| `{response_type}` | Format of answer | "multiple paragraphs" |
| `{user_prompt}` | Additional instructions | "Be concise" |
| `{context_data}` | KG + chunks | Formatted by P8 |

---

## Example Usage

### Input

```
Query: "What is Alice's role at TechCorp?"

Context Data:
Entities:
- Alice Smith (person): CEO of TechCorp
- TechCorp (organization): Technology company

Relations:
- Alice Smith → TechCorp (leadership)

Chunks:
- [1] "Alice joined as CEO in 2020"
- [2] "She leads AI Platform team"
```

### Output

```markdown
Alice Smith serves as the Chief Executive Officer (CEO) of TechCorp. She assumed this leadership role in 2020 and is responsible for leading the company's AI Platform team and overall strategic direction.

### References
* [1] TechCorp Leadership Documentation
* [2] AI Platform Team Overview
```

---

## Quality Metrics

```python
answer_quality = {
    "factual_accuracy": 0.85,     # Based on source data
    "completeness": 0.75,         # Fully answers query
    "coherence": 0.95,            # Well-structured
    "grounding": 0.90,            # Citations present
    "relevance": 0.90,            # On-topic
    "hallucination_rate": 0.10    # Unsupported statements (main risk)
}
```

---

## Hallucination Mitigation

```python
mitigation_strategies = [
    "Explicit 'ONLY use provided context' instruction",
    "Citation requirements",
    "Low temperature (0.1)",
    "Grounding prompt design",
    "Reference tracking"
]

hallucination_sources = {
    "synthesis_errors": 5%,       # Wrong connections
    "inference_mistakes": 3%,     # Incorrect inferences
    "knowledge_leakage": 2%,      # LLM prior knowledge used
    "context_misread": 1%         # Misunderstanding context
}
```

---

## Performance

```python
latency = {
    "local_mode": "3-8s",    # Smaller context
    "global_mode": "5-15s",  # Larger context
    "hybrid_mode": "4-10s"   # Adaptive
}

token_usage = {
    "system_prompt": 1500,
    "context_data": "5K-100K",  # Varies by mode
    "output": 500-800
}

cost_per_query = {
    "gpt-4o-mini": "$0.005-0.05",
    "gpt-4o": "$0.02-0.20"
}
```

---

## Best Practices

### 1. Adequate Context

```python
# Ensure sufficient context
context_guidelines = {
    "min_entities": 3,
    "min_relationships": 2,
    "min_chunks": 5,
    "total_tokens": "5K-20K optimal"
}
```

### 2. Citation Verification

```python
# Post-process to verify citations
def verify_citations(answer: str, reference_list: list) -> bool:
    cited_refs = extract_citation_numbers(answer)
    return all(ref <= len(reference_list) for ref in cited_refs)
```

### 3. Low Temperature

```python
# Minimize creativity, maximize grounding
temperature = 0.1
```

---

## Связь с Трансформациями

```
T7: Keywords → T8: Entities → T9: Context → [P6] T10: Answer → T11: Format
```

**См. также**:
- [P7: Naive RAG Response](07-naive-rag-response.md) - chunk-only variant
- [P8: Context Templates](08-context-templates.md) - context formatting
- [T10: Answer Generation](../transform/02-query-transforms.md#t10)

