# LLM Semantic Processing Patterns: Паттерны Семантической Обработки Языковыми Моделями

## Концептуальная Парадигма

LLM агенты в LightRAG демонстрируют **attractor-based processing patterns** - паттерны обработки, основанные на семантических аттракторах. Каждый LLM agent выполняет специфическую роль в:
- **Идентификации аттракторов** (entity extraction)
- **Уточнении аттракторов** (gleaning)
- **Консолидации аттракторов** (summarization)
- **Навигации по аттракторам** (keywords extraction)
- **Синтезе из аттракторов** (answer generation)

## Pattern 1: Extraction as Attractor Identification (Извлечение как Идентификация Аттракторов)

### Концепция

LLM **идентифицирует семантические аттракторы** (entities + relations) в unstructured text.

```
Unstructured Text (semantic soup)
    ↓
[Entity Extraction via P1+P2]
    ↓
Structured Attractors:
    • Entities (concept nodes)
    • Relations (connections between nodes)
```

**Философия**: "В тексте скрыты концепции-аттракторы, которые организуют семантическое пространство."

### Prompt Architecture

**System Prompt (P1)**:
```python
PROMPTS["entity_extraction_system_prompt"] = """---Role---
You are a Knowledge Graph Specialist responsible for extracting
entities and relationships from the input text.

---Instructions---
1. **Entity Extraction & Output:**
   * Identify ALL entities (person, organization, location, concept, etc.)
   * Format: entity{tuple_delimiter}entity_name{tuple_delimiter}entity_type{tuple_delimiter}entity_description

2. **Relationship Extraction & Output:**
   * Identify relationships between entities
   * Format: relation{tuple_delimiter}source{tuple_delimiter}target{tuple_delimiter}keywords{tuple_delimiter}description
...
"""
```

**См.**: [P1: Entity Extraction System](../prompts/01-entity-extraction-system.md)

### Attractor Identification Process

```python
async def identify_attractors(chunk_text):
    """
    LLM identifies semantic attractors in text.

    Input: Unstructured text chunk
    Output: Structured attractors (entities + relations)
    """

    # Step 1: LLM processes text with P1+P2
    system_prompt = PROMPTS["entity_extraction_system_prompt"]
    user_prompt = PROMPTS["entity_extraction_user_prompt"].format(
        input_text=chunk_text,
        entity_types=entity_types,
        ...
    )

    response = await llm_func(
        user_prompt,
        system_prompt=system_prompt,
        temperature=0.0  # ← Deterministic extraction
    )

    # Step 2: Parse LLM output
    entities, relations = parse_extraction_output(response)

    # entities = identified attractors
    # [
    #     {"entity_name": "Apple Inc", "entity_type": "organization", ...},
    #     {"entity_name": "Steve Jobs", "entity_type": "person", ...},
    #     ...
    # ]

    # relations = connections between attractors
    # [
    #     {"src": "Steve Jobs", "tgt": "Apple Inc", "type": "founded", ...},
    #     ...
    # ]

    return entities, relations
```

**Код** (lightrag/operate.py:2010):
```python
async def extract_entities(
    chunks: list[dict],
    knowledge_graph_inst: BaseGraphStorage,
    ...
):
    """
    ATTRACTOR IDENTIFICATION: LLM extracts entities/relations.

    Semantic Pattern:
    • Unstructured text → Structured concepts
    • Hidden attractors → Explicit nodes
    • Implicit relations → Explicit edges
    """

    # Process chunks with LLM (parallel)
    entity_extraction_tasks = []
    for chunk in chunks:
        task = _handle_single_entity_extraction(
            chunk["content"],
            llm_func=llm_func,
            prompts={"system": P1, "user": P2},
            ...
        )
        entity_extraction_tasks.append(task)

    results = await asyncio.gather(*entity_extraction_tasks)

    # results = identified attractors from all chunks
```

### Semantic Properties

```
✅ Discovers hidden structure:
   Text: "Apple Inc, founded by Steve Jobs..."
   → Entities: ["Apple Inc", "Steve Jobs"]
   → Relation: ("Steve Jobs", "founded", "Apple Inc")

✅ Creates semantic anchors:
   Entities become attractors in graph space
   Future queries can activate these attractors

✅ Deterministic (temperature=0):
   Same text → same entities (consistency)
```

---

## Pattern 2: Gleaning as Attractor Refinement (Уточнение Аттракторов)

### Концепция

LLM **уточняет и дополняет** идентифицированные аттракторы через multi-turn dialog.

```
Initial Extraction (incomplete attractors)
    ↓
[Gleaning via P3 + Conversation History]
    ↓
Refined Attractors (higher coverage)
```

**Философия**: "Первичное извлечение неполно - LLM может найти пропущенные аттракторы при повторном рассмотрении."

### Prompt Architecture

**Gleaning Prompt (P3)**:
```python
PROMPTS["entity_extraction_gleaning_prompt"] = """---Role---
You are an expert Knowledge Graph refiner...

---Task---
Review the previously extracted entities and relationships.
Identify ANY entities or relationships that were MISSED in the first pass.

---Conversation History---
{conversation_history}

---Question---
Are there any additional entities or relationships that should be extracted?
If yes, provide them in the same format...
"""
```

**См.**: [P3: Entity Gleaning](../prompts/03-entity-gleaning.md)

### Multi-turn Refinement

```python
async def glean_attractors(chunk_text, initial_entities, max_rounds=2):
    """
    LLM refines attractors through multi-turn conversation.

    Pattern: Self-correction loop
    • Round 1: Initial extraction
    • Round 2+: "Did I miss anything?"
    """

    conversation_history = [
        {"role": "user", "content": chunk_text},
        {"role": "assistant", "content": format_entities(initial_entities)}
    ]

    refined_entities = initial_entities.copy()

    for round in range(max_rounds):
        # Ask LLM: "Did you miss anything?"
        gleaning_prompt = PROMPTS["entity_extraction_gleaning_prompt"].format(
            conversation_history=format_conversation(conversation_history)
        )

        response = await llm_func(gleaning_prompt)

        # Parse additional attractors
        additional_entities = parse_gleaning_output(response)

        if not additional_entities:
            break  # No more attractors found

        # Add to refined set
        refined_entities.extend(additional_entities)

        # Update conversation history
        conversation_history.append({
            "role": "user",
            "content": gleaning_prompt
        })
        conversation_history.append({
            "role": "assistant",
            "content": response
        })

    return refined_entities
```

**Код** (lightrag/operate.py:2091):
```python
# Gleaning loop
for round in range(entity_extract_max_gleaning):
    """
    ATTRACTOR REFINEMENT: Multi-turn gleaning

    Each round:
    1. Show LLM previous extraction + history
    2. Ask: "Anything missed?"
    3. Extract additional entities
    4. Update conversation history
    """

    # Gleaning prompt with conversation history
    glean_response = await llm_func(
        PROMPTS["entity_extraction_gleaning_prompt"],
        conversation_history=conversation_history,
        ...
    )

    # Parse additional attractors
    new_entities, new_relations = parse_gleaning(glean_response)

    if not new_entities and not new_relations:
        break  # No more refinements needed

    # Add to attractor set
    all_entities.extend(new_entities)
    all_relations.extend(new_relations)
```

### Semantic Properties

```
✅ Increases recall:
   Round 1: 10 entities extracted
   Round 2: +3 additional entities (gleaned)
   Total: 13 entities (30% improvement)

✅ Self-correction:
   LLM reviews own work, finds omissions

✅ Diminishing returns:
   Round 1 → Round 2: +15% recall
   Round 2 → Round 3: +3% recall
   (Usually 1-2 rounds sufficient)
```

---

## Pattern 3: Summary as Attractor Consolidation (Консолидация Аттракторов)

### Концепция

LLM **объединяет множественные описания** одного аттрактора в единое консолидированное представление.

```
Multiple Descriptions (of same entity)
    ↓
[Map-Reduce Summarization via P4]
    ↓
Consolidated Attractor Description
```

**Философия**: "Концепция проявляется многократно в разных контекстах - нужно консолидировать в единое представление."

### Map-Reduce Pattern

```python
async def consolidate_attractor(entity_name, descriptions, max_tokens=4000):
    """
    LLM consolidates multiple manifestations into single concept.

    Pattern: Map-Reduce
    • Map: Summarize chunks of descriptions
    • Reduce: Recursively combine until single summary
    """

    tokenizer = global_config["tokenizer"]
    total_tokens = sum(len(tokenizer.encode(d)) for d in descriptions)

    # Base case: Fits in context
    if total_tokens <= max_tokens:
        return await llm_summarize(entity_name, descriptions)

    # Recursive case: Map-Reduce

    # MAP phase: Split and summarize chunks
    chunks = split_descriptions_by_tokens(descriptions, max_tokens)
    chunk_summaries = await asyncio.gather(*[
        llm_summarize(entity_name, chunk)
        for chunk in chunks
    ])

    # REDUCE phase: Recursively consolidate summaries
    return await consolidate_attractor(
        entity_name,
        chunk_summaries,
        max_tokens
    )
```

**Код** (lightrag/operate.py:121):
```python
async def _handle_entity_relation_summary(
    entity_or_relation_name: str,
    description_list: list[str],
    global_config: dict,
    llm_response_cache: BaseKVStorage | None = None,
) -> tuple[str, bool]:
    """
    ATTRACTOR CONSOLIDATION: Map-Reduce summarization.

    Pattern:
    1. If descriptions fit in context → direct summary
    2. Else → split into chunks, summarize each, recursively reduce
    """

    # Map-Reduce loop
    while True:
        total_tokens = sum(tokenizer.encode(desc) for desc in current_list)

        if total_tokens <= summary_context_size:
            # Base case: Direct summary
            return await _summarize_descriptions(
                entity_or_relation_name,
                current_list,
                global_config,
                llm_response_cache
            )

        # Recursive case: Map phase
        chunks = split_into_chunks(current_list, summary_context_size)

        # Reduce phase: Summarize each chunk
        summaries = await asyncio.gather(*[
            _summarize_descriptions(entity_or_relation_name, chunk, ...)
            for chunk in chunks
        ])

        current_list = summaries  # Next iteration
```

### Prompt Architecture

**Summarization Prompt (P4)**:
```python
PROMPTS["summarize_entity_descriptions"] = """You are a text summarizer...

Entity/Relation Name: {description_name}
Type: {description_type}

Descriptions (JSONL format):
{description_list}

Task: Summarize into {summary_length} words in {language}.
Focus on core attributes and relationships.
Preserve important details, remove redundancy.
"""
```

**См.**: [P4: Entity Summarization](../prompts/04-entity-summarization.md)

### Semantic Properties

```
✅ Consolidates manifestations:
   Description 1: "Apple Inc founded 1976..."
   Description 2: "Apple's headquarters in Cupertino..."
   Description 3: "Apple produces iPhone, iPad..."
   → Consolidated: "Apple Inc, founded 1976 in Cupertino, produces..."

✅ Removes redundancy:
   Multiple mentions of same fact → single unified statement

✅ Scalable:
   Map-Reduce handles arbitrary number of descriptions
   No context length limitation
```

---

## Pattern 4: Keywords as Semantic Compass (Ключевые Слова как Семантический Компас)

### Концепция

LLM создает **иерархический семантический компас**, который направляет query к релевантным аттракторам.

```
User Query
    ↓
[Keywords Extraction via P5]
    ↓
High-level Attractors (themes, domains)
Low-level Attractors (specific entities, details)
```

**Философия**: "Query имеет многоуровневую структуру намерений - high-level темы и low-level детали."

### Hierarchical Decomposition

```python
async def extract_semantic_compass(query):
    """
    LLM decomposes query into hierarchical semantic keys.

    Output:
    • High-level: Abstract themes, domains
    • Low-level: Specific entities, attributes
    """

    examples = "\n".join(PROMPTS["keywords_extraction_examples"])

    prompt = PROMPTS["keywords_extraction"].format(
        query=query,
        examples=examples,
        language=language
    )

    response = await llm_func(prompt, keyword_extraction=True)

    keywords = json.loads(response)

    # keywords = {
    #     "high_level_keywords": ["Apple Inc", "products", "innovation"],
    #     "low_level_keywords": ["iPhone", "iPad", "user experience"]
    # }

    return keywords
```

**Код** (lightrag/operate.py:2478):
```python
async def extract_keywords_only(
    text: str,
    param: QueryParam,
    global_config: dict[str, str],
    hashing_kv: BaseKVStorage | None = None,
) -> tuple[list[str], list[str]]:
    """
    SEMANTIC COMPASS GENERATION: Extract hierarchical keywords.

    Pattern:
    • High-level: Activate major attractors (global search)
    • Low-level: Activate specific attractors (local search)
    """

    # LLM decomposes query
    kw_prompt = PROMPTS["keywords_extraction"].format(query=text, ...)
    result = await use_model_func(kw_prompt, keyword_extraction=True)

    keywords_data = json_repair.loads(result)

    hl_keywords = keywords_data.get("high_level_keywords", [])
    ll_keywords = keywords_data.get("low_level_keywords", [])

    return hl_keywords, ll_keywords
```

### Semantic Properties

```
✅ Hierarchical structure:
   Query: "What innovative products does Apple make?"

   High-level (themes):
   • "Apple Inc" (organization scope)
   • "products" (category)
   • "innovation" (quality)

   Low-level (specifics):
   • "iPhone" (specific product)
   • "design" (specific attribute)
   • "user experience" (specific quality)

✅ Mode-specific usage:
   Local mode → Use low-level (specific attractors)
   Global mode → Use high-level (thematic attractors)
   Hybrid → Use both

✅ Semantic guidance:
   Keywords = "compass needles" pointing to relevant attractors
```

---

## Pattern 5: Answer Generation as Concept Synthesis (Синтез Концепций)

### Концепция

LLM **синтезирует ответ** из activated attractors (entities + relations) и их manifestations (chunks).

```
Central Query (attractor in query space)
    ↓
Attracted Context:
    • Entities (concepts)
    • Relations (connections)
    • Chunks (manifestations)
    ↓
[RAG Response via P6]
    ↓
Synthesized Answer
```

**Философия**: "Query притягивает релевантные аттракторы, LLM синтезирует knowledge from attractor constellation."

### Synthesis Architecture

```python
async def synthesize_from_attractors(query, entities, relations, chunks):
    """
    LLM synthesizes answer from attractor constellation.

    Input:
    • Query: Central attractor in query space
    • Entities: Activated concept attractors
    • Relations: Connections between attractors
    • Chunks: Manifestations of attractors

    Output: Grounded, synthesized answer
    """

    # Format attractor constellation
    context = format_rag_context(
        entities=entities,
        relations=relations,
        chunks=chunks
    )

    # Synthesis prompt
    prompt = PROMPTS["rag_response"].format(
        query=query,
        context=context
    )

    # LLM synthesizes
    answer = await llm_func(
        prompt,
        system_prompt="You are a helpful AI assistant...",
        temperature=0.7  # ← Creative synthesis
    )

    return answer
```

**Код** (lightrag/operate.py:3280):
```python
async def _generate_answer(
    query: str,
    context: dict,
    query_param: QueryParam,
    ...
):
    """
    CONCEPT SYNTHESIS: Generate answer from attractor constellation.

    Pattern:
    • Entities (concepts) provide structure
    • Relations provide connections
    • Chunks (manifestations) provide grounding
    • LLM synthesizes cohesive answer
    """

    # Format context
    formatted_context = await _assemble_query_context(
        global_config,
        query_param,
        entities=context["entities"],    # ← Concepts
        relations=context["relations"],  # ← Connections
        chunks=context["chunks"],        # ← Manifestations
        ...
    )

    # Synthesis
    use_prompt = PROMPTS["rag_response"].format(
        query=query,
        context=formatted_context
    )

    answer = await llm_func(use_prompt, ...)

    return answer
```

### Semantic Properties

```
✅ Grounded synthesis:
   LLM must cite chunks (concrete evidence)
   Prevents hallucination

✅ Multi-attractor fusion:
   Combines info from multiple entities/relations
   Comprehensive answer

✅ Structured + unstructured:
   Entities/relations provide structure
   Chunks provide rich details
   Best of both worlds
```

---

## Comparative Analysis of LLM Patterns

| Pattern | Input | Output | Temperature | Role | Cost |
|---------|-------|--------|-------------|------|------|
| **Extraction (P1+P2)** | Unstructured text | Entities + relations | 0.0 | Identify attractors | Medium |
| **Gleaning (P3)** | Initial extraction + history | Additional entities | 0.0 | Refine attractors | High |
| **Summarization (P4)** | Multiple descriptions | Consolidated description | 0.0 | Consolidate attractor | Medium |
| **Keywords (P5)** | Query | High/low keywords | 0.0 | Guide to attractors | Low |
| **Synthesis (P6)** | Query + context | Answer | 0.7 | Synthesize from attractors | Medium |

---

## Attractor-Based LLM Philosophy

### Principle 1: LLM as Attractor Detector

LLM не просто "генерирует текст" - он **обнаруживает семантические аттракторы** в unstructured space.

### Principle 2: Multi-turn as Self-Refinement

Gleaning pattern показывает: LLM может **уточнять собственные результаты** через рефлексию.

### Principle 3: Map-Reduce as Scalable Consolidation

Summarization pattern: LLM может **консолидировать arbitrary amounts** of information через recursive decomposition.

### Principle 4: Hierarchical Decomposition

Keywords pattern: LLM понимает **multi-scale semantic structure** (high-level themes + low-level details).

### Principle 5: Grounded Synthesis

Answer generation: LLM **синтезирует из concrete evidence**, не "выдумывает".

---

**Версия**: 1.0
**Дата**: 2025-01-13
**См. также**: [LLM Agents](../06-llm-agents.md), [Prompts](../prompts/README.md), [Semantic Transformations](../transform/README.md)
