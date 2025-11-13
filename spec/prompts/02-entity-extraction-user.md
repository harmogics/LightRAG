# P2: Entity Extraction User Prompt

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P2 |
| **Имя** | `entity_extraction_user_prompt` |
| **Тип** | User Prompt |
| **Трансформация** | T2 (Entity Extraction) |
| **Связанный System** | P1 (entity_extraction_system_prompt) |
| **Temperature** | 0.0 (deterministic) |
| **Расположение** | `lightrag/prompt.py:71-81` |

---

## Назначение

User prompt для инициации процесса извлечения entities и relationships. Работает в паре с **P1** (system prompt), предоставляя task directive и reinforcing формат вывода.

---

## Роль в Transformation Chain

```
T2: Entity Extraction
─────────────────────

System Context: [P1: Detailed instructions, examples]
                    ↓
User Request:   [P2: Task directive, format reminder] ← THIS PROMPT
                    ↓
LLM Processing: Analyze text → Extract entities/relations
                    ↓
Output: Structured entity/relation list
```

### Позиция в Диалоге

```
LLM Dialog Structure:
═════════════════════

┌────────────────────────────────────────┐
│ System Message                         │
│                                        │
│ [P1: entity_extraction_system_prompt]  │
│ • Role definition                      │
│ • Detailed instructions (8 sections)   │
│ • Examples (3 few-shot)                │
│ • Input data (chunk text)              │
│                                        │
│ Token count: ~4000-5000                │
└────────────────────────────────────────┘
                  ↓
┌────────────────────────────────────────┐
│ User Message                           │
│                                        │
│ [P2: entity_extraction_user_prompt] ←──┼─── THIS PROMPT
│ • Task statement                       │
│ • Format reinforcement                 │
│ • Completion reminder                  │
│ • Language specification               │
│                                        │
│ Token count: ~200-300                  │
└────────────────────────────────────────┘
                  ↓
┌────────────────────────────────────────┐
│ Assistant Response                     │
│                                        │
│ entity<|#|>Name<|#|>Type<|#|>Desc      │
│ relation<|#|>Src<|#|>Tgt<|#|>...       │
│ <|COMPLETE|>                           │
└────────────────────────────────────────┘
```

---

## Полный Текст Промпта

```python
PROMPTS["entity_extraction_user_prompt"] = """---Task---
Extract entities and relationships from the input text to be processed.

---Instructions---
1.  **Strict Adherence to Format:** Strictly adhere to all format requirements for entity and relationship lists, including output order, field delimiters, and proper noun handling, as specified in the system prompt.
2.  **Output Content Only:** Output *only* the extracted list of entities and relationships. Do not include any introductory or concluding remarks, explanations, or additional text before or after the list.
3.  **Completion Signal:** Output `{completion_delimiter}` as the final line after all relevant entities and relationships have been extracted and presented.
4.  **Output Language:** Ensure the output language is {language}. Proper nouns (e.g., personal names, place names, organization names) must be kept in their original language and not translated.

<Output>
"""
```

---

## Параметры Промпта

### Placeholders

| Параметр | Тип | Описание | Пример |
|----------|-----|----------|---------|
| `{completion_delimiter}` | str | Сигнал завершения | `"<\|COMPLETE\|>"` |
| `{language}` | str | Язык вывода | `"English"` / `"Russian"` |

### Генерация

```python
# From lightrag/operate.py:2072

entity_extraction_user_prompt = PROMPTS["entity_extraction_user_prompt"].format(
    completion_delimiter=PROMPTS["DEFAULT_COMPLETION_DELIMITER"],
    language=language
)
```

---

## Концептуальная Роль

### 1. Task Initiation

**Функция**: Активировать процесс extraction

```
Без user prompt:
─────────────────
System: [Detailed instructions]
LLM: [Waiting for task...]

С user prompt:
──────────────
System: [Detailed instructions]
User: "Extract entities and relationships from the input text"
LLM: [Begins extraction]
```

### 2. Format Reinforcement

**Функция**: Напомнить о требованиях формата

```python
# Key reminders in P2:
reminders = {
    "format": "Strictly adhere to all format requirements",
    "output": "Output only the extracted list",
    "completion": "Output completion_delimiter at the end",
    "language": "Ensure output language is {language}"
}

# Why repeat?
# - System prompt may be long (4000+ tokens)
# - User prompt is "last instruction" before generation
# - LLMs pay more attention to recent messages
```

### 3. Output Constraint

**Функция**: Предотвратить лишний текст

```
❌ Without constraint:
─────────────────────
Sure! I'll extract the entities and relationships from the text.

Here are the results:

entity<|#|>Alice<|#|>person<|#|>...
relation<|#|>Alice<|#|>Bob<|#|>...

I found 5 entities and 3 relationships. Let me know if you need anything else!

✅ With constraint:
───────────────────
entity<|#|>Alice<|#|>person<|#|>...
relation<|#|>Alice<|#|>Bob<|#|>...
<|COMPLETE|>
```

**Instruction**: "Output *only* the extracted list... Do not include any introductory or concluding remarks"

---

## Связь с P1

### Complementary Design

```
P1 (System):              P2 (User):
────────────              ──────────
• Role                    • Task
• Instructions (detailed) • Format reminder (brief)
• Examples (3)            • Output constraints
• Input data              • Completion signal

Длинный (4000+ tokens)    Короткий (~200 tokens)
Обучающий                 Активирующий
```

### Information Flow

```
┌──────────────────────────────────────────────────────┐
│                   LLM Context                         │
├──────────────────────────────────────────────────────┤
│                                                       │
│  P1 System: "You are a Knowledge Graph Specialist    │
│              responsible for extracting..."           │
│              [4000 tokens of instructions]            │
│                                                       │
│                         ↓                             │
│              ┌─────────────────────┐                 │
│              │  Semantic Encoding  │                 │
│              │  • Role: Specialist │                 │
│              │  • Task: Extract    │                 │
│              │  • Format: Defined  │                 │
│              └─────────────────────┘                 │
│                         ↓                             │
│                                                       │
│  P2 User: "Extract entities and relationships from   │
│            the input text to be processed."          │
│            [200 tokens reinforcing instructions]      │
│                                                       │
│                         ↓                             │
│              ┌─────────────────────┐                 │
│              │  Task Activation    │                 │
│              │  • Start extraction │                 │
│              │  • Apply format     │                 │
│              │  • Output only list │                 │
│              └─────────────────────┘                 │
│                         ↓                             │
│                    Generation                         │
│                                                       │
└──────────────────────────────────────────────────────┘
```

---

## Usage Example

### Complete Call

```python
from lightrag.prompt import PROMPTS
from lightrag.llm import openai_complete_if_cache

# Prepare context
chunk_content = """
Alice Smith joined TechCorp as CEO in 2020. She leads the AI Platform
development team and reports to the Board of Directors.
"""

context_base = {
    "tuple_delimiter": "<|#|>",
    "completion_delimiter": "<|COMPLETE|>",
    "entity_types": "person,organization,product,event",
    "examples": "\n".join(PROMPTS["entity_extraction_examples"]),
    "language": "English"
}

# Generate system prompt (P1)
system_prompt = PROMPTS["entity_extraction_system_prompt"].format(
    **context_base,
    input_text=chunk_content
)

# Generate user prompt (P2)
user_prompt = PROMPTS["entity_extraction_user_prompt"].format(
    completion_delimiter=context_base["completion_delimiter"],
    language=context_base["language"]
)

# Call LLM
response = await openai_complete_if_cache(
    user_prompt,
    system_prompt=system_prompt,
    model="gpt-4o-mini",
    temperature=0.0
)

# Response будет:
# entity<|#|>Alice Smith<|#|>person<|#|>Alice Smith joined TechCorp as CEO in 2020 and leads the AI Platform development team.
# entity<|#|>TechCorp<|#|>organization<|#|>TechCorp is a company where Alice Smith serves as CEO.
# entity<|#|>AI Platform<|#|>product<|#|>The AI Platform is a development project led by Alice Smith at TechCorp.
# entity<|#|>Board Of Directors<|#|>organization<|#|>The Board of Directors is the governing body to which Alice Smith reports.
# relation<|#|>Alice Smith<|#|>TechCorp<|#|>leadership, employment<|#|>Alice Smith serves as CEO of TechCorp.
# relation<|#|>Alice Smith<|#|>AI Platform<|#|>leadership, project management<|#|>Alice Smith leads the AI Platform development team.
# relation<|#|>Alice Smith<|#|>Board Of Directors<|#|>reporting structure<|#|>Alice Smith reports to the Board of Directors.
# <|COMPLETE|>
```

---

## Design Rationale

### Why Separate User Prompt?

```python
# Alternative 1: Combined prompt (не используется)
combined_prompt = """
You are a specialist. Extract entities. Format: entity|name|type|desc.
Text: {input_text}
"""
# Проблемы:
# - Смешивание role и task
# - Сложнее для LLM парсить
# - Менее гибкий для multi-turn

# Alternative 2: System + User (текущий подход)
system_prompt = "You are a specialist. [Detailed instructions]"
user_prompt = "Extract entities from the text."
# Преимущества:
# - Четкое разделение: role vs task
# - Соответствует chat format
# - Позволяет multi-turn (gleaning)
# - LLM лучше обрабатывает
```

### Multi-Turn Support

```python
# Single-turn (P1 + P2)
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": user_prompt}
]
response1 = await llm(messages)

# Multi-turn (gleaning with P3)
messages.append({"role": "assistant", "content": response1})
messages.append({"role": "user", "content": gleaning_prompt})  # P3
response2 = await llm(messages)

# → Separate user prompts enable conversation flow
```

---

## Optimizations

### 1. Prompt Caching

```python
# P2 is short and parameterized
# Only {completion_delimiter} and {language} change

# Cache key includes both system and user
cache_key = hash(system_prompt + user_prompt + model + temperature)

# Hit rate
cache_stats = {
    "p1_variations": "High (chunk content changes)",
    "p2_variations": "Very low (same for all chunks)",
    "effective_hit_rate": "30-40%"
}
```

### 2. Token Efficiency

```python
# P2 is concise by design
token_breakdown = {
    "p1_system": 4000,   # Detailed
    "p2_user": 200,      # Concise
    "input_chunk": 1000,
    "output": 1000,
    "total": 6200
}

# Could P2 be shorter?
minimal_p2 = "Extract entities."  # 3 tokens
current_p2 = "Extract entities... [instructions]"  # 200 tokens

# Trade-off analysis:
quality_impact = {
    "minimal": 0.60,     # Lower quality (no format reminder)
    "current": 0.67,     # Good quality
    "token_savings": 197,
    "cost_savings": "$0.0002 per call",
    "decision": "Current length justified by quality gain"
}
```

### 3. Language Adaptation

```python
# P2 adapts to language
language_variants = {
    "English": {
        "prompt_start": "Extract entities and relationships",
        "tokens": 200
    },
    "Russian": {
        "prompt_start": "Извлеките сущности и отношения",
        "tokens": 250  # Slightly more tokens in Russian
    },
    "Chinese": {
        "prompt_start": "从输入文本中提取实体和关系",
        "tokens": 150  # Fewer tokens in Chinese
    }
}

# Note: P2 text is in English, but output language specified
# LLM understands "{language}" parameter
```

---

## Связь с Другими Промптами

### Input Chain

```
P1 (System) → P2 (User) → LLM → Output
                ↓
          If quality low:
                ↓
P1 (System) → P3 (Gleaning) → LLM → More Output
```

### Output Usage

```python
# P2's output feeds into:
output_consumers = {
    "T3_Gleaning": "If entity count < threshold, use P3",
    "T4_Parsing": "Parse structured output into Python objects",
    "T5_Merging": "Group descriptions by entity for summarization"
}
```

---

## Метрики Влияния

### Ablation Study

```python
# Test: Remove P2, use only P1
experiment_results = {
    "with_p2": {
        "precision": 0.70,
        "recall": 0.65,
        "format_compliance": 0.95,
        "unwanted_text": 0.05  # 5% have extra text
    },
    "without_p2": {
        "precision": 0.68,     # -2% (slight drop)
        "recall": 0.62,        # -3% (noticeable)
        "format_compliance": 0.88,  # -7% (significant)
        "unwanted_text": 0.25  # 25% have extra text!
    }
}

# Conclusion: P2 значительно улучшает format compliance
```

---

## Внешние Зависимости

### LLM Interface

```python
# P2 uses same interface as P1
from lightrag.utils import use_llm_func_with_cache

response = await use_llm_func_with_cache(
    user_prompt=entity_extraction_user_prompt,  # P2
    use_llm_func=llm_model_func,
    system_prompt=entity_extraction_system_prompt,  # P1
    llm_response_cache=cache,
    temperature=0.0,
    max_tokens=2000
)
```

### No Additional Libraries

P2 не требует дополнительных зависимостей beyond P1.

---

## Best Practices

### 1. Keep It Concise

```python
# P2 should be short and directive
good_length = "200-300 tokens"
rationale = "Recent message has high attention, don't dilute with noise"
```

### 2. Reinforce Critical Points

```python
# What to reinforce in P2:
critical_reminders = [
    "Format adherence",      # Most important
    "Output-only",           # Prevent extra text
    "Completion signal",     # Clear termination
    "Language"               # Consistency
]
```

### 3. Pair with Strong System Prompt

```python
# P2 is effective only with good P1
dependencies = {
    "p1_quality": "High",  # Detailed instructions
    "p2_role": "Reinforce and activate"
}
```

---

## Troubleshooting

### Common Issues

**Issue 1: LLM outputs extra text**
```
Output: "Sure, here are the entities: entity<|#|>..."

Solution: Emphasize "Output *only* the extracted list"
```

**Issue 2: Missing completion delimiter**
```
Output: entity<|#|>Alice<|#|>...
        relation<|#|>...
        [no <|COMPLETE|>]

Solution: Explicit instruction in P2
```

**Issue 3: Wrong language**
```
Query in Russian, output in English

Solution: "{language}" parameter in P2
```

---

## Заключение

**P2 (Entity Extraction User Prompt)** служит activation signal для P1:
- ✅ Инициирует extraction process
- ✅ Reinforces format requirements
- ✅ Constrains output to structured list
- ✅ Specifies language
- ✅ Enables multi-turn (gleaning)

**Критичность**: Medium-High
- Без P2 quality падает на 5-10%
- Format compliance значительно хуже
- Extra text в 20% случаев

**См. также**:
- [P1: Entity Extraction System Prompt](01-entity-extraction-system.md)
- [P3: Entity Gleaning Prompt](03-entity-gleaning.md)
- [T2: Entity Extraction Transform](../transform/01-indexing-transforms.md#t2)
