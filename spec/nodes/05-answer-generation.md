# Answer Generation: Формулировка Ответов на Основе Графа

## Обзор

Answer Generation - это финальный этап pipeline, где информация из Knowledge Graph, retrieved chunks и vector search объединяется LLM для создания comprehensive, accurate и well-sourced ответа на пользовательский запрос.

## Answer Generation Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│              ANSWER GENERATION PIPELINE                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Search Results (Subgraph + Chunks)                         │
│              │                                               │
│              ▼                                               │
│  ┌────────────────────────────────┐                         │
│  │ STAGE 1: CONTEXT ASSEMBLY      │                         │
│  │ • Subgraph extraction          │                         │
│  │ • Entity aggregation           │                         │
│  │ • Relationship chain building  │                         │
│  │ • Chunk collection             │                         │
│  └────────────┬───────────────────┘                         │
│               │                                              │
│               ▼                                              │
│  ┌────────────────────────────────┐                         │
│  │ STAGE 2: EVIDENCE COLLECTION   │                         │
│  │ • Source attribution           │                         │
│  │ • Citation tracking            │                         │
│  │ • Confidence scoring           │                         │
│  │ • Redundancy removal           │                         │
│  └────────────┬───────────────────┘                         │
│               │                                              │
│               ▼                                              │
│  ┌────────────────────────────────┐                         │
│  │ STAGE 3: ANSWER COMPOSITION    │                         │
│  │ • Template selection           │                         │
│  │ • LLM synthesis                │                         │
│  │ • Format structuring           │                         │
│  │ • Citation embedding           │                         │
│  └────────────┬───────────────────┘                         │
│               │                                              │
│               ▼                                              │
│  ┌────────────────────────────────┐                         │
│  │ STAGE 4: EXPLAINABILITY        │                         │
│  │ • Reasoning chain display      │                         │
│  │ • Source highlighting          │                         │
│  │ • Path visualization           │                         │
│  │ • Confidence indicators        │                         │
│  └────────────┬───────────────────┘                         │
│               │                                              │
│               ▼                                              │
│        Final Answer + Metadata                               │
└──────────────────────────────────────────────────────────────┘
```

## Stage 1: Context Assembly

### Subgraph Extraction

```python
async def assemble_subgraph_context(
    subgraph: dict,
    query: str
) -> dict:
    """
    Transform subgraph into structured context for LLM

    Input:
        subgraph = {
            "nodes": [entity nodes],
            "edges": [relationship edges]
        }

    Output:
        Structured context dict
    """

    # === ENTITY CONTEXT ===
    entities_context = []
    for node in subgraph["nodes"]:
        entities_context.append({
            "name": node["entity_name"],
            "type": node["entity_type"],
            "description": node["description"],
            "importance": calculate_importance(node)
        })

    # Sort by importance
    entities_context.sort(key=lambda x: x["importance"], reverse=True)

    # === RELATIONSHIP CONTEXT ===
    relationships_context = []
    for edge in subgraph["edges"]:
        relationships_context.append({
            "source": edge["source_entity"],
            "target": edge["target_entity"],
            "type": edge.get("keywords", "related_to"),
            "description": edge["description"]
        })

    # === GRAPH SUMMARY ===
    graph_summary = {
        "entity_count": len(subgraph["nodes"]),
        "relationship_count": len(subgraph["edges"]),
        "key_entities": [e["name"] for e in entities_context[:5]],
        "entity_types": list(set(e["type"] for e in entities_context))
    }

    return {
        "entities": entities_context,
        "relationships": relationships_context,
        "summary": graph_summary
    }
```

### Entity Aggregation

```python
def aggregate_entity_information(entities: list[dict]) -> list[dict]:
    """
    Aggregate information about same entity from different sources

    Handles:
    - Duplicate entity mentions
    - Conflicting information
    - Information completeness
    """

    entity_map = defaultdict(list)

    # Group by entity name
    for entity in entities:
        entity_map[entity["entity_name"]].append(entity)

    # Aggregate each entity
    aggregated = []
    for entity_name, entity_versions in entity_map.items():
        if len(entity_versions) == 1:
            aggregated.append(entity_versions[0])
        else:
            # Merge multiple versions
            merged = merge_entity_versions(entity_versions)
            aggregated.append(merged)

    return aggregated


def merge_entity_versions(versions: list[dict]) -> dict:
    """
    Merge multiple versions of same entity

    Strategy:
    1. Take most recent description
    2. Combine all source_ids
    3. Use highest confidence
    4. Merge metadata
    """

    # Sort by updated_at (most recent first)
    versions.sort(key=lambda x: x.get("updated_at", 0), reverse=True)

    base = versions[0].copy()

    # Combine source_ids
    all_source_ids = set()
    for v in versions:
        source_ids = v.get("source_id", "").split(",")
        all_source_ids.update(source_ids)

    base["source_id"] = ",".join(all_source_ids)

    # Use highest confidence
    base["confidence"] = max(
        v.get("confidence", 0.5) for v in versions
    )

    # Merge metadata
    base["mention_count"] = sum(
        v.get("mention_count", 1) for v in versions
    )

    return base
```

### Relationship Chain Building

```python
def build_relationship_chains(
    relationships: list[dict],
    target_entities: set[str]
) -> list[list[dict]]:
    """
    Build chains of relationships connecting target entities

    Example:
    Input: ["Tim Cook", "iPhone"]
    Output: [
        [
            {"source": "Tim Cook", "relation": "works_for", "target": "Apple Inc"},
            {"source": "Apple Inc", "relation": "produces", "target": "iPhone"}
        ]
    ]
    """

    # Build adjacency list
    graph = defaultdict(list)
    for rel in relationships:
        graph[rel["source"]].append(rel)

    chains = []

    # For each pair of target entities
    for entity1 in target_entities:
        for entity2 in target_entities:
            if entity1 == entity2:
                continue

            # Find paths using BFS
            paths = find_relationship_paths(graph, entity1, entity2, max_depth=3)

            for path in paths:
                chains.append(path)

    # Deduplicate and sort by length
    unique_chains = deduplicate_chains(chains)
    unique_chains.sort(key=lambda x: len(x))

    return unique_chains[:10]  # Top 10 shortest chains
```

### Chunk Collection

```python
async def collect_relevant_chunks(
    entities: list[dict],
    query: str,
    max_chunks: int = 20
) -> list[dict]:
    """
    Collect and rank chunks relevant to query and entities

    Ranking factors:
    1. Semantic similarity to query
    2. Entity mention overlap
    3. Chunk quality score
    4. Source diversity
    """

    all_chunks = []

    # Collect chunks for each entity
    for entity in entities:
        source_ids = entity.get("source_id", "").split(",")

        for source_id in source_ids:
            chunk = await text_chunks.get_by_id(source_id)
            if chunk:
                all_chunks.append(chunk)

    # Also add top semantic matches
    semantic_chunks = await chunks_vdb.query(query, top_k=10)
    all_chunks.extend(semantic_chunks)

    # Deduplicate by chunk_id
    unique_chunks = {chunk["chunk_id"]: chunk for chunk in all_chunks}
    chunks_list = list(unique_chunks.values())

    # Rank chunks
    ranked_chunks = rank_chunks_by_relevance(
        chunks_list,
        query,
        entities
    )

    return ranked_chunks[:max_chunks]


def rank_chunks_by_relevance(
    chunks: list[dict],
    query: str,
    entities: list[dict]
) -> list[dict]:
    """
    Multi-factor ranking of chunks

    Factors:
    1. Query similarity (vector)
    2. Entity mentions
    3. Chunk position (earlier = more important)
    4. Chunk quality (length, completeness)
    """

    entity_names = set(e["entity_name"] for e in entities)

    scored_chunks = []

    for chunk in chunks:
        # Query similarity
        if "score" in chunk:
            query_score = chunk["score"]
        else:
            query_score = 0.5

        # Entity mention count
        entity_mentions = sum(
            1 for entity in entity_names
            if entity.lower() in chunk["content"].lower()
        )
        entity_score = min(entity_mentions / len(entity_names), 1.0)

        # Position score (earlier chunks more important)
        position_score = 1.0 / (1 + chunk.get("chunk_order_index", 0) * 0.1)

        # Quality score
        token_count = chunk.get("tokens", 0)
        quality_score = min(token_count / 500, 1.0)  # Prefer 500+ token chunks

        # Combined score
        final_score = (
            0.4 * query_score +
            0.3 * entity_score +
            0.2 * position_score +
            0.1 * quality_score
        )

        scored_chunks.append((chunk, final_score))

    # Sort by score
    scored_chunks.sort(key=lambda x: x[1], reverse=True)

    return [chunk for chunk, score in scored_chunks]
```

---

## Stage 2: Evidence Collection

### Source Attribution

```python
def extract_source_attribution(
    entities: list[dict],
    relationships: list[dict],
    chunks: list[dict]
) -> dict:
    """
    Build comprehensive source attribution map

    Tracks:
    - Which chunks mention which entities
    - Which relationships came from which chunks
    - File paths for citations
    """

    attribution = {
        "entities": defaultdict(list),
        "relationships": defaultdict(list),
        "files": defaultdict(set)
    }

    # Entity attribution
    for entity in entities:
        source_ids = entity.get("source_id", "").split(",")
        file_path = entity.get("file_path", "unknown")

        attribution["entities"][entity["entity_name"]] = source_ids
        attribution["files"][file_path].add(entity["entity_name"])

    # Relationship attribution
    for rel in relationships:
        source_ids = rel.get("source_id", "").split(",")
        file_path = rel.get("file_path", "unknown")

        rel_key = f"{rel['source']}-{rel['target']}"
        attribution["relationships"][rel_key] = source_ids
        attribution["files"][file_path].add(rel_key)

    return dict(attribution)
```

### Citation Tracking

```python
def generate_citations(
    answer_text: str,
    chunks: list[dict],
    entities: list[dict]
) -> tuple[str, list[dict]]:
    """
    Add inline citations to answer text

    Format: [1], [2], [3]

    Returns:
        (answer_with_citations, citation_list)
    """

    citations = []
    cited_chunks = {}

    # Build citation map (chunk_id -> citation number)
    for i, chunk in enumerate(chunks, start=1):
        chunk_id = chunk["chunk_id"]
        cited_chunks[chunk_id] = {
            "number": i,
            "source": chunk.get("file_path", "unknown"),
            "text": chunk["content"][:200] + "..."  # Preview
        }

    # Insert citations in answer
    # (Simplified - actual implementation would be more sophisticated)
    answer_with_citations = answer_text

    # Generate citation list
    citation_list = [
        {
            "id": i,
            "source": cited_chunks[chunk_id]["source"],
            "preview": cited_chunks[chunk_id]["text"]
        }
        for chunk_id, info in cited_chunks.items()
        for i in [info["number"]]
    ]

    return answer_with_citations, citation_list
```

### Confidence Scoring

```python
def calculate_answer_confidence(
    context: dict,
    answer: str
) -> dict:
    """
    Calculate confidence metrics for answer

    Factors:
    1. Entity coverage (% of query entities in context)
    2. Source count (number of supporting chunks)
    3. Consistency (agreement between sources)
    4. Completeness (answer addresses all query aspects)
    """

    # Entity coverage
    query_entities = extract_entities_from_text(context["query"])
    found_entities = [
        e for e in query_entities
        if any(e.lower() in ent["entity_name"].lower()
               for ent in context["entities"])
    ]
    entity_coverage = len(found_entities) / len(query_entities) if query_entities else 0

    # Source count score
    source_count = len(context["chunks"])
    source_score = min(source_count / 5, 1.0)  # 5+ sources = max

    # Consistency score (simplified)
    # In practice: check for contradictions between chunks
    consistency_score = 0.8  # Placeholder

    # Completeness score
    # Check if answer mentions key entities
    key_entities = [e["entity_name"] for e in context["entities"][:5]]
    mentioned = sum(1 for e in key_entities if e.lower() in answer.lower())
    completeness_score = mentioned / len(key_entities) if key_entities else 0

    # Overall confidence
    overall_confidence = (
        0.3 * entity_coverage +
        0.2 * source_score +
        0.3 * consistency_score +
        0.2 * completeness_score
    )

    return {
        "overall": overall_confidence,
        "entity_coverage": entity_coverage,
        "source_score": source_score,
        "consistency": consistency_score,
        "completeness": completeness_score,
        "level": get_confidence_level(overall_confidence)
    }


def get_confidence_level(score: float) -> str:
    """Convert confidence score to level"""
    if score >= 0.8:
        return "high"
    elif score >= 0.6:
        return "medium"
    else:
        return "low"
```

---

## Stage 3: Answer Composition

### Template Selection

```python
ANSWER_TEMPLATES = {
    "factual": """
Based on the provided context, {answer_statement}.

This information is supported by {source_count} sources{file_citation}.
""",

    "analytical": """
{opening_statement}

Key insights:
{key_points}

Analysis:
{detailed_analysis}

This conclusion is drawn from {source_count} sources across the knowledge graph{file_citation}.
""",

    "comparative": """
Comparing {entity1} and {entity2}:

Similarities:
{similarities}

Differences:
{differences}

{conclusion}
""",

    "explanatory": """
{main_explanation}

This relationship can be understood through the following connections:
{relationship_chain}

Supporting evidence:
{evidence_points}
"""
}


def select_answer_template(
    query: str,
    context: dict
) -> str:
    """
    Select appropriate answer template based on query type

    Query types:
    - factual: "What is X?"
    - analytical: "How/Why does X?"
    - comparative: "Compare X and Y"
    - explanatory: "Explain the relationship between X and Y"
    """

    query_lower = query.lower()

    if any(word in query_lower for word in ["compare", "difference", "vs", "versus"]):
        return "comparative"

    elif any(word in query_lower for word in ["how", "why", "explain"]):
        if "relationship" in query_lower or "connection" in query_lower:
            return "explanatory"
        else:
            return "analytical"

    else:
        return "factual"
```

### LLM Synthesis

```python
async def synthesize_answer_with_llm(
    query: str,
    context: dict,
    template_type: str
) -> str:
    """
    Generate answer using LLM with structured context

    Process:
    1. Format context into prompt
    2. Select appropriate system prompt
    3. Call LLM
    4. Post-process response
    """

    # === FORMAT CONTEXT ===
    context_text = format_context_for_llm(context)

    # === SYSTEM PROMPT ===
    system_prompt = PROMPTS["rag_response"].format(
        template_type=template_type
    )

    # === USER PROMPT ===
    user_prompt = f"""
Query: {query}

Knowledge Graph Context:
{context_text}

Instructions:
1. Provide a comprehensive answer based on the knowledge graph
2. Include relevant entities and their relationships
3. Use specific details from the provided chunks
4. Maintain objectivity and factual accuracy
5. Structure the answer according to the {template_type} format

Answer:
"""

    # === LLM CALL ===
    answer = await llm_func(
        user_prompt,
        system_prompt=system_prompt,
        max_tokens=1000,
        temperature=0.3  # Low temperature for factual accuracy
    )

    # === POST-PROCESSING ===
    answer = post_process_answer(answer, context)

    return answer


def format_context_for_llm(context: dict) -> str:
    """
    Format structured context into LLM-readable text

    Includes:
    - Entity descriptions
    - Relationship descriptions
    - Relevant text chunks
    """

    sections = []

    # === ENTITIES SECTION ===
    if context.get("entities"):
        entities_text = "### Entities:\n\n"
        for entity in context["entities"][:10]:  # Top 10
            entities_text += f"- **{entity['entity_name']}** ({entity['entity_type']}): {entity['description']}\n"
        sections.append(entities_text)

    # === RELATIONSHIPS SECTION ===
    if context.get("relationships"):
        relations_text = "### Relationships:\n\n"
        for rel in context["relationships"][:15]:  # Top 15
            relations_text += f"- {rel['source']} → {rel['target']}: {rel['description']}\n"
        sections.append(relations_text)

    # === TEXT CHUNKS SECTION ===
    if context.get("chunks"):
        chunks_text = "### Supporting Text:\n\n"
        for i, chunk in enumerate(context["chunks"][:10], start=1):
            chunks_text += f"{i}. {chunk['content']}\n\n"
        sections.append(chunks_text)

    return "\n".join(sections)
```

### Format Structuring

```python
def structure_answer(
    raw_answer: str,
    template_type: str,
    context: dict
) -> dict:
    """
    Structure answer into sections based on template

    Output format:
    {
        "main_answer": str,
        "sections": [
            {"title": str, "content": str}
        ],
        "key_points": [str],
        "entities_mentioned": [str]
    }
    """

    structured = {
        "main_answer": raw_answer,
        "sections": [],
        "key_points": [],
        "entities_mentioned": []
    }

    # Parse answer into sections (if formatted with headers)
    sections = parse_markdown_sections(raw_answer)
    structured["sections"] = sections

    # Extract key points (bullet points or numbered lists)
    key_points = extract_bullet_points(raw_answer)
    structured["key_points"] = key_points

    # Identify mentioned entities
    mentioned_entities = []
    for entity in context["entities"]:
        if entity["entity_name"].lower() in raw_answer.lower():
            mentioned_entities.append(entity["entity_name"])

    structured["entities_mentioned"] = mentioned_entities

    return structured
```

---

## Stage 4: Explainability

### Reasoning Chain Display

```python
def generate_reasoning_chain(
    query: str,
    context: dict,
    answer: str
) -> list[dict]:
    """
    Generate step-by-step reasoning chain

    Shows:
    1. Query analysis
    2. Entity identification
    3. Graph traversal
    4. Evidence collection
    5. Answer synthesis
    """

    chain = []

    # Step 1: Query Analysis
    chain.append({
        "step": 1,
        "action": "Query Analysis",
        "description": f"Analyzed query: '{query}'",
        "result": f"Identified {len(context.get('entities', []))} relevant entities"
    })

    # Step 2: Entity Identification
    key_entities = [e["entity_name"] for e in context.get("entities", [])[:5]]
    chain.append({
        "step": 2,
        "action": "Entity Identification",
        "description": "Found key entities in knowledge graph",
        "result": ", ".join(key_entities)
    })

    # Step 3: Graph Traversal
    chain.append({
        "step": 3,
        "action": "Graph Traversal",
        "description": f"Explored {len(context.get('relationships', []))} relationships",
        "result": "Built local subgraph around key entities"
    })

    # Step 4: Evidence Collection
    chain.append({
        "step": 4,
        "action": "Evidence Collection",
        "description": f"Retrieved {len(context.get('chunks', []))} relevant text chunks",
        "result": "Gathered supporting evidence from source documents"
    })

    # Step 5: Answer Synthesis
    chain.append({
        "step": 5,
        "action": "Answer Synthesis",
        "description": "Combined knowledge graph and text evidence",
        "result": "Generated comprehensive answer"
    })

    return chain
```

### Path Visualization

```python
def visualize_reasoning_paths(
    relationships: list[dict],
    key_entities: list[str]
) -> str:
    """
    Generate ASCII visualization of reasoning paths

    Example output:
    Tim Cook -[works_for]-> Apple Inc -[produces]-> iPhone
    """

    # Find shortest paths between key entities
    paths = find_paths_between_entities(relationships, key_entities)

    visualization = "### Reasoning Paths:\n\n"

    for i, path in enumerate(paths[:5], start=1):
        path_str = format_path_as_string(path)
        visualization += f"{i}. {path_str}\n"

    return visualization


def format_path_as_string(path: list[dict]) -> str:
    """
    Format path as: Entity1 -[relation]-> Entity2 -[relation]-> Entity3
    """

    if not path:
        return ""

    segments = []
    for rel in path:
        segment = f"{rel['source']} -[{rel.get('keywords', 'related')}]-> {rel['target']}"
        segments.append(segment)

    # Deduplicate consecutive entities
    result = segments[0]
    for seg in segments[1:]:
        # Extract target from previous and source from current
        # Append only if different
        result += " " + seg.split("->")[1]

    return result
```

### Confidence Indicators

```python
def add_confidence_indicators(
    answer: str,
    confidence: dict
) -> str:
    """
    Add confidence indicators to answer

    Formats:
    - High confidence: ✓ (strong evidence)
    - Medium confidence: ~ (moderate evidence)
    - Low confidence: ? (limited evidence)
    """

    confidence_level = confidence["level"]

    indicator = {
        "high": "✓",
        "medium": "~",
        "low": "?"
    }[confidence_level]

    confidence_text = f"\n\n**Confidence: {indicator} {confidence_level.title()}**\n"
    confidence_text += f"- Entity Coverage: {confidence['entity_coverage']:.0%}\n"
    confidence_text += f"- Sources: {int(confidence['source_score'] * 5)}\n"
    confidence_text += f"- Consistency: {confidence['consistency']:.0%}\n"

    return answer + confidence_text
```

---

## Complete Answer Object

```python
AnswerObject = {
    "answer": {
        "text": str,                    # Main answer text
        "structured": {                # Structured version
            "sections": list[dict],
            "key_points": list[str],
            "entities_mentioned": list[str]
        }
    },
    "metadata": {
        "query": str,                  # Original query
        "mode": str,                   # "naive", "local", "global"
        "processing_time": float,      # Milliseconds
        "timestamp": int               # Unix timestamp
    },
    "context": {
        "entities": list[dict],        # Entities used
        "relationships": list[dict],   # Relationships used
        "chunks": list[dict],          # Chunks used
        "subgraph_size": {
            "nodes": int,
            "edges": int
        }
    },
    "evidence": {
        "citations": list[dict],       # Source citations
        "attribution": dict,           # Source attribution map
        "confidence": dict             # Confidence scores
    },
    "explainability": {
        "reasoning_chain": list[dict], # Step-by-step reasoning
        "paths": list[str],            # Reasoning paths
        "confidence_indicators": dict  # Confidence details
    }
}
```

### Example Complete Answer

```json
{
    "answer": {
        "text": "Apple Inc manufactures several major product lines including iPhone (smartphones), iPad (tablets), Mac (computers), Apple Watch (smartwatches), and AirPods (wireless earbuds). The company is headquartered in Cupertino, California and is led by CEO Tim Cook.",
        "structured": {
            "sections": [
                {
                    "title": "Products",
                    "content": "iPhone, iPad, Mac, Apple Watch, AirPods"
                },
                {
                    "title": "Leadership",
                    "content": "CEO Tim Cook"
                }
            ],
            "key_points": [
                "Five major product lines",
                "Headquartered in Cupertino",
                "Led by Tim Cook"
            ],
            "entities_mentioned": ["Apple Inc", "iPhone", "iPad", "Mac", "Apple Watch", "AirPods", "Tim Cook", "Cupertino"]
        }
    },
    "metadata": {
        "query": "What products does Apple Inc manufacture?",
        "mode": "local",
        "processing_time": 245.7,
        "timestamp": 1705012345
    },
    "context": {
        "entities": [
            {
                "entity_name": "Apple Inc",
                "entity_type": "organization",
                "description": "..."
            }
        ],
        "relationships": [
            {
                "source": "Apple Inc",
                "target": "iPhone",
                "type": "produces"
            }
        ],
        "chunks": [...],
        "subgraph_size": {
            "nodes": 15,
            "edges": 12
        }
    },
    "evidence": {
        "citations": [
            {
                "id": 1,
                "source": "apple_overview.pdf",
                "preview": "Apple Inc designs and manufactures..."
            }
        ],
        "confidence": {
            "overall": 0.92,
            "level": "high",
            "entity_coverage": 1.0,
            "source_score": 0.8,
            "consistency": 0.95,
            "completeness": 0.95
        }
    },
    "explainability": {
        "reasoning_chain": [
            {
                "step": 1,
                "action": "Query Analysis",
                "result": "Identified 'Apple Inc' and 'products'"
            }
        ],
        "paths": [
            "Apple Inc -[produces]-> iPhone",
            "Apple Inc -[produces]-> iPad"
        ]
    }
}
```

---

## Best Practices

### ✅ DO

1. **Include citations** for all factual claims
2. **Show confidence levels** to set expectations
3. **Provide reasoning chains** for transparency
4. **Structure answers** for readability
5. **Mention key entities** from graph
6. **Use multiple sources** for verification

### ❌ DON'T

1. **Don't fabricate information** not in context
2. **Don't ignore low confidence** - communicate uncertainty
3. **Don't omit sources** - always cite
4. **Don't over-complicate** - keep language clear
5. **Don't mix facts** from conflicting sources without noting

---

## Performance Metrics

### Answer Quality Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| **Accuracy** | >0.85 | Factual correctness |
| **Completeness** | >0.80 | Addresses all query aspects |
| **Clarity** | >0.75 | Readability score |
| **Citation Rate** | 100% | % of claims cited |
| **Confidence** | >0.70 | Average confidence |

### Generation Time

| Mode | Target Time | Typical |
|------|------------|---------|
| **Naive** | <200ms | 100-150ms |
| **Local** | <500ms | 300-400ms |
| **Global** | <1500ms | 800-1200ms |

---

**End of Answer Generation Documentation**

This completes the comprehensive documentation of LightRAG's node types, relationships, graph topology, search patterns, and answer generation mechanisms.
