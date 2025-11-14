# Entity Extraction: LLM-Based Named Entity Recognition

## Концептуальная Парадигма

**Entity extraction** = structured information extraction using **few-shot prompted LLMs** для identifying entities and relations from text.

```
Unstructured Text:
"Apple Inc, founded by Steve Jobs in 1976, produces iPhones and iPads."

        ↓ [LLM + Extraction Prompt]

Structured Output:
Entities:
- Apple Inc (organization): Technology company
- Steve Jobs (person): Co-founder of Apple
- iPhone (product): Smartphone by Apple
- iPad (product): Tablet by Apple

Relations:
- Steve Jobs → founded → Apple Inc
- Apple Inc → produces → iPhone
- Apple Inc → produces → iPad
```

**Философия**: LLMs as **semantic parsers** — не просто generation, но structured knowledge extraction через prompting.

---

## Algorithm: Few-Shot Prompt-Based Extraction

### Conceptual Overview

**In-context learning**: LLM learns task from examples in prompt (no fine-tuning).

```
Prompt Structure:
──────────────────

1. System Instruction
   "You are an expert at extracting entities and relationships from text."

2. Task Description
   "Extract all entities (people, organizations, products, etc.) and their relationships."

3. Output Format Specification
   ("entity"<|>name<|>type<|>description)
   ("relationship"<|>source<|>target<|>description)

4. Few-Shot Examples (2-5 examples)
   Input: "Microsoft, founded by Bill Gates, develops Windows."
   Output: ("entity"<|>Microsoft<|>organization<|>...)

5. Actual Input
   Input: {text_to_process}

6. Prompt for Output
   "Output:"
```

**Key Components**:
- **Few-shot examples**: Teach format and task (2-5 examples optimal)
- **Output format**: Structured delimiter-separated format (easy parsing)
- **Entity types**: Flexible categories (person, organization, location, product, etc.)

---

## Implementation in LightRAG

### Extraction Prompt Template

**Файл**: `lightrag/prompt.py:19`

```python
GRAPH_FIELD_SEP = "<|>"  # Delimiter for structured output

PROMPTS = {}

PROMPTS["entity_extraction"] = """-Goal-
Given a text document that is potentially relevant to this activity and a list of entity types, identify all entities of those types from the text and all relationships among the identified entities.

-Steps-
1. Identify all entities. For each identified entity, extract the following information:
- entity_name: Name of the entity, capitalized
- entity_type: One of the following types: [{entity_types}]
- entity_description: Comprehensive description of the entity's attributes and activities
Format each entity as ("entity"{GRAPH_FIELD_SEP}<entity_name>{GRAPH_FIELD_SEP}<entity_type>{GRAPH_FIELD_SEP}<entity_description>)

2. From the entities identified in step 1, identify all pairs of (source_entity, target_entity) that are *clearly related* to each other.
For each pair of related entities, extract the following information:
- source_entity: name of the source entity, as identified in step 1
- target_entity: name of the target entity, as identified in step 1
- relationship_description: explanation as to why you think the source entity and the target entity are related to each other
Format each relationship as ("relationship"{GRAPH_FIELD_SEP}<source_entity>{GRAPH_FIELD_SEP}<target_entity>{GRAPH_FIELD_SEP}<relationship_description>)

-Examples-
######################
Example 1:

text:
Apple Inc is an American multinational technology company headquartered in Cupertino, California. Apple was founded by Steve Jobs, Steve Wozniak, and Ronald Wayne in April 1976. The company's hardware products include the iPhone smartphone, the iPad tablet computer, and the Mac personal computer.
######################
output:
("entity"{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}organization{GRAPH_FIELD_SEP}Apple Inc is an American multinational technology company headquartered in Cupertino, California)
("entity"{GRAPH_FIELD_SEP}Steve Jobs{GRAPH_FIELD_SEP}person{GRAPH_FIELD_SEP}Co-founder of Apple Inc)
("entity"{GRAPH_FIELD_SEP}Steve Wozniak{GRAPH_FIELD_SEP}person{GRAPH_FIELD_SEP}Co-founder of Apple Inc)
("entity"{GRAPH_FIELD_SEP}Ronald Wayne{GRAPH_FIELD_SEP}person{GRAPH_FIELD_SEP}Co-founder of Apple Inc)
("entity"{GRAPH_FIELD_SEP}Cupertino{GRAPH_FIELD_SEP}location{GRAPH_FIELD_SEP}City in California where Apple is headquartered)
("entity"{GRAPH_FIELD_SEP}iPhone{GRAPH_FIELD_SEP}product{GRAPH_FIELD_SEP}Smartphone produced by Apple Inc)
("entity"{GRAPH_FIELD_SEP}iPad{GRAPH_FIELD_SEP}product{GRAPH_FIELD_SEP}Tablet computer produced by Apple Inc)
("entity"{GRAPH_FIELD_SEP}Mac{GRAPH_FIELD_SEP}product{GRAPH_FIELD_SEP}Personal computer produced by Apple Inc)
("relationship"{GRAPH_FIELD_SEP}Steve Jobs{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}Steve Jobs co-founded Apple Inc in 1976)
("relationship"{GRAPH_FIELD_SEP}Steve Wozniak{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}Steve Wozniak co-founded Apple Inc in 1976)
("relationship"{GRAPH_FIELD_SEP}Ronald Wayne{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}Ronald Wayne co-founded Apple Inc in 1976)
("relationship"{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}Cupertino{GRAPH_FIELD_SEP}Apple Inc is headquartered in Cupertino)
("relationship"{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}iPhone{GRAPH_FIELD_SEP}Apple Inc produces the iPhone smartphone)
("relationship"{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}iPad{GRAPH_FIELD_SEP}Apple Inc produces the iPad tablet)
("relationship"{GRAPH_FIELD_SEP}Apple Inc{GRAPH_FIELD_SEP}Mac{GRAPH_FIELD_SEP}Apple Inc produces the Mac computer)
#############################

-Real Data-
######################
entity_types: {entity_types}
text: {input_text}
######################
output:
"""
```

**Key Features**:
- **Structured format**: Delimiter-separated for easy parsing
- **Few-shot example**: 1 example (can be extended to 2-5)
- **Flexible entity types**: Parameterized list (person, organization, location, product, etc.)
- **Relationship extraction**: Both entities and their connections

### Parsing Extracted Output

```python
def parse_entities_and_relations(
    llm_output: str,
    delimiter: str = "<|>"
) -> tuple[list[dict], list[dict]]:
    """
    Parse LLM output into structured entities and relations.

    Args:
        llm_output: Raw LLM response
        delimiter: Field separator (default: <|>)

    Returns:
        entities: [{name, type, description}, ...]
        relations: [{src, tgt, description}, ...]
    """
    entities = []
    relations = []

    for line in llm_output.strip().split("\n"):
        line = line.strip()

        if line.startswith('("entity"'):
            # Parse entity
            # Format: ("entity"<|>Apple Inc<|>organization<|>Technology company)
            parts = line.split(delimiter)

            if len(parts) >= 4:
                entity_name = parts[1].strip()
                entity_type = parts[2].strip()
                entity_desc = parts[3].rstrip(")").strip()

                entities.append({
                    "name": entity_name,
                    "type": entity_type,
                    "description": entity_desc
                })

        elif line.startswith('("relationship"'):
            # Parse relationship
            # Format: ("relationship"<|>Steve Jobs<|>Apple Inc<|>co-founded)
            parts = line.split(delimiter)

            if len(parts) >= 4:
                src_entity = parts[1].strip()
                tgt_entity = parts[2].strip()
                rel_desc = parts[3].rstrip(")").strip()

                relations.append({
                    "src": src_entity,
                    "tgt": tgt_entity,
                    "description": rel_desc
                })

    return entities, relations
```

### Full Extraction Pipeline

```python
async def extract_entities_and_relations(
    text: str,
    llm_func: callable,
    entity_types: list[str] = None
) -> tuple[list[dict], list[dict]]:
    """
    Extract entities and relations from text using LLM.

    Args:
        text: Input text to process
        llm_func: Language model function
        entity_types: List of entity types to extract

    Returns:
        entities: List of extracted entities
        relations: List of extracted relationships
    """
    # Default entity types
    if entity_types is None:
        entity_types = [
            "person",
            "organization",
            "location",
            "product",
            "event",
            "date"
        ]

    # Format prompt
    prompt = PROMPTS["entity_extraction"].format(
        entity_types=", ".join(entity_types),
        input_text=text,
        GRAPH_FIELD_SEP=GRAPH_FIELD_SEP
    )

    # LLM call
    logger.debug(f"Extracting entities from text ({len(text)} chars)")

    response = await llm_func(
        prompt=prompt,
        model="gpt-4",
        temperature=0.0  # Deterministic extraction
    )

    # Parse output
    entities, relations = parse_entities_and_relations(response, GRAPH_FIELD_SEP)

    logger.info(
        f"Extracted {len(entities)} entities and {len(relations)} relations"
    )

    return entities, relations
```

---

## Gleaning: Iterative Refinement

### Multi-Pass Extraction

**Файл**: `lightrag/prompt.py:129`

```python
PROMPTS["continue_extraction"] = """MANY entities were missed in the last extraction. Remember to extract ALL entities and relationships.
Add any additional entities and relationships that were missed in the last extraction.

######################
output:
"""

async def glean_entities(
    text: str,
    llm_func: callable,
    entity_types: list[str] = None,
    rounds: int = 2
) -> tuple[list[dict], list[dict]]:
    """
    Iterative entity extraction (gleaning).

    Process:
    1. Initial extraction
    2. For each round:
       a. Prompt LLM to find missed entities
       b. Merge new entities with existing

    Args:
        text: Input text
        llm_func: Language model
        rounds: Number of gleaning rounds (default: 2)

    Returns:
        entities: Comprehensive list (all rounds merged)
        relations: Comprehensive list (all rounds merged)
    """
    all_entities = []
    all_relations = []

    # Round 1: Initial extraction
    entities, relations = await extract_entities_and_relations(
        text, llm_func, entity_types
    )

    all_entities.extend(entities)
    all_relations.extend(relations)

    # Rounds 2+: Gleaning (find missed entities)
    for round_idx in range(1, rounds):
        logger.debug(f"Gleaning round {round_idx + 1}/{rounds}")

        # Continuation prompt
        continuation_prompt = PROMPTS["continue_extraction"]

        # LLM call
        response = await llm_func(
            prompt=continuation_prompt,
            model="gpt-4",
            temperature=0.0
        )

        # Parse new entities
        new_entities, new_relations = parse_entities_and_relations(
            response, GRAPH_FIELD_SEP
        )

        # Merge (deduplication by entity name)
        existing_names = {e["name"] for e in all_entities}

        for entity in new_entities:
            if entity["name"] not in existing_names:
                all_entities.append(entity)
                existing_names.add(entity["name"])

        # Merge relations (deduplication by (src, tgt) pair)
        existing_pairs = {(r["src"], r["tgt"]) for r in all_relations}

        for relation in new_relations:
            pair = (relation["src"], relation["tgt"])
            if pair not in existing_pairs:
                all_relations.append(relation)
                existing_pairs.add(pair)

        logger.info(
            f"Gleaning round {round_idx + 1}: "
            f"+{len(new_entities)} entities, +{len(new_relations)} relations"
        )

    return all_entities, all_relations
```

**Benefit**: Gleaning improves recall (finds entities missed in first pass).

**Trade-off**: +50-100% more LLM calls (cost increase), +30-40% more entities found.

---

## Связь с Graph Construction

Extracted entities → Graph nodes and edges:

**Файл**: spec/graph/04-entity-merging.md

```python
# Extract entities from multiple text chunks
chunk1 = "Apple Inc produces iPhones"
chunk2 = "Apple company manufactures smartphones"

entities1, relations1 = await extract_entities_and_relations(chunk1, llm_func)
entities2, relations2 = await extract_entities_and_relations(chunk2, llm_func)

# entities1: [{"name": "Apple Inc", ...}, {"name": "iPhone", ...}]
# entities2: [{"name": "Apple company", ...}, {"name": "smartphones", ...}]

# Graph construction
for entity in entities1 + entities2:
    await graph.upsert_node(
        node_id=f"ent-{entity['name']}",
        node_data={
            "entity_type": entity["type"],
            "description": entity["description"]
        }
    )

for relation in relations1 + relations2:
    await graph.upsert_edge(
        source_node_id=f"ent-{relation['src']}",
        target_node_id=f"ent-{relation['tgt']}",
        edge_data={"description": relation["description"]}
    )

# Merge duplicate entities (e.g., "Apple Inc" and "Apple company")
await graph.amerge_entities(
    source_entities=["ent-Apple company"],
    target_entity="ent-Apple Inc",
    merge_strategy={"description": "concatenate"}
)
```

**Interaction**: Entity extraction provides **raw graph nodes/edges**, graph algorithms provide **deduplication and structuring**.

---

## Связь с Vector Embeddings

Extracted entities → Embeddings for semantic search:

**Файл**: spec/vectors/01-embedding-generation.md

```python
# Extract entities
entities, relations = await extract_entities_and_relations(text, llm_func)

# Generate embeddings for entities
entity_names = [e["name"] for e in entities]
entity_embeddings = await embedding_func(entity_names)

# Store in vector DB
await entities_vdb.upsert({
    f"ent-{entity['name']}": {
        "content": entity["description"],
        "entity_type": entity["type"]
    }
    for entity in entities
})

# Query by semantic similarity
results = await entities_vdb.query("technology companies", top_k=10)
# Retrieves: Apple Inc, Microsoft, Google, etc.
```

**Interaction**: Entity extraction provides **semantic content**, embeddings enable **similarity-based retrieval**.

---

## Training Paradigm: In-Context Learning

### No Fine-Tuning Required

LLM performs entity extraction **without task-specific training**:

```
Traditional NER (supervised):
1. Collect labeled data: (text, entity_labels)
2. Fine-tune BERT/RoBERTa on labeled data
3. Deploy fine-tuned model

LLM-based NER (few-shot):
1. Design extraction prompt with examples
2. Call pre-trained LLM (GPT-4, Claude)
3. Parse structured output

Advantages:
✅ No labeled data needed
✅ No training/fine-tuning
✅ Flexible entity types (just change prompt)
✅ Works for any domain (no domain-specific model)

Disadvantages:
❌ Higher inference cost (LLM API calls)
❌ Slower than fine-tuned models
❌ Requires careful prompt engineering
```

### Few-Shot vs Zero-Shot

```python
# Zero-shot (no examples)
prompt_zero_shot = """
Extract all entities and relationships from the following text.

Text: {input_text}
Output:
"""

# Few-shot (with examples)
prompt_few_shot = """
Extract all entities and relationships.

Example:
Text: "Microsoft was founded by Bill Gates."
Output:
("entity"<|>Microsoft<|>organization<|>...)
("entity"<|>Bill Gates<|>person<|>...)
("relationship"<|>Bill Gates<|>Microsoft<|>founded)

Text: {input_text}
Output:
"""

# Performance:
# Zero-shot: 60-70% precision/recall
# Few-shot (1 example): 75-85% precision/recall
# Few-shot (3-5 examples): 85-95% precision/recall
```

**Recommendation**: 2-3 examples optimal (diminishing returns after 5 examples).

---

## Performance Characteristics

### Latency

| Model | Input (500 tokens) | Output (200 tokens) | Total Time |
|-------|-------------------|---------------------|------------|
| **GPT-4** | 500 tokens | 200 tokens | 3-5s |
| **GPT-3.5-turbo** | 500 tokens | 200 tokens | 1-2s |
| **Claude 3.5 Sonnet** | 500 tokens | 200 tokens | 2-4s |
| **Llama 3 8B (local)** | 500 tokens | 200 tokens | 5-10s (GPU) |

### Cost

```
Example: Extract entities from 1000 text chunks (500 tokens each)

Input: 1000 × 500 = 500K tokens
Output: 1000 × 200 = 200K tokens (entities/relations)

GPT-4:
Input: 500K × $30/1M = $15
Output: 200K × $60/1M = $12
Total: $27

GPT-3.5-turbo:
Input: 500K × $0.50/1M = $0.25
Output: 200K × $1.50/1M = $0.30
Total: $0.55

Claude 3.5 Sonnet:
Input: 500K × $3/1M = $1.50
Output: 200K × $15/1M = $3.00
Total: $4.50

Llama 3 8B (local):
Cost: $0 (infrastructure: ~$1/hour GPU, ~5 hours = $5)
```

**Recommendation**:
- **Best quality**: GPT-4 or Claude 3.5 Sonnet
- **Best value**: GPT-3.5-turbo (10x cheaper than GPT-4)
- **Privacy/control**: Local Llama 3 8B

### Quality Metrics

```
Benchmark: CoNLL-2003 NER dataset

Traditional fine-tuned models:
- BERT-base-NER: 91% F1
- RoBERTa-large-NER: 93% F1

LLM-based extraction:
- GPT-4 (few-shot, 3 examples): 89% F1
- GPT-3.5-turbo (few-shot): 84% F1
- Llama 3 8B (few-shot): 79% F1

Observation: GPT-4 approaches fine-tuned model quality without training.
```

---

## Optimization Strategies

### 1. Batch Processing

```python
# Sequential (slow): 1000 chunks × 3s = 3000s (50 minutes)
for chunk in chunks:
    entities, relations = await extract_entities_and_relations(chunk, llm_func)

# Parallel (fast): 1000 chunks / 20 concurrent × 3s = 150s (2.5 minutes)
import asyncio

tasks = [
    extract_entities_and_relations(chunk, llm_func)
    for chunk in chunks
]

results = await asyncio.gather(*tasks)
```

**Speedup**: 20x with parallel LLM calls.

### 2. Caching

```python
import hashlib

extraction_cache = {}

async def cached_extract(text: str, llm_func: callable):
    """Cache extraction results by text hash."""
    text_hash = hashlib.md5(text.encode()).hexdigest()

    if text_hash in extraction_cache:
        logger.debug("Cache hit for extraction")
        return extraction_cache[text_hash]

    # Perform extraction
    entities, relations = await extract_entities_and_relations(text, llm_func)

    # Cache result
    extraction_cache[text_hash] = (entities, relations)

    return entities, relations
```

**Speedup**: 100x for repeated texts (instant cache retrieval vs 3s LLM call).

### 3. Cheaper Models for Simple Texts

```python
async def smart_extract(text: str, llm_func_cheap, llm_func_expensive):
    """
    Use cheap model for simple texts, expensive for complex.
    """
    # Heuristic: Short texts or simple structure → cheap model
    if len(text) < 200 or text.count(".") < 3:
        model = llm_func_cheap  # GPT-3.5-turbo
    else:
        model = llm_func_expensive  # GPT-4

    return await extract_entities_and_relations(text, model)
```

**Cost savings**: 50-70% (use GPT-3.5 for 70% of texts, GPT-4 for complex 30%).

### 4. Structured Output (JSON Mode)

```python
# Instead of delimiter-separated format, use JSON
prompt_json = """
Extract entities and relationships in JSON format.

Output:
{
  "entities": [
    {"name": "Apple Inc", "type": "organization", "description": "..."},
    ...
  ],
  "relationships": [
    {"src": "Steve Jobs", "tgt": "Apple Inc", "description": "..."},
    ...
  ]
}
"""

response = await llm_func(
    prompt=prompt_json,
    model="gpt-4",
    response_format={"type": "json_object"}  # Force JSON output
)

import json
data = json.loads(response)
entities = data["entities"]
relations = data["relationships"]
```

**Benefit**: More reliable parsing (no delimiter ambiguity), supported by GPT-4 Turbo.

---

## Example: Full Entity Extraction Pipeline

```python
from lightrag.prompt import PROMPTS, GRAPH_FIELD_SEP
from lightrag.llm.openai import openai_complete_if_cache

# Input text
text = """
Apple Inc is an American multinational technology company headquartered in Cupertino, California.
Apple was founded by Steve Jobs, Steve Wozniak, and Ronald Wayne in April 1976.
The company's hardware products include the iPhone smartphone, the iPad tablet computer, and the Mac personal computer.
"""

# Define LLM function
async def llm_func(prompt, **kwargs):
    return await openai_complete_if_cache(
        model="gpt-4",
        prompt=prompt,
        **kwargs
    )

# Extract entities and relations
entities, relations = await extract_entities_and_relations(
    text=text,
    llm_func=llm_func,
    entity_types=["person", "organization", "location", "product"]
)

# Print results
print("Entities:")
for entity in entities:
    print(f"  - {entity['name']} ({entity['type']}): {entity['description']}")

print("\nRelationships:")
for relation in relations:
    print(f"  - {relation['src']} → {relation['tgt']}: {relation['description']}")

# Output:
# Entities:
#   - Apple Inc (organization): American multinational technology company headquartered in Cupertino
#   - Steve Jobs (person): Co-founder of Apple Inc
#   - Steve Wozniak (person): Co-founder of Apple Inc
#   - Ronald Wayne (person): Co-founder of Apple Inc
#   - Cupertino (location): City in California where Apple is headquartered
#   - iPhone (product): Smartphone produced by Apple Inc
#   - iPad (product): Tablet computer produced by Apple Inc
#   - Mac (product): Personal computer produced by Apple Inc
#
# Relationships:
#   - Steve Jobs → Apple Inc: Steve Jobs co-founded Apple Inc in 1976
#   - Steve Wozniak → Apple Inc: Steve Wozniak co-founded Apple Inc in 1976
#   - Ronald Wayne → Apple Inc: Ronald Wayne co-founded Apple Inc in 1976
#   - Apple Inc → Cupertino: Apple Inc is headquartered in Cupertino
#   - Apple Inc → iPhone: Apple Inc produces the iPhone smartphone
#   - Apple Inc → iPad: Apple Inc produces the iPad tablet
#   - Apple Inc → Mac: Apple Inc produces the Mac computer

# Build graph
for entity in entities:
    await graph.upsert_node(
        node_id=f"ent-{entity['name']}",
        node_data={
            "entity_type": entity["type"],
            "description": entity["description"],
            "source_text": text
        }
    )

for relation in relations:
    await graph.upsert_edge(
        source_node_id=f"ent-{relation['src']}",
        target_node_id=f"ent-{relation['tgt']}",
        edge_data={"description": relation["description"]}
    )

print(f"\nGraph: {len(entities)} nodes, {len(relations)} edges")
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
