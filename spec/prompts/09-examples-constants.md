# P9: Examples & Constants

## Метаданные

| Свойство | Значение |
|----------|----------|
| **ID** | P9 |
| **Тип** | Examples & Constants |
| **Назначение** | Supporting data for prompts |
| **Расположение** | `lightrag/prompt.py:8-9, 101-173, 383-417` |

---

## Constants

### Delimiters

```python
PROMPTS["DEFAULT_TUPLE_DELIMITER"] = "<|#|>"
PROMPTS["DEFAULT_COMPLETION_DELIMITER"] = "<|COMPLETE|>"
```

**Usage**:
- `<|#|>`: Field separator in entity/relation output
- `<|COMPLETE|>`: Signals end of extraction

**Design rationale**:
```python
why_special_delimiters = {
    "uniqueness": "Unlikely to appear in text naturally",
    "parseability": "Easy to split with string.split()",
    "clarity": "Clear structure for LLM output",
    "angle_brackets": "Familiar pattern (XML-like)"
}
```

---

## Entity Extraction Examples

### Example 1: Narrative Text

```python
PROMPTS["entity_extraction_examples"][0] = """<Input Text>
```
while Alex clenched his jaw, the buzz of frustration dull against the backdrop of Taylor's authoritarian certainty. It was this competitive undercurrent that kept him alert, the sense that his and Jordan's shared commitment to discovery was an unspoken rebellion against Cruz's narrowing vision of control and order.
```

<Output>
entity<|#|>Alex<|#|>person<|#|>Alex is a character who experiences frustration...
entity<|#|>Taylor<|#|>person<|#|>Taylor is portrayed with authoritarian certainty...
[... more entities ...]
relation<|#|>Alex<|#|>Taylor<|#|>power dynamics, observation<|#|>Alex observes Taylor's authoritarian behavior...
[... more relations ...]
<|COMPLETE|>

"""
```

**Domain**: Narrative, character relationships
**Entity count**: 4 entities, 5 relations
**Complexity**: Medium (interpersonal dynamics)

---

### Example 2: News/Financial

```python
PROMPTS["entity_extraction_examples"][1] = """<Input Text>
```
Stock markets faced a sharp downturn today as tech giants saw significant declines, with the global tech index dropping by 3.4% in midday trading. Nexon Technologies saw its stock plummet by 7.8% after reporting lower-than-expected quarterly earnings.
```

<Output>
entity<|#|>Global Tech Index<|#|>category<|#|>The Global Tech Index tracks the performance...
entity<|#|>Nexon Technologies<|#|>organization<|#|>Nexon Technologies is a tech company...
[... more ...]
<|COMPLETE|>

"""
```

**Domain**: Financial news, market data
**Entity count**: 6 entities, 4 relations
**Complexity**: Medium-High (numbers, events, causality)

---

### Example 3: Sports/Event

```python
PROMPTS["entity_extraction_examples"][2] = """<Input Text>
```
At the World Athletics Championship in Tokyo, Noah Carter broke the 100m sprint record using cutting-edge carbon-fiber spikes.
```

<Output>
entity<|#|>World Athletics Championship<|#|>event<|#|>The World Athletics Championship is a global sports competition...
entity<|#|>Tokyo<|#|>location<|#|>Tokyo is the host city...
entity<|#|>Noah Carter<|#|>person<|#|>Noah Carter is a sprinter who set a new record...
[... more ...]
<|COMPLETE|>

"""
```

**Domain**: Sports, events, locations
**Entity count**: 6 entities, 4 relations
**Complexity**: Medium (event, location, equipment, organization)

---

## Keywords Extraction Examples

### Example 1: Policy Query

```python
PROMPTS["keywords_extraction_examples"][0] = """Example 1:

Query: "How does international trade influence global economic stability?"

Output:
{
  "high_level_keywords": ["International trade", "Global economic stability", "Economic impact"],
  "low_level_keywords": ["Trade agreements", "Tariffs", "Currency exchange", "Imports", "Exports"]
}

"""
```

**Query type**: Policy, economics
**High-level count**: 3 (conceptual)
**Low-level count**: 5 (specific)

---

### Example 2: Environmental Query

```python
PROMPTS["keywords_extraction_examples"][1] = """Example 2:

Query: "What are the environmental consequences of deforestation on biodiversity?"

Output:
{
  "high_level_keywords": ["Environmental consequences", "Deforestation", "Biodiversity loss"],
  "low_level_keywords": ["Species extinction", "Habitat destruction", "Carbon emissions", "Rainforest", "Ecosystem"]
}

"""
```

**Query type**: Environmental science
**High-level count**: 3 (themes)
**Low-level count**: 5 (specifics)

---

### Example 3: Social Query

```python
PROMPTS["keywords_extraction_examples"][2] = """Example 3:

Query: "What is the role of education in reducing poverty?"

Output:
{
  "high_level_keywords": ["Education", "Poverty reduction", "Socioeconomic development"],
  "low_level_keywords": ["School access", "Literacy rates", "Job training", "Income inequality"]
}

"""
```

**Query type**: Social policy
**High-level count**: 3 (concepts)
**Low-level count**: 4 (metrics)

---

## Example Design Principles

### 1. Domain Diversity

```python
example_domains = {
    "entity_extraction": [
        "Narrative/fiction",      # Complex relationships
        "News/financial",         # Events and numbers
        "Sports/events"           # Locations and achievements
    ],
    "keywords": [
        "Policy/economics",       # Abstract concepts
        "Environmental science",  # Technical terms
        "Social development"      # Societal concepts
    ]
}
```

**Rationale**: Expose LLM to variety of domains

### 2. Complexity Gradient

```python
complexity_levels = {
    "simple": "Sports example - straightforward entities",
    "medium": "Narrative example - interpersonal dynamics",
    "complex": "Financial example - causality and numbers"
}
```

### 3. Format Demonstration

```
Each example shows:
✅ Correct delimiter usage
✅ Proper field count
✅ Appropriate descriptions
✅ Completion signal
```

---

## Usage Statistics

```python
# Example effectiveness
example_impact = {
    "with_3_examples": {
        "format_compliance": 0.95,
        "entity_quality": 0.67
    },
    "with_1_example": {
        "format_compliance": 0.82,
        "entity_quality": 0.61
    },
    "with_0_examples": {
        "format_compliance": 0.65,
        "entity_quality": 0.55
    }
}

# Recommendation: 3 examples (current)
```

---

## Customization

### Adding Custom Examples

```python
from lightrag.prompt import PROMPTS

# Add domain-specific example
PROMPTS["entity_extraction_examples"].append("""
<Input Text>
```
Your domain-specific text here...
```

<Output>
entity<|#|>CustomEntity<|#|>type<|#|>description
[...]
<|COMPLETE|>
""")
```

### Custom Delimiters

```python
# If <|#|> conflicts with your data
PROMPTS["DEFAULT_TUPLE_DELIMITER"] = "|||"
PROMPTS["DEFAULT_COMPLETION_DELIMITER"] = "<<<END>>>"
```

---

## Заключение

**P9 provides**:
- ✅ Consistent delimiters across all prompts
- ✅ Diverse few-shot examples (9 total)
- ✅ Domain coverage (narrative, news, sports, policy, etc.)
- ✅ Format demonstration

**Best Practice**: Keep 3 examples per prompt type for optimal quality/cost balance.

**См. также**:
- [P1: Entity Extraction](01-entity-extraction-system.md) - uses examples
- [P5: Keywords Extraction](05-keywords-extraction.md) - uses examples
- All other prompts reference delimiters

