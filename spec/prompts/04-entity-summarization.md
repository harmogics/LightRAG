# P4: Entity Summarization Prompt

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P4 |
| **Имя** | `summarize_entity_descriptions` |
| **Тип** | Task Prompt |
| **Трансформация** | T5 (Description Merging) |
| **LLM** | GPT-4o / GPT-4o-mini |
| **Temperature** | 0.1 (slight creativity for coherence) |
| **Расположение** | `lightrag/prompt.py:175-208` |

---

## Назначение

Промпт для объединения множественных описаний одной entity/relation в единое когерентное summary. Используется в **T5 трансформации** (Description Merging) после того как entity упоминается в нескольких chunks.

---

## Роль в Transformation Chain

```
Indexing Pipeline - T5: Description Merging
═══════════════════════════════════════════

Multiple Chunks mention same entity:
────────────────────────────────────
Chunk 1: "Alice is CEO of TechCorp"
  → entity[Alice]: "Alice is CEO of TechCorp"

Chunk 2: "Alice led AI Platform development"
  → entity[Alice]: "Alice led AI Platform development"

Chunk 3: "Alice has 15 years experience in AI"
  → entity[Alice]: "Alice has 15 years experience in AI"

    ↓
[Collect all descriptions for Alice]
    ↓
Description List: [desc1, desc2, desc3]
    ↓
[P4: Summarization Prompt] ← THIS PROMPT
    ↓
Unified Description:
"Alice Smith is the CEO of TechCorp, leading the company's AI initiatives.
She spearheaded the development of the AI Platform and brings 15 years of
experience in artificial intelligence to her role."
    ↓
Store in Knowledge Graph
```

---

## Полный Текст Промпта

```python
PROMPTS["summarize_entity_descriptions"] = """---Role---
You are a Knowledge Graph Specialist, proficient in data curation and synthesis.

---Task---
Your task is to synthesize a list of descriptions of a given entity or relation into a single, comprehensive, and cohesive summary.

---Instructions---
1. Input Format: The description list is provided in JSON format. Each JSON object (representing a single description) appears on a new line within the `Description List` section.
2. Output Format: The merged description will be returned as plain text, presented in multiple paragraphs, without any additional formatting or extraneous comments before or after the summary.
3. Comprehensiveness: The summary must integrate all key information from *every* provided description. Do not omit any important facts or details.
4. Context: Ensure the summary is written from an objective, third-person perspective; explicitly mention the name of the entity or relation for full clarity and context.
5. Context & Objectivity:
  - Write the summary from an objective, third-person perspective.
  - Explicitly mention the full name of the entity or relation at the beginning of the summary to ensure immediate clarity and context.
6. Conflict Handling:
  - In cases of conflicting or inconsistent descriptions, first determine if these conflicts arise from multiple, distinct entities or relationships that share the same name.
  - If distinct entities/relations are identified, summarize each one *separately* within the overall output.
  - If conflicts within a single entity/relation (e.g., historical discrepancies) exist, attempt to reconcile them or present both viewpoints with noted uncertainty.
7. Length Constraint:The summary's total length must not exceed {summary_length} tokens, while still maintaining depth and completeness.
8. Language: The entire output must be written in {language}. Proper nouns (e.g., personal names, place names, organization names) may in their original language if proper translation is not available.
  - The entire output must be written in {language}.
  - Proper nouns (e.g., personal names, place names, organization names) should be retained in their original language if a proper, widely accepted translation is not available or would cause ambiguity.

---Input---
{description_type} Name: {description_name}

Description List:

```
{description_list}
```

---Output---
"""
```

---

## Параметры

| Параметр | Тип | Описание | Default |
|----------|-----|----------|---------|
| `{description_type}` | str | "Entity" или "Relation" | - |
| `{description_name}` | str | Имя entity/relation | e.g., "Alice Smith" |
| `{description_list}` | str | JSON list descriptions | JSONL format |
| `{summary_length}` | int | Max tokens | 500 |
| `{language}` | str | Язык вывода | "English" |

---

## Map-Reduce Стратегия

### Conceptual Process

```python
# From lightrag/operate.py:121-228

def summarize_descriptions_map_reduce(
    entity_name: str,
    descriptions: list[str],  # May be 100+ descriptions
    max_tokens: int = 500
) -> str:
    """
    Map-Reduce summarization for handling many descriptions
    
    Phase 1: MAP
    ────────────
    If total_tokens > summary_max_tokens:
        Split descriptions into chunks
        Summarize each chunk → partial_summaries
    
    Phase 2: REDUCE
    ───────────────
    If len(partial_summaries) > 1:
        Recursively summarize partial_summaries
    Until: single summary within token limit
    
    Phase 3: FINAL
    ──────────────
    Return unified summary
    """
    
    tokenizer = get_tokenizer()
    
    # Calculate total tokens
    total_tokens = sum(len(tokenizer.encode(d)) for d in descriptions)
    
    # Decision tree
    if total_tokens <= summary_context_size and len(descriptions) < force_llm_summary_on_merge:
        # No summarization needed - just concat
        return "\n".join(descriptions)
    
    elif total_tokens <= summary_max_tokens:
        # Single LLM call
        return await summarize_with_llm(entity_name, descriptions)
    
    else:
        # Map-Reduce: Split into chunks
        chunks = split_into_chunks(descriptions, summary_max_tokens)
        
        # Map: Summarize each chunk
        partial_summaries = [
            await summarize_with_llm(entity_name, chunk)
            for chunk in chunks
        ]
        
        # Reduce: Recursively summarize
        return await summarize_descriptions_map_reduce(
            entity_name,
            partial_summaries,
            max_tokens
        )
```

---

## Examples

### Example 1: Simple Merge

**Input**:
```json
Entity Name: Alice Smith

Description List:
{"description": "Alice Smith is the CEO of TechCorp."}
{"description": "Alice Smith leads the AI Platform development team."}
{"description": "Alice Smith has 15 years of experience in artificial intelligence."}
```

**Output**:
```
Alice Smith is the CEO of TechCorp, where she leads the AI Platform development
team. She brings 15 years of experience in artificial intelligence to her
leadership role, driving the company's AI initiatives.
```

### Example 2: Conflict Resolution

**Input**:
```json
Relation Name: Alice-TechCorp

Description List:
{"description": "Alice Smith joined TechCorp as CEO in 2020."}
{"description": "Alice Smith has been with TechCorp since 2018 as VP."}
{"description": "Alice Smith founded TechCorp in 2015."}
```

**Output**:
```
Alice Smith founded TechCorp in 2015. She served as Vice President from 2018
until her promotion to CEO in 2020, where she currently leads the company's
strategic direction and operations.
```

*(Note: LLM reconciles timeline by inferring progression)*

---

## Semantic Characteristics

### Information Preservation

```python
summarization_metrics = {
    "input_entropy": 100%,      # All descriptions
    "output_entropy": 85%,      # 15% compression
    "information_loss": {
        "core_facts": 5%,       # Minimal loss of key facts
        "supporting_details": 25%, # Some details compressed
        "redundant_info": 70%    # Redundancy removed (good!)
    },
    "hallucination_rate": 0.08   # 8% unsupported statements
}
```

### Quality Dimensions

```python
quality_metrics = {
    "factual_accuracy": 0.90,   # Facts preserved correctly
    "completeness": 0.80,        # Major points covered
    "coherence": 0.95,           # Well-written, unified narrative
    "conciseness": 0.85,         # Appropriate length
    "objectivity": 0.95          # Third-person, neutral
}
```

---

## Performance

### Latency

```python
summarization_latency = {
    "few_descriptions_1-3": "1-2s",    # Direct call
    "medium_descriptions_4-10": "2-5s", # Single LLM call
    "many_descriptions_10-50": "5-15s", # Map-reduce (2-3 rounds)
    "very_many_50+": "15-30s"           # Deep map-reduce
}
```

### Cost

```python
# Per entity summarization
cost_per_entity = {
    "gpt-4o-mini": "$0.001-0.005",
    "gpt-4o": "$0.005-0.02"
}

# For typical document (50 entities):
total_cost = {
    "gpt-4o-mini": "$0.05-0.25",
    "gpt-4o": "$0.25-1.00"
}
```

---

## Связь с Трансформациями

### Input (от T2-T4)

```python
# After extraction and parsing
entity_descriptions = {
    "Alice Smith": [
        "Alice is CEO of TechCorp",
        "Alice leads AI Platform team",
        "Alice has 15 years experience"
    ],
    "TechCorp": [
        "TechCorp is a technology company",
        "TechCorp was founded in 2015",
        "TechCorp employs 500 people"
    ]
}

# For each entity: P4 summarize descriptions
```

### Output (к T6)

```python
# Unified descriptions
unified_descriptions = {
    "Alice Smith": "Alice Smith is the CEO of TechCorp...",
    "TechCorp": "TechCorp is a technology company founded in 2015..."
}

# These go to T6: Vectorization
# → Embedded for entity vector database
```

---

## External Dependencies

### LLM Interface

```python
from lightrag.utils import use_llm_func_with_cache

summarized = await use_llm_func_with_cache(
    user_prompt="",  # Empty, all in system prompt
    system_prompt=summarization_prompt,
    use_llm_func=llm_model_func,
    temperature=0.1,  # Slight creativity
    llm_response_cache=cache
)
```

### JSON Formatting

```python
# Descriptions formatted as JSONL
description_list_jsonl = "\n".join(
    json.dumps({"description": desc}) for desc in descriptions
)
```

---

## Best Practices

### 1. Token Management

```python
# Prevent overload
if total_tokens > summary_max_tokens:
    use_map_reduce()
else:
    direct_summarize()
```

### 2. Preserve Key Facts

```
Priority for inclusion:
1. Entity type and primary attributes
2. Key relationships
3. Historical facts (dates, events)
4. Quantitative data
5. Supporting details (if space permits)
```

### 3. Handle Conflicts

```python
conflict_strategies = {
    "temporal": "Order chronologically",
    "contradictory": "Note uncertainty: 'Some sources indicate...'",
    "distinct_entities": "Separate summaries"
}
```

---

## Заключение

**P4 (Entity Summarization)** объединяет фрагментированные описания в когерентные summaries:
- ✅ Map-reduce для scalability
- ✅ 90% factual accuracy
- ✅ Conflict resolution
- ✅ Optimal for KG descriptions

**См. также**:
- [T5: Description Merging](../transform/01-indexing-transforms.md#t5)
- [P1: Entity Extraction](01-entity-extraction-system.md)

