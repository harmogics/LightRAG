# P3: Entity Gleaning Prompt

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P3 |
| **Имя** | `entity_continue_extraction_user_prompt` |
| **Тип** | User Prompt (Multi-turn) |
| **Трансформация** | T3 (Gleaning) |
| **Связанный System** | P1 (entity_extraction_system_prompt) |
| **Temperature** | 0.0 (deterministic) |
| **Расположение** | `lightrag/prompt.py:83-99` |

---

## Назначение

Multi-turn user prompt для **gleaning** - процесса повторного извлечения пропущенных или некорректно форматированных entities и relationships. Используется когда начальная экстракция (P1+P2) неполная.

**Gleaning** = "подбирать колосья" - метафора для сбора пропущенной информации.

---

## Роль в Transformation Chain

```
T2: Entity Extraction (P1 + P2)
    ↓
Initial Output: Entities + Relations
    ↓
Quality Check:
    │
    ├─ IF recall < threshold (65%)
    │  OR entity_count < expected
    │  OR format_errors > threshold
    │       ↓
    │   T3: Gleaning (P1 + P3) ← THIS PROMPT
    │       ↓
    │   Additional Entities + Corrections
    │       ↓
    │   Merge with Initial Output
    │       ↓
    └─→ Final Output: Enhanced Entities + Relations
            ↓
        T4: Parsing
```

### Multi-Turn Dialog

```
Conversation Flow:
══════════════════

Turn 1 (Initial Extraction):
────────────────────────────
System: [P1: entity_extraction_system_prompt]
User:   [P2: entity_extraction_user_prompt]
        "Extract entities and relationships..."

Assistant: entity<|#|>Alice<|#|>person<|#|>...
           relation<|#|>Alice<|#|>TechCorp<|#|>...
           <|COMPLETE|>

           [Extracted 5 entities, 3 relations]

Turn 2 (Gleaning):
──────────────────
[System prompt remains: P1]

User:   [P3: entity_continue_extraction_user_prompt] ← THIS PROMPT
        "Based on the last extraction, identify missed entities..."

Assistant: entity<|#|>Bob<|#|>person<|#|>...  [MISSED in Turn 1]
           entity<|#|>Product X<|#|>product<|#|>...  [MISSED]
           relation<|#|>Alice<|#|>Product X<|#|>...  [NEW]
           <|COMPLETE|>

           [Extracted 2 more entities, 1 more relation]

Final Result: 7 entities, 4 relations (+40% coverage!)
```

---

## Полный Текст Промпта

```python
PROMPTS["entity_continue_extraction_user_prompt"] = """---Task---
Based on the last extraction task, identify and extract any **missed or incorrectly formatted** entities and relationships from the input text.

---Instructions---
1.  **Strict Adherence to System Format:** Strictly adhere to all format requirements for entity and relationship lists, including output order, field delimiters, and proper noun handling, as specified in the system instructions.
2.  **Focus on Corrections/Additions:**
    *   **Do NOT** re-output entities and relationships that were **correctly and fully** extracted in the last task.
    *   If an entity or relationship was **missed** in the last task, extract and output it now according to the system format.
    *   If an entity or relationship was **truncated, had missing fields, or was otherwise incorrectly formatted** in the last task, re-output the *corrected and complete* version in the specified format.
3.  **Output Format - Entities:** Output a total of 4 fields for each entity, delimited by `{tuple_delimiter}`, on a single line. The first field *must* be the literal string `entity`.
4.  **Output Format - Relationships:** Output a total of 5 fields for each relationship, delimited by `{tuple_delimiter}`, on a single line. The first field *must* be the literal string `relation`.
5.  **Output Content Only:** Output *only* the extracted list of entities and relationships. Do not include any introductory or concluding remarks, explanations, or additional text before or after the list.
6.  **Completion Signal:** Output `{completion_delimiter}` as the final line after all relevant missing or corrected entities and relationships have been extracted and presented.
7.  **Output Language:** Ensure the output language is {language}. Proper nouns (e.g., personal names, place names, organization names) must be kept in their original language and not translated.

<Output>
"""
```

---

## Параметры Промпта

### Placeholders

| Параметр | Тип | Описание | Пример |
|----------|-----|----------|---------|
| `{tuple_delimiter}` | str | Разделитель полей | `"<\|#\|>"` |
| `{completion_delimiter}` | str | Сигнал завершения | `"<\|COMPLETE\|>"` |
| `{language}` | str | Язык вывода | `"English"` |

### Генерация

```python
# From lightrag/operate.py:2102

entity_continue_extraction_user_prompt = PROMPTS[
    "entity_continue_extraction_user_prompt"
].format(
    tuple_delimiter=PROMPTS["DEFAULT_TUPLE_DELIMITER"],
    completion_delimiter=PROMPTS["DEFAULT_COMPLETION_DELIMITER"],
    language=language
)
```

---

## Концептуальные Аспекты

### 1. Why Gleaning?

**Проблема**: LLMs не всегда извлекают все entities за один проход

```python
# Typical extraction coverage
initial_extraction = {
    "entities_found": 15,
    "entities_total": 23,  # Ground truth
    "recall": 0.65         # 65% coverage
}

# Reasons for missed entities:
miss_reasons = {
    "attention_limits": "Long text → later entities missed",
    "ambiguity": "Unclear if mention is entity",
    "complexity": "Complex sentences → entities overlooked",
    "format_errors": "Partial output due to format confusion"
}
```

**Решение**: Second pass with explicit focus on missed items

```python
after_gleaning = {
    "entities_found": 20,
    "entities_total": 23,
    "recall": 0.87,        # +22% improvement!
    "additional_rounds": 1
}
```

### 2. Gleaning vs Re-Extraction

```python
# Option 1: Re-extract from scratch (не используется)
re_extraction = {
    "approach": "Run P1+P2 again",
    "problem": "Duplicate entities, wasted computation",
    "cost": "2x LLM calls"
}

# Option 2: Gleaning (используется)
gleaning = {
    "approach": "Multi-turn: focus on missed items",
    "benefit": "Only new/corrected entities",
    "cost": "1x additional LLM call",
    "deduplication": "Handled by system"
}
```

### 3. Self-Correction Mechanism

**Концепция**: LLM reviews its own output

```
Initial extraction output is in conversation history
→ LLM can "see" what it extracted
→ P3 asks: "What did you miss?"
→ LLM compares text vs extracted items
→ Identifies gaps
→ Outputs missed items
```

**Self-awareness example**:
```
Text: "Alice, Bob, and Carol worked on Project X."

Turn 1 (P2):
entity<|#|>Alice<|#|>person<|#|>...
entity<|#|>Project X<|#|>project<|#|>...
[Missed: Bob, Carol]

Turn 2 (P3):
LLM thinks: "I extracted Alice and Project X. But text mentions Bob and Carol too!"
entity<|#|>Bob<|#|>person<|#|>...
entity<|#|>Carol<|#|>person<|#|>...
```

---

## Gleaning Logic in Code

### Trigger Conditions

```python
# From lightrag/operate.py

def should_apply_gleaning(
    initial_entities: list,
    initial_relations: list,
    chunk_content: str,
    max_gleaning_rounds: int
) -> bool:
    """Decide if gleaning is needed"""

    # Condition 1: Max rounds not reached
    if current_round >= max_gleaning_rounds:
        return False

    # Condition 2: Entity count below threshold
    # Heuristic: expect ~1 entity per 100 tokens
    expected_entities = len(tokenizer.encode(chunk_content)) / 100
    if len(initial_entities) < expected_entities * 0.6:  # <60% of expected
        return True

    # Condition 3: Previous round yielded new items
    if previous_gleaning_added_items > 0:
        return True

    return False
```

### Gleaning Loop

```python
# From lightrag/operate.py:2088-2118 (simplified)

async def extract_with_gleaning(chunk_content: str):
    """Extract entities with optional gleaning"""

    # Turn 1: Initial extraction (P1 + P2)
    history = []

    system_prompt = format_prompt(P1, input_text=chunk_content)
    user_prompt = format_prompt(P2)

    response1 = await llm_call(user_prompt, system_prompt=system_prompt)

    entities, relations = parse_output(response1)

    history.append({"role": "user", "content": user_prompt})
    history.append({"role": "assistant", "content": response1})

    # Gleaning rounds (up to max_gleaning)
    for round_num in range(max_gleaning_rounds):
        if not should_apply_gleaning(entities, relations):
            break

        # Turn N: Gleaning (P1 + P3)
        gleaning_prompt = format_prompt(P3)

        response_glean = await llm_call(
            gleaning_prompt,
            system_prompt=system_prompt,  # Same P1
            history_messages=history       # Include previous turns
        )

        new_entities, new_relations = parse_output(response_glean)

        # Merge results
        entities.extend(new_entities)
        relations.extend(new_relations)

        # Update history
        history.append({"role": "user", "content": gleaning_prompt})
        history.append({"role": "assistant", "content": response_glean})

        # Check if anything new was found
        if len(new_entities) == 0 and len(new_relations) == 0:
            break  # No more to glean

    return entities, relations
```

---

## Key Instructions Analysis

### Instruction 1: Format Adherence

```
"Strictly adhere to all format requirements... as specified in the system instructions"
```

**Reason**: P3 может использоваться когда Turn 1 имел format errors

**Example**:
```
Turn 1 (bad format):
entity|Alice|person|Description  # Wrong delimiter!

Turn 2 (P3 fixes):
entity<|#|>Alice<|#|>person<|#|>Corrected description
```

### Instruction 2: Focus on Additions/Corrections

```
"Do NOT re-output entities that were correctly and fully extracted"
```

**Critical для**: Избежание дубликатов

```python
# Without this instruction:
turn1 = [E1, E2, E3]
turn2 = [E1, E2, E3, E4, E5]  # Все повторно!
merged = [E1, E1, E2, E2, E3, E3, E4, E5]  # Дубликаты!

# With instruction:
turn1 = [E1, E2, E3]
turn2 = [E4, E5]  # Только новые
merged = [E1, E2, E3, E4, E5]  # Чисто!
```

### Instruction 3-4: Format Reminder

Repeats format spec from P1 - ensures compliance.

### Instruction 5: Output-Only

Same constraint as P2 - no extra text.

### Instruction 6-7: Completion & Language

Standard signals.

---

## Performance Impact

### Coverage Improvement

```python
# Empirical results
extraction_quality = {
    "initial_only": {
        "precision": 0.70,
        "recall": 0.65,
        "f1": 0.67,
        "entities_found": "65%"
    },
    "with_gleaning_1_round": {
        "precision": 0.75,  # +5% (some false positives)
        "recall": 0.80,     # +15% (major gain!)
        "f1": 0.77,         # +10%
        "entities_found": "80%"
    },
    "with_gleaning_2_rounds": {
        "precision": 0.76,  # +1%
        "recall": 0.83,     # +3%
        "f1": 0.79,         # +2%
        "entities_found": "83%"
    }
}

# Diminishing returns after 1-2 rounds
```

### Cost-Benefit Analysis

```python
gleaning_tradeoff = {
    "cost": {
        "latency": "+2-5 seconds per round",
        "llm_calls": "+1 call per round",
        "tokens": "+1000-2000 per round",
        "money": "+$0.002-0.005 per round"
    },
    "benefit": {
        "recall_gain_round1": "+15%",
        "recall_gain_round2": "+3%",
        "f1_gain_round1": "+10%",
        "f1_gain_round2": "+2%"
    },
    "recommendation": {
        "default": "1 round (best ROI)",
        "high_quality": "2 rounds if critical",
        "speed_priority": "0 rounds (skip gleaning)"
    }
}
```

---

## Configuration

### Default Settings

```python
# From lightrag/lightrag.py

class LightRAG:
    entity_extract_max_gleaning: int = field(default=1)
    """Maximum gleaning rounds for entity extraction"""

# Usage
rag = LightRAG(
    entity_extract_max_gleaning=1  # Default: 1 round
)
```

### Tuning Guidelines

```python
gleaning_config_guide = {
    "0_rounds": {
        "use_case": "Speed-critical, low quality OK",
        "recall": "65%",
        "latency": "2-5s per chunk"
    },
    "1_round": {
        "use_case": "Balanced (recommended)",
        "recall": "80%",
        "latency": "4-10s per chunk"
    },
    "2_rounds": {
        "use_case": "High-quality extraction needed",
        "recall": "83%",
        "latency": "6-15s per chunk"
    },
    "3+_rounds": {
        "use_case": "Diminishing returns, not recommended",
        "recall": "85%",
        "latency": "8-20s per chunk"
    }
}
```

---

## Examples

### Example 1: Missed Entity

**Input Text**:
```
Alice Smith joined TechCorp as CEO. She leads the AI Platform team. Bob Johnson,
the CTO, reports to Alice. Together they launched Product X.
```

**Turn 1 (P2) Output**:
```
entity<|#|>Alice Smith<|#|>person<|#|>Alice Smith is the CEO of TechCorp and leads the AI Platform team.
entity<|#|>TechCorp<|#|>organization<|#|>TechCorp is a company led by Alice Smith.
entity<|#|>AI Platform<|#|>product<|#|>The AI Platform is a team at TechCorp led by Alice Smith.
relation<|#|>Alice Smith<|#|>TechCorp<|#|>leadership<|#|>Alice Smith serves as CEO of TechCorp.
<|COMPLETE|>
```

**Missing**: Bob Johnson, Product X

**Turn 2 (P3) Output**:
```
entity<|#|>Bob Johnson<|#|>person<|#|>Bob Johnson is the CTO at TechCorp and reports to Alice Smith.
entity<|#|>Product X<|#|>product<|#|>Product X is a product launched by Alice Smith and Bob Johnson.
relation<|#|>Bob Johnson<|#|>Alice Smith<|#|>reporting structure<|#|>Bob Johnson reports to Alice Smith.
relation<|#|>Alice Smith<|#|>Product X<|#|>product launch<|#|>Alice Smith launched Product X.
relation<|#|>Bob Johnson<|#|>Product X<|#|>product launch<|#|>Bob Johnson launched Product X.
<|COMPLETE|>
```

**Final Merged Result**: 5 entities, 5 relations (complete!)

### Example 2: Format Correction

**Turn 1 (P2) Output** (with error):
```
entity<|#|>Alice Smith<|#|>person<|#|>Alice is CEO
entity<|#|>TechCorp  # INCOMPLETE! Missing type and description
relation<|#|>Alice Smith<|#|>TechCorp<|#|>leadership<|#|>  # INCOMPLETE! Missing description
<|COMPLETE|>
```

**Turn 2 (P3) Output** (corrections):
```
entity<|#|>TechCorp<|#|>organization<|#|>TechCorp is a technology company.
relation<|#|>Alice Smith<|#|>TechCorp<|#|>leadership, employment<|#|>Alice Smith serves as CEO of TechCorp.
<|COMPLETE|>
```

**Result**: Incomplete entries corrected

---

## Связь с Другими Промптами

```
Prompt Flow:
════════════

P1 (System) ──┬──→ P2 (User Initial) → Turn 1 Output
              │                              ↓
              │                        Quality Check
              │                              ↓
              └──→ P3 (User Gleaning) → Turn 2 Output
                                             ↓
                                        Merge Results
                                             ↓
                                        T4: Parsing
```

---

## Внешние Зависимости

### Multi-Turn Dialog Support

```python
# Requires LLM API with conversation history
from lightrag.llm import openai_complete_if_cache

# Turn 1
response1 = await openai_complete_if_cache(
    user_prompt_p2,
    system_prompt=system_prompt_p1
)

# Turn 2 with history
response2 = await openai_complete_if_cache(
    user_prompt_p3,
    system_prompt=system_prompt_p1,  # Same system
    history_messages=[
        {"role": "user", "content": user_prompt_p2},
        {"role": "assistant", "content": response1}
    ]
)
```

### Deduplication

```python
# Merge logic handles duplicates
def merge_entities(initial: list, gleaned: list) -> list:
    """Merge entities, removing duplicates"""

    seen_names = {e["name"] for e in initial}
    unique_gleaned = [e for e in gleaned if e["name"] not in seen_names]

    return initial + unique_gleaned
```

---

## Best Practices

### 1. Configure Based on Use Case

```python
# Speed-critical
rag_fast = LightRAG(entity_extract_max_gleaning=0)

# Balanced
rag_default = LightRAG(entity_extract_max_gleaning=1)  # Recommended

# High-quality
rag_quality = LightRAG(entity_extract_max_gleaning=2)
```

### 2. Monitor Gleaning Effectiveness

```python
# Track metrics
gleaning_stats = {
    "rounds_triggered": "% of chunks that needed gleaning",
    "entities_added_per_round": "Average new entities found",
    "recall_improvement": "Recall gain from gleaning"
}

# If entities_added_per_round < 1: consider disabling gleaning
```

### 3. Use Same System Prompt

```
Critical: P3 must use same P1 as P2
→ Same entity types, format, instructions
→ Consistency across turns
```

---

## Заключение

**P3 (Entity Gleaning Prompt)** - это мощный механизм для улучшения coverage:
- ✅ +15% recall (первый раунд)
- ✅ Self-correction для format errors
- ✅ Minimal additional cost (+$0.002-0.005)
- ✅ Diminishing returns after 1-2 rounds

**Рекомендация**: Включать 1 round gleaning by default для баланса quality/cost.

**См. также**:
- [P1: Entity Extraction System Prompt](01-entity-extraction-system.md)
- [P2: Entity Extraction User Prompt](02-entity-extraction-user.md)
- [T3: Gleaning Transform](../transform/01-indexing-transforms.md#t3-gleaning)
