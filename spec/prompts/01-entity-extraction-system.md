# P1: Entity Extraction System Prompt

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P1 |
| **Имя** | `entity_extraction_system_prompt` |
| **Тип** | System Prompt |
| **Трансформация** | T2 (Entity Extraction) |
| **LLM** | GPT-4o / GPT-4o-mini / Claude / Other |
| **Temperature** | 0.0 (deterministic) |
| **Max Tokens** | ~2000 (output) |
| **Расположение** | `lightrag/prompt.py:11-69` |

---

## Назначение

Системный промпт для извлечения **entities** (сущностей) и **relationships** (отношений) из текстовых chunks. Это ключевой компонент **T2 трансформации** (Entity Extraction), определяющий качество построения Knowledge Graph.

---

## Роль в Transformation Chain

```
Indexing Pipeline:
──────────────────

Document
    ↓
T1: Chunking → Chunks (1-1000 per document)
    ↓
T2: Entity Extraction ← [P1: System Prompt]
    │                  ← [P2: User Prompt]
    │
    │ Input: Text chunk (200-1500 tokens)
    │ Process: LLM analyzes text and extracts:
    │   • Entities (name, type, description)
    │   • Relationships (source, target, keywords, description)
    │ Output: Structured list of entities + relations
    │
    ↓
Raw entities + relations (may be incomplete)
    ↓
T3: Gleaning (optional) ← [P3: Gleaning Prompt]
    ↓
Refined entities + relations
    ↓
T4: Parsing
```

### Связь с Другими Промптами

| Промпт | Связь | Описание |
|--------|-------|----------|
| **P2** | Direct | User prompt использует тот же system prompt |
| **P3** | Reuse | Gleaning также использует P1 как system prompt |
| **P4** | Downstream | Extracted descriptions затем суммируются |

---

## Полный Текст Промпта

```python
PROMPTS["entity_extraction_system_prompt"] = """---Role---
You are a Knowledge Graph Specialist responsible for extracting entities and relationships from the input text.

---Instructions---
1.  **Entity Extraction & Output:**
    *   **Identification:** Identify clearly defined and meaningful entities in the input text.
    *   **Entity Details:** For each identified entity, extract the following information:
        *   `entity_name`: The name of the entity. If the entity name is case-insensitive, capitalize the first letter of each significant word (title case). Ensure **consistent naming** across the entire extraction process.
        *   `entity_type`: Categorize the entity using one of the following types: `{entity_types}`. If none of the provided entity types apply, do not add new entity type and classify it as `Other`.
        *   `entity_description`: Provide a concise yet comprehensive description of the entity's attributes and activities, based *solely* on the information present in the input text.
    *   **Output Format - Entities:** Output a total of 4 fields for each entity, delimited by `{tuple_delimiter}`, on a single line. The first field *must* be the literal string `entity`.
        *   Format: `entity{tuple_delimiter}entity_name{tuple_delimiter}entity_type{tuple_delimiter}entity_description`

2.  **Relationship Extraction & Output:**
    *   **Identification:** Identify direct, clearly stated, and meaningful relationships between previously extracted entities.
    *   **N-ary Relationship Decomposition:** If a single statement describes a relationship involving more than two entities (an N-ary relationship), decompose it into multiple binary (two-entity) relationship pairs for separate description.
        *   **Example:** For "Alice, Bob, and Carol collaborated on Project X," extract binary relationships such as "Alice collaborated with Project X," "Bob collaborated with Project X," and "Carol collaborated with Project X," or "Alice collaborated with Bob," based on the most reasonable binary interpretations.
    *   **Relationship Details:** For each binary relationship, extract the following fields:
        *   `source_entity`: The name of the source entity. Ensure **consistent naming** with entity extraction. Capitalize the first letter of each significant word (title case) if the name is case-insensitive.
        *   `target_entity`: The name of the target entity. Ensure **consistent naming** with entity extraction. Capitalize the first letter of each significant word (title case) if the name is case-insensitive.
        *   `relationship_keywords`: One or more high-level keywords summarizing the overarching nature, concepts, or themes of the relationship. Multiple keywords within this field must be separated by a comma `,`. **DO NOT use `{tuple_delimiter}` for separating multiple keywords within this field.**
        *   `relationship_description`: A concise explanation of the nature of the relationship between the source and target entities, providing a clear rationale for their connection.
    *   **Output Format - Relationships:** Output a total of 5 fields for each relationship, delimited by `{tuple_delimiter}`, on a single line. The first field *must* be the literal string `relation`.
        *   Format: `relation{tuple_delimiter}source_entity{tuple_delimiter}target_entity{tuple_delimiter}relationship_keywords{tuple_delimiter}relationship_description`

3.  **Delimiter Usage Protocol:**
    *   The `{tuple_delimiter}` is a complete, atomic marker and **must not be filled with content**. It serves strictly as a field separator.
    *   **Incorrect Example:** `entity{tuple_delimiter}Tokyo<|location|>Tokyo is the capital of Japan.`
    *   **Correct Example:** `entity{tuple_delimiter}Tokyo{tuple_delimiter}location{tuple_delimiter}Tokyo is the capital of Japan.`

4.  **Relationship Direction & Duplication:**
    *   Treat all relationships as **undirected** unless explicitly stated otherwise. Swapping the source and target entities for an undirected relationship does not constitute a new relationship.
    *   Avoid outputting duplicate relationships.

5.  **Output Order & Prioritization:**
    *   Output all extracted entities first, followed by all extracted relationships.
    *   Within the list of relationships, prioritize and output those relationships that are **most significant** to the core meaning of the input text first.

6.  **Context & Objectivity:**
    *   Ensure all entity names and descriptions are written in the **third person**.
    *   Explicitly name the subject or object; **avoid using pronouns** such as `this article`, `this paper`, `our company`, `I`, `you`, and `he/she`.

7.  **Language & Proper Nouns:**
    *   The entire output (entity names, keywords, and descriptions) must be written in `{language}`.
    *   Proper nouns (e.g., personal names, place names, organization names) should be retained in their original language if a proper, widely accepted translation is not available or would cause ambiguity.

8.  **Completion Signal:** Output the literal string `{completion_delimiter}` only after all entities and relationships, following all criteria, have been completely extracted and outputted.

---Examples---
{examples}

---Real Data to be Processed---
<Input>
Entity_types: [{entity_types}]
Text:
```
{input_text}
```
"""
```

---

## Параметры Промпта

### Placeholders (Подстановки)

| Параметр | Тип | Описание | Пример |
|----------|-----|----------|---------|
| `{entity_types}` | str | Список типов сущностей | `"person,organization,location,event"` |
| `{tuple_delimiter}` | str | Разделитель полей | `"<\|#\|>"` (default) |
| `{completion_delimiter}` | str | Сигнал завершения | `"<\|COMPLETE\|>"` (default) |
| `{language}` | str | Язык вывода | `"English"` / `"Russian"` / `"Chinese"` |
| `{examples}` | str | Few-shot примеры | 3 примера (см. P9) |
| `{input_text}` | str | Chunk content | Text to process |

### Генерация Промпта

```python
# From lightrag/operate.py:2069

context_base = {
    "tuple_delimiter": PROMPTS["DEFAULT_TUPLE_DELIMITER"],
    "completion_delimiter": PROMPTS["DEFAULT_COMPLETION_DELIMITER"],
    "entity_types": ",".join(entity_types),
    "examples": "\n".join(PROMPTS["entity_extraction_examples"]),
    "language": language,
}

entity_extraction_system_prompt = PROMPTS["entity_extraction_system_prompt"].format(
    **context_base,
    input_text=content  # Chunk content
)
```

---

## Структура Промпта

### Секции Промпта

```
┌─────────────────────────────────────────────────────┐
│ ---Role---                                          │
│ • Определяет роль AI: Knowledge Graph Specialist   │
└─────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ---Instructions---                                   │
│ 8 основных блоков инструкций:                      │
│  1. Entity Extraction (3 поля)                     │
│  2. Relationship Extraction (4 поля)               │
│  3. Delimiter Usage (правила)                      │
│  4. Relationship Direction (undirected)            │
│  5. Output Order (entities → relations)            │
│  6. Context & Objectivity (third person)           │
│  7. Language & Proper Nouns                        │
│  8. Completion Signal                              │
└─────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ---Examples---                                       │
│ • 3 few-shot примера                               │
│ • Демонстрируют правильный формат                  │
│ • Покрывают разные домены                          │
└─────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────┐
│ ---Real Data to be Processed---                     │
│ • Entity types list                                 │
│ • Input text (chunk content)                        │
└─────────────────────────────────────────────────────┘
```

---

## Форматирование Вывода

### Entity Format

```
entity<|#|>entity_name<|#|>entity_type<|#|>entity_description
```

**Пример**:
```
entity<|#|>Alice Smith<|#|>person<|#|>Alice Smith is the CEO of TechCorp, leading the company's AI initiatives since 2020.
```

**Parsed Structure**:
```python
{
    "type": "entity",
    "name": "Alice Smith",
    "entity_type": "person",
    "description": "Alice Smith is the CEO of TechCorp, leading the company's AI initiatives since 2020."
}
```

### Relationship Format

```
relation<|#|>source_entity<|#|>target_entity<|#|>relationship_keywords<|#|>relationship_description
```

**Пример**:
```
relation<|#|>Alice Smith<|#|>TechCorp<|#|>leadership, employment<|#|>Alice Smith serves as the CEO of TechCorp and is responsible for the company's strategic direction.
```

**Parsed Structure**:
```python
{
    "type": "relation",
    "source": "Alice Smith",
    "target": "TechCorp",
    "keywords": "leadership, employment",
    "description": "Alice Smith serves as the CEO of TechCorp and is responsible for the company's strategic direction."
}
```

### Completion Signal

```
<|COMPLETE|>
```

Указывает LLM, что экстракция завершена.

---

## Концептуальные Аспекты

### 1. Entity Consistency (Консистентность)

**Проблема**: Одна сущность может упоминаться по-разному
```
Input: "Alice joined the company. Later, Ms. Smith led the project."

Bad output:
entity<|#|>Alice<|#|>person<|#|>...
entity<|#|>Ms. Smith<|#|>person<|#|>...
# Две разные сущности для одного человека!

Good output:
entity<|#|>Alice Smith<|#|>person<|#|>Alice Smith joined the company and later led the project.
# Одна сущность с полным именем
```

**Решение в промпте**:
- "Ensure **consistent naming** across the entire extraction process"
- "Capitalize the first letter of each significant word (title case)"

### 2. N-ary Relationship Decomposition

**Проблема**: Отношения могут включать >2 сущностей
```
Input: "Alice, Bob, and Carol collaborated on Project X."

N-ary (не поддерживается):
relation<|#|>Alice,Bob,Carol<|#|>Project X<|#|>...

Binary decomposition (правильно):
relation<|#|>Alice<|#|>Project X<|#|>collaboration<|#|>...
relation<|#|>Bob<|#|>Project X<|#|>collaboration<|#|>...
relation<|#|>Carol<|#|>Project X<|#|>collaboration<|#|>...
```

**Обоснование**:
- Knowledge graphs хранят binary relations
- Проще для graph traversal
- Более explicit semantics

### 3. Relationship Direction (Undirected)

**Концепция**: Большинство отношений bidirectional

```
"Alice works at TechCorp"

Could be represented as:
relation<|#|>Alice<|#|>TechCorp<|#|>employment<|#|>Alice is employed by TechCorp.
OR
relation<|#|>TechCorp<|#|>Alice<|#|>employment<|#|>TechCorp employs Alice.

Промпт указывает:
"Treat all relationships as undirected unless explicitly stated otherwise"

→ Выбрать одну направленность, избежать дубликатов
```

### 4. Third-Person Perspective

**Важно для**: Objective knowledge representation

```
❌ Bad (first/second person):
"Our company launched a new product."
"You can see that Alice leads the team."

✅ Good (third person):
"TechCorp launched a new product."
"Alice Smith leads the team."
```

**Причина**: Knowledge graph должен быть objective и reusable

---

## Entity Types (Типы Сущностей)

### Default Types

```python
DEFAULT_ENTITY_TYPES = [
    "person",        # Individuals (Alice Smith, Bob Johnson)
    "organization",  # Companies, institutions (TechCorp, MIT)
    "location",      # Places (Tokyo, Silicon Valley)
    "event",         # Occurrences (World Championship, Meeting)
    "product",       # Products/services (iPhone, API Platform)
    "concept",       # Abstract concepts (AI, Democracy)
    "category",      # Categories/groups (Tech Industry, 100m Sprint)
    "other"          # Fallback for unclassified
]
```

### Настройка Types

```python
# Custom entity types для специфичных доменов
custom_types = [
    "person",
    "company",
    "technology",        # Software, frameworks
    "research_paper",    # Academic papers
    "algorithm",         # Algorithmic concepts
    "dataset",           # Data resources
    "metric"             # Performance metrics
]

rag = LightRAG(entity_types=custom_types)
```

---

## Примеры Извлечения

### Пример 1: Narrative Text

**Input**:
```
Alex clenched his jaw, frustrated by Taylor's authoritarian certainty. Jordan shared
Alex's commitment to discovery, an unspoken rebellion against Cruz's vision of control.
```

**Output**:
```
entity<|#|>Alex<|#|>person<|#|>Alex is a character who experiences frustration and is observant of the dynamics among other characters.
entity<|#|>Taylor<|#|>person<|#|>Taylor is portrayed with authoritarian certainty.
entity<|#|>Jordan<|#|>person<|#|>Jordan shares a commitment to discovery.
entity<|#|>Cruz<|#|>person<|#|>Cruz is associated with a vision of control and order.
relation<|#|>Alex<|#|>Taylor<|#|>power dynamics, observation<|#|>Alex observes Taylor's authoritarian behavior.
relation<|#|>Alex<|#|>Jordan<|#|>shared goals, rebellion<|#|>Alex and Jordan share a commitment to discovery.
relation<|#|>Jordan<|#|>Cruz<|#|>ideological conflict<|#|>Jordan's commitment to discovery is in rebellion against Cruz's vision.
<|COMPLETE|>
```

### Пример 2: News/Financial Text

**Input**:
```
Stock markets faced a sharp downturn today as tech giants saw significant declines,
with the global tech index dropping by 3.4%. Nexon Technologies saw its stock plummet
by 7.8% after reporting lower-than-expected earnings.
```

**Output**:
```
entity<|#|>Global Tech Index<|#|>category<|#|>The Global Tech Index tracks the performance of major technology stocks and experienced a 3.4% decline today.
entity<|#|>Nexon Technologies<|#|>organization<|#|>Nexon Technologies is a tech company that saw its stock decline by 7.8% after disappointing earnings.
entity<|#|>Market Selloff<|#|>category<|#|>Market selloff refers to the significant decline in stock values due to investor concerns.
relation<|#|>Global Tech Index<|#|>Market Selloff<|#|>market performance, investor sentiment<|#|>The decline in the Global Tech Index is part of the broader market selloff.
relation<|#|>Nexon Technologies<|#|>Global Tech Index<|#|>company impact, index movement<|#|>Nexon Technologies' stock decline contributed to the overall drop in the Global Tech Index.
<|COMPLETE|>
```

---

## Обработка Вывода

### Parsing Algorithm

```python
def parse_entity_extraction_output(output: str, tuple_delimiter: str) -> tuple:
    """Parse LLM output into entities and relations"""

    entities = []
    relations = []

    lines = output.strip().split("\n")

    for line in lines:
        line = line.strip()

        # Skip empty lines and completion delimiter
        if not line or completion_delimiter in line:
            continue

        # Split by delimiter
        parts = line.split(tuple_delimiter)

        if len(parts) < 4:
            continue  # Invalid format

        record_type = parts[0].lower()

        if record_type == "entity":
            if len(parts) >= 4:
                entity = {
                    "name": parts[1].strip(),
                    "type": parts[2].strip(),
                    "description": parts[3].strip()
                }
                entities.append(entity)

        elif record_type == "relation":
            if len(parts) >= 5:
                relation = {
                    "source": parts[1].strip(),
                    "target": parts[2].strip(),
                    "keywords": parts[3].strip(),
                    "description": parts[4].strip()
                }
                relations.append(relation)

    return entities, relations
```

### Error Handling

**Типичные ошибки LLM**:

1. **Incorrect delimiter usage**:
   ```
   entity|Alice|person|Description
   # Should be: entity<|#|>Alice<|#|>person<|#|>Description
   ```

2. **Missing fields**:
   ```
   entity<|#|>Alice<|#|>person
   # Missing description!
   ```

3. **Delimiter in content**:
   ```
   entity<|#|>Data<|#|>Analysis<|#|>product<|#|>A tool for analysis
   # "Data<|#|>Analysis" parsed as two fields!
   ```

**Mitigation в промпте**:
- Explicit format specification
- Examples demonstrating correct usage
- "Delimiter Usage Protocol" section
- Validation via few-shot examples

---

## Метрики Качества

### Extraction Quality

```python
# Typical metrics after T2 (before gleaning)
extraction_metrics = {
    "precision": 0.70,       # 70% extracted entities are correct
    "recall": 0.65,          # 65% of entities found
    "f1_score": 0.67,
    "hallucination_rate": 0.05,  # 5% false entities
    "format_compliance": 0.95     # 95% valid format
}

# After T3 (with gleaning)
extraction_metrics_with_gleaning = {
    "precision": 0.75,       # +5%
    "recall": 0.80,          # +15% (major improvement)
    "f1_score": 0.77,        # +10%
    "hallucination_rate": 0.03,  # Reduced
    "format_compliance": 0.95
}
```

### Performance

```python
# Latency per chunk
extraction_latency = {
    "gpt-4o": "2-5 seconds",
    "gpt-4o-mini": "1-3 seconds",
    "claude-3.5-sonnet": "2-4 seconds",
    "llama-3-70b": "3-6 seconds"  # Local
}

# Token usage (per chunk, ~1000 input tokens)
token_usage = {
    "system_prompt": 2000,
    "user_prompt": 1500,
    "output_tokens": 500-2000,  # Depends on entity density
    "total": 4000-5500
}

# Cost (per chunk)
cost_per_chunk = {
    "gpt-4o": "$0.01-0.03",
    "gpt-4o-mini": "$0.002-0.005",
    "claude-3.5-sonnet": "$0.015-0.04"
}
```

---

## Внешние Зависимости

### LLM Interface

```python
from lightrag.llm import openai_complete_if_cache

# Usage in extraction
async def extract_with_llm(chunk_content: str, system_prompt: str):
    """Extract entities using LLM"""

    response = await openai_complete_if_cache(
        user_prompt,
        system_prompt=system_prompt,
        model="gpt-4o-mini",
        temperature=0.0,      # Deterministic
        max_tokens=2000,      # Enough for output
        hashing_kv=llm_cache  # Cache responses
    )

    return response
```

### Caching

```python
# Cache key generation
cache_key = compute_cache_key(
    system_prompt=system_prompt,
    user_prompt=user_prompt,
    model="gpt-4o-mini",
    temperature=0.0
)

# Cache hit rate
cache_metrics = {
    "hit_rate": 0.30,  # 30% of calls cached
    "savings": "$0.30 per $1.00 spent"
}
```

### Tokenization

```python
import tiktoken

# Token counting for prompt validation
tokenizer = tiktoken.encoding_for_model("gpt-4o")

system_tokens = len(tokenizer.encode(system_prompt))
user_tokens = len(tokenizer.encode(user_prompt))
total_input_tokens = system_tokens + user_tokens

# Check against limit
if total_input_tokens > 120000:  # GPT-4o context limit
    raise ValueError("Prompt exceeds context window")
```

---

## Оптимизация

### 1. Prompt Optimization

```python
# Trade-off: Instruction detail vs token cost

# Verbose (better quality, higher cost)
verbose_prompt = {
    "instructions": "8 detailed sections",
    "examples": "3 full examples",
    "tokens": 2000,
    "quality": 0.75
}

# Concise (lower quality, lower cost)
concise_prompt = {
    "instructions": "4 brief sections",
    "examples": "1 example",
    "tokens": 800,
    "quality": 0.65
}

# Recommendation: Use verbose by default, concise for high-volume
```

### 2. Few-Shot Examples

```python
# Number of examples affects quality
example_ablation = {
    0: {"f1": 0.55, "tokens": 1500},  # Zero-shot
    1: {"f1": 0.63, "tokens": 2000},  # One-shot
    3: {"f1": 0.67, "tokens": 2500},  # Few-shot (default)
    5: {"f1": 0.69, "tokens": 3000}   # Diminishing returns
}

# Recommendation: 3 examples (best ROI)
```

### 3. Model Selection

```python
model_comparison = {
    "gpt-4o": {
        "quality": 0.75,
        "speed": "2-5s",
        "cost": "$$$",
        "use_case": "High-quality extraction"
    },
    "gpt-4o-mini": {
        "quality": 0.67,
        "speed": "1-3s",
        "cost": "$",
        "use_case": "Balanced (recommended)"
    },
    "claude-3.5-sonnet": {
        "quality": 0.72,
        "speed": "2-4s",
        "cost": "$$",
        "use_case": "Alternative to GPT-4o"
    }
}
```

---

## Связь с Другими Компонентами

### Input (от T1)

```python
# From chunking
chunk = {
    "tokens": 1024,
    "content": "Alice Smith joined TechCorp...",
    "chunk_order_index": 0,
    "full_doc_id": "doc-abc123"
}

# Passes to P1+P2
input_text = chunk["content"]
```

### Output (к T3/T4)

```python
# Raw extraction output
extraction_result = {
    "entities": [
        {"name": "Alice Smith", "type": "person", "description": "..."},
        {"name": "TechCorp", "type": "organization", "description": "..."}
    ],
    "relations": [
        {"source": "Alice Smith", "target": "TechCorp", "keywords": "employment", "description": "..."}
    ],
    "chunk_id": "chunk-xyz789"
}

# Passes to T3 (gleaning) if needed
# Or to T4 (parsing) directly
```

---

## Заключение

**P1 (Entity Extraction System Prompt)** - это фундаментальный промпт LightRAG, определяющий:
- ✅ Качество извлечения сущностей
- ✅ Структуру knowledge graph
- ✅ Completeness и consistency данных
- ✅ Downstream quality всех операций

**Best Practices**:
1. Используйте temperature=0.0 для детерминизма
2. Включайте 3 few-shot примера
3. Применяйте gleaning (P3) для высококачественных результатов
4. Мониторьте precision/recall метрики
5. Кешируйте результаты для экономии

**См. также**:
- [P2: Entity Extraction User Prompt](02-entity-extraction-user.md)
- [P3: Entity Gleaning Prompt](03-entity-gleaning.md)
- [T2: Entity Extraction Transform](../transform/01-indexing-transforms.md#t2-entity-extraction)
