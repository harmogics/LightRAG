# Query Transformations: Преобразования при Обработке Запросов

## Обзор

Query Pipeline выполняет 5 основных семантических преобразований для конвертации пользовательского запроса в структурированный, well-sourced ответ.

```
User Query → [T7] → Keywords → [T8] → Entities → [T9] → Context
                                                            ↓
                                                          [T10]
                                                            ↓
                                                       Raw Answer
                                                            ↓
                                                          [T11]
                                                            ↓
                                                   Structured Answer
```

## T7: Keyword Extraction (Intent Analysis)

### Semantic Operation

**Type**: EXTRACTION (Intent → Keywords)

**Semantic Change**: Natural language query → Structured keywords (high/low level)

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | User query (natural language) |
| **Output** | Hierarchical keywords (high/low level) |
| **Input Length** | 5-100 words |
| **Output Count** | 3-10 keywords per level |

### Transformation Details

```python
Input:
{
    "query": "What products does Apple Inc manufacture and how do they use AI?"
}

Transform: extract_keywords_with_llm()
Prompt: keywords_extraction
Model: GPT-4

Output:
{
    "high_level": [
        "Apple Inc",
        "products",
        "manufacturing",
        "AI technology"
    ],
    "low_level": [
        "iPhone",
        "iPad",
        "Mac",
        "machine learning",
        "artificial intelligence",
        "computational photography",
        "voice recognition"
    ]
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Intent Capture** | 85-95% | Identifies user intent |
| **Information Expansion** | +20-30% | Adds related terms |
| **Abstraction** | Medium | High-level concepts |
| **Specificity** | High | Low-level details |
| **Deterministic** | No | LLM-based |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Keyword Extraction Agent** | Primary | Extracts keywords |
| **LLM (GPT-4)** | Executor | Generates keywords |
| **Query Analyzer** | Support | Analyzes intent |

### Information Flow

```
Information Type: Intent Analysis

Input Information:
- Query text: 100%
- Implicit intent: ~70%
- Context: Limited

Extraction Process:
- Identify main entities: 80-90%
- Extract concepts: 70-80%
- Hierarchical organization
- Add related terms: +20%

Output Information:
- High-level keywords: Core intent
- Low-level keywords: Specific details
- Semantic expansion: +20-30%

Net Change: 100% → 120-130% (expansion)
```

### Mode Usage

| Query Mode | Usage | Keyword Role |
|------------|-------|--------------|
| **Naive** | Not used | Direct vector search |
| **Local** | High-level only | Entity identification |
| **Global** | Both levels | Comprehensive search |
| **Hybrid** | Adaptive | Based on complexity |

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 1-3s per query |
| **LLM Calls** | 1 per query |
| **Token Usage** | ~800 tokens |
| **Cache Hit Rate** | 10-20% (queries vary) |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Intent Accuracy** | >85% | 88-94% |
| **Keyword Relevance** | >80% | 83-89% |
| **Coverage** | >90% | 92-96% |
| **Redundancy** | <20% | 15-18% |
| **Expansion Quality** | >75% | 78-85% |

### Role in Chain

**Position**: First transformation

**Criticality**: HIGH - drives entity search

**Dependencies**: None

**Dependents**: T8 (Entity Resolution) - uses keywords

**Optimization Impact**:
- Good keywords → Better entity matches
- Poor keywords → Irrelevant results

---

## T8: Entity Resolution (Disambiguation)

### Semantic Operation

**Type**: EXTRACTION (Keywords → Entities)

**Semantic Change**: Keywords → Concrete entities from graph

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Keywords (structured) |
| **Output** | Ranked entity list with metadata |
| **Search Type** | Vector similarity + Graph lookup |
| **Output Count** | 5-20 entities |

### Transformation Details

```python
Input:
{
    "keywords": {
        "high_level": ["Apple Inc", "products", "AI technology"],
        "low_level": ["iPhone", "machine learning"]
    }
}

Transform: entity_vdb.query() + knowledge_graph.get_node()
Operations:
1. Vector search for each high-level keyword
2. Combine results
3. Deduplicate by entity_name
4. Rank by relevance
5. Enrich with graph metadata

Output:
[
    {
        "entity_id": "ent-apple-inc",
        "entity_name": "Apple Inc",
        "entity_type": "organization",
        "description": "Apple Inc is a multinational technology company...",
        "relevance_score": 0.92,
        "source_id": "chunk-xyz789",
        "file_path": "apple_overview.pdf"
    },
    {
        "entity_id": "ent-iphone",
        "entity_name": "iPhone",
        "entity_type": "product",
        "description": "iPhone is a line of smartphones...",
        "relevance_score": 0.88,
        "source_id": "chunk-abc123",
        "file_path": "products.pdf"
    },
    {
        "entity_id": "ent-ml",
        "entity_name": "Machine Learning",
        "entity_type": "concept",
        "description": "Machine Learning is a subset of AI...",
        "relevance_score": 0.85,
        "source_id": "chunk-def456",
        "file_path": "ai_tech.pdf"
    }
]
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Grounding** | High | Connects to actual entities |
| **Disambiguation** | 80-90% | Resolves entity ambiguity |
| **Precision** | 75-85% | Accurate entity selection |
| **Recall** | 70-80% | Finds relevant entities |
| **Deterministic** | Mostly | Vector search has variance |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Entity Resolver** | Primary | Maps keywords to entities |
| **Vector DB** | Support | Semantic search |
| **Graph DB** | Support | Entity enrichment |
| **LLM** | None | Not directly involved |

### Information Flow

```
Information Type: Entity Grounding

Input Information:
- High-level keywords: 4-8 terms
- Low-level keywords: 5-15 terms
- Query context: Implicit

Resolution Process:
- Vector search: Top-K per keyword
- Merge results: 20-50 candidates
- Deduplicate: 10-20 unique entities
- Rank by relevance
- Enrich with metadata

Output Information:
- Grounded entities: 5-20 entities
- Entity metadata: Full descriptions
- Relevance scores: Ranking signal

Net Change: Keywords → Entities (grounding)
```

### Mode Usage

| Query Mode | Top-K | Usage Pattern |
|------------|-------|---------------|
| **Naive** | 0 | Not used |
| **Local** | 5-10 | Seed entities |
| **Global** | 5-15 | Seed + expansion |
| **Hybrid** | Variable | Adaptive |

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 50-150ms per query |
| **Vector Searches** | 1-5 (per keyword batch) |
| **Graph Lookups** | 5-20 (per entity) |
| **Throughput** | 10-20 queries/sec |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Entity Precision** | >80% | 83-89% |
| **Entity Recall** | >75% | 77-83% |
| **Disambiguation Accuracy** | >85% | 87-92% |
| **Relevance Correlation** | >0.75 | 0.78-0.85 |

### Role in Chain

**Position**: Second transformation

**Criticality**: CRITICAL - bridges query to graph

**Dependencies**: T7 (Keywords) - uses extracted keywords

**Dependents**: T9 (Context Assembly) - uses resolved entities

**Optimization Impact**:
- Good resolution → Relevant subgraph
- Poor resolution → Off-topic results

---

## T9: Context Assembly (Aggregation)

### Semantic Operation

**Type**: AGGREGATION (Entities + Graph → Structured Context)

**Semantic Change**: Scattered graph data → Unified context for LLM

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Entities + Relationships + Chunks |
| **Output** | Structured context document |
| **Input Sources** | 3 (graph nodes, edges, text chunks) |
| **Output Format** | Hierarchical dict |

### Transformation Details

```python
Input:
{
    "entities": [
        {"entity_name": "Apple Inc", "description": "..."},
        {"entity_name": "iPhone", "description": "..."}
    ],
    "relationships": [
        {
            "source": "Apple Inc",
            "target": "iPhone",
            "keywords": "production, manufacturing",
            "description": "Apple Inc manufactures iPhone"
        }
    ],
    "chunks": [
        {"content": "Apple Inc designs and manufactures iPhone...", "score": 0.89},
        {"content": "iPhone uses machine learning for...", "score": 0.85}
    ]
}

Transform: assemble_context()
Operations:
1. Extract subgraph
2. Aggregate entity information
3. Build relationship chains
4. Collect relevant chunks
5. Rank and filter
6. Format for LLM

Output:
{
    "query": "What products does Apple Inc manufacture...",
    "entities": [
        {
            "name": "Apple Inc",
            "type": "organization",
            "description": "Merged comprehensive description",
            "importance": 0.95
        },
        {
            "name": "iPhone",
            "type": "product",
            "description": "...",
            "importance": 0.88
        }
    ],
    "relationships": [
        {
            "source": "Apple Inc",
            "target": "iPhone",
            "type": "produces",
            "description": "...",
            "strength": 0.92
        }
    ],
    "chunks": [
        {
            "id": 1,
            "content": "...",
            "relevance": 0.89,
            "source": "apple_overview.pdf"
        }
    ],
    "summary": {
        "entity_count": 5,
        "relationship_count": 8,
        "chunk_count": 10,
        "key_entities": ["Apple Inc", "iPhone", "Machine Learning"]
    }
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Aggregation** | High | Combines multiple sources |
| **Redundancy Removal** | 70-80% | Deduplicates information |
| **Coherence** | High | Organizes logically |
| **Relevance Filtering** | 80-90% | Keeps relevant info |
| **Deterministic** | Mostly | Some ranking variance |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Context Assembler** | Primary | Aggregates context |
| **Graph Traversal** | Support | Extracts subgraph |
| **Ranking Engine** | Support | Ranks by relevance |
| **LLM** | None | Not directly involved |

### Information Flow

```
Information Type: Multi-Source Aggregation

Input Information:
- Entities: 5-20 nodes
- Relationships: 10-50 edges
- Chunks: 10-30 text fragments
- Total: ~100KB raw data

Assembly Process:
- Subgraph extraction: Select relevant nodes/edges
- Entity aggregation: Merge duplicates
- Relationship chains: Connect entities
- Chunk ranking: Top-K selection
- Format structuring: Organize hierarchically

Output Information:
- Structured context: ~20-50KB
- Coherent narrative: Organized
- Redundancy removed: 70-80%
- LLM-ready format: JSON/Text

Net Change: 100KB → 30KB (filtered + structured)
```

### Mode Usage

| Query Mode | Subgraph Size | Context Size | Assembly Complexity |
|------------|--------------|--------------|---------------------|
| **Naive** | 0 nodes, 0 edges | Chunks only | Minimal |
| **Local** | 10-20 nodes, 15-30 edges | Medium | Moderate |
| **Global** | 50-100 nodes, 100-200 edges | Large | High |
| **Hybrid** | Variable | Variable | Adaptive |

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 100-500ms |
| **Graph Queries** | 1-5 (traversal) |
| **Chunk Retrievals** | 10-30 |
| **Memory** | 10-50MB |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Context Relevance** | >85% | 87-93% |
| **Completeness** | >80% | 82-88% |
| **Coherence Score** | >75% | 78-85% |
| **Redundancy** | <20% | 15-18% |
| **LLM Readability** | >90% | 92-96% |

### Role in Chain

**Position**: Third transformation

**Criticality**: HIGH - prepares LLM input

**Dependencies**: T8 (Entity Resolution) - uses entities

**Dependents**: T10 (Answer Generation) - provides context

**Optimization Impact**:
- Good context → Accurate answer
- Poor context → Hallucination risk

---

## T10: Answer Generation (Synthesis)

### Semantic Operation

**Type**: SYNTHESIS (Context → Natural Language Answer)

**Semantic Change**: Structured context → Coherent natural language answer

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Structured context (entities + relations + chunks) |
| **Output** | Natural language answer |
| **Input Size** | 20-50KB structured data |
| **Output Size** | 500-2000 words |

### Transformation Details

```python
Input:
{
    "query": "What products does Apple Inc manufacture and how do they use AI?",
    "context": {
        "entities": [...],
        "relationships": [...],
        "chunks": [...]
    }
}

Transform: llm_func()
Prompt: rag_response
Model: GPT-4
System Prompt: Answer generation instructions

Output:
"""
Apple Inc manufactures several major product lines that extensively utilize
artificial intelligence technologies:

**Products:**
1. **iPhone**: Apple's smartphone line incorporates machine learning for
   computational photography, Face ID recognition, and Siri voice assistant.

2. **iPad**: The tablet series uses AI for Apple Pencil prediction,
   handwriting recognition, and augmented reality applications.

3. **Mac**: Apple's computer line integrates the Neural Engine in M-series
   chips for AI-accelerated tasks.

**AI Integration:**
- **Computational Photography**: Machine learning algorithms enhance photo
  quality, enable portrait mode, and provide night mode capabilities.

- **Voice Recognition**: Siri uses natural language processing and speech
  recognition powered by neural networks.

- **Privacy-Focused AI**: On-device machine learning ensures user data
  privacy while providing intelligent features.

This information is supported by multiple sources including product
documentation and technical specifications [1][2][3].
"""
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Synthesis** | High | Combines multiple facts |
| **Coherence** | Very High | Natural narrative flow |
| **Grounding** | 85-95% | Based on context |
| **Hallucination Risk** | 5-15% | Some invented details |
| **Deterministic** | No | LLM variance |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Answer Generation Agent** | Primary | Creates answer |
| **LLM (GPT-4)** | Executor | Generates text |
| **Context Formatter** | Support | Prepares LLM input |

### Information Flow

```
Information Type: Synthesis

Input Information:
- Context data: 100%
- Query intent: 100%
- Implicit knowledge: LLM's training

Generation Process:
- Understand query: Intent analysis
- Synthesize facts: Combine context
- Generate narrative: Natural language
- Add structure: Headings, lists
- Include citations: Source attribution

Output Information:
- Answer text: Comprehensive
- Structure: Organized
- Citations: Source-grounded
- Coherence: Narrative flow

Net Change: Structured → Natural Language
```

### Mode Usage

| Query Mode | Context Size | Answer Length | Complexity |
|------------|--------------|---------------|------------|
| **Naive** | Small (chunks) | Short (100-300w) | Simple |
| **Local** | Medium | Medium (300-800w) | Moderate |
| **Global** | Large | Long (800-2000w) | Complex |
| **Hybrid** | Variable | Variable | Adaptive |

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 2-8s |
| **LLM Calls** | 1 per query |
| **Token Usage** | 2000-5000 tokens |
| **Cost** | $0.01-0.05 per query |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Accuracy** | >85% | 87-93% |
| **Completeness** | >80% | 82-88% |
| **Coherence** | >90% | 92-96% |
| **Relevance** | >85% | 87-92% |
| **Grounding** | >85% | 87-94% |
| **Hallucination Rate** | <15% | 8-12% |

### Role in Chain

**Position**: Fourth transformation

**Criticality**: CRITICAL - creates final answer

**Dependencies**: T9 (Context Assembly) - requires context

**Dependents**: T11 (Formatting) - formats output

**Optimization Impact**:
- Good prompt → Better answer quality
- Good context → Higher accuracy

---

## T11: Answer Formatting (Structuring)

### Semantic Operation

**Type**: TRANSFORMATION (Format Enhancement)

**Semantic Change**: Raw answer → Structured answer with metadata

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Raw answer text |
| **Output** | Structured answer object |
| **Format Change** | Text → JSON object with sections |

### Transformation Details

```python
Input:
"""
Apple Inc manufactures several major product lines that extensively utilize
artificial intelligence technologies:

**Products:**
1. iPhone
2. iPad
3. Mac

**AI Integration:**
- Computational Photography
- Voice Recognition
- Privacy-Focused AI
"""

Transform: structure_answer()
Operations:
1. Parse markdown structure
2. Extract sections
3. Identify key points
4. Extract entity mentions
5. Add citations
6. Calculate confidence
7. Generate reasoning chain

Output:
{
    "answer": {
        "text": "Apple Inc manufactures several major product lines...",
        "structured": {
            "sections": [
                {"title": "Products", "content": "..."},
                {"title": "AI Integration", "content": "..."}
            ],
            "key_points": [
                "iPhone uses ML for photography",
                "Siri uses voice recognition",
                "On-device AI for privacy"
            ],
            "entities_mentioned": [
                "Apple Inc", "iPhone", "iPad", "Mac", "Siri"
            ]
        }
    },
    "metadata": {
        "query": "...",
        "mode": "local",
        "processing_time": 3245.7,
        "timestamp": 1705012345
    },
    "evidence": {
        "citations": [
            {"id": 1, "source": "apple_products.pdf", "preview": "..."},
            {"id": 2, "source": "ai_features.pdf", "preview": "..."}
        ],
        "confidence": {
            "overall": 0.89,
            "level": "high",
            "entity_coverage": 0.95,
            "source_score": 0.85
        }
    },
    "explainability": {
        "reasoning_chain": [
            {"step": 1, "action": "Query Analysis", "result": "..."},
            {"step": 2, "action": "Entity Identification", "result": "..."}
        ],
        "paths": [
            "Apple Inc -[produces]-> iPhone",
            "iPhone -[uses]-> Machine Learning"
        ]
    }
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Preservation** | 100% | No content loss |
| **Structure Addition** | High | Adds rich metadata |
| **Explainability** | High | Reasoning transparent |
| **Machine Readability** | High | JSON format |
| **Deterministic** | Yes | Consistent parsing |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Formatter** | Primary | Structures output |
| **Parser** | Support | Parses markdown |
| **Metadata Builder** | Support | Adds metadata |
| **LLM** | None | Not involved |

### Information Flow

```
Information Type: Structural Enhancement

Input Information:
- Answer text: 100%
- Implicit structure: Markdown

Formatting Process:
- Parse markdown: Extract sections
- Extract key points: Bullets/numbered
- Identify entities: Mention detection
- Add citations: Source linking
- Calculate confidence: Multi-factor
- Build reasoning chain: Step-by-step

Output Information:
- Original text: 100% preserved
- Structured sections: NEW
- Key points: NEW
- Entity mentions: NEW
- Citations: NEW
- Confidence: NEW
- Reasoning: NEW

Net Change: 100% → 200%+ (enrichment)
```

### Mode Usage

| Query Mode | Formatting Complexity | Metadata Richness |
|------------|---------------------|-------------------|
| **Naive** | Minimal | Basic |
| **Local** | Moderate | Medium |
| **Global** | High | Rich |
| **Hybrid** | Variable | Adaptive |

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 10-50ms |
| **Memory** | Low |
| **CPU** | Low |
| **Deterministic** | Yes |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Parse Accuracy** | >95% | 97-99% |
| **Structure Quality** | >90% | 92-96% |
| **Metadata Completeness** | >85% | 87-93% |
| **JSON Validity** | 100% | 100% |

### Role in Chain

**Position**: Fifth transformation (final)

**Criticality**: MEDIUM - enhances usability

**Dependencies**: T10 (Answer Generation) - formats its output

**Dependents**: None (terminal)

**Optimization Impact**:
- Good formatting → Better UX
- Rich metadata → Better explainability

---

## Summary Comparison Table

| Transform | Type | Input → Output | Agent | Latency | Query Mode | Criticality |
|-----------|------|---------------|-------|---------|-----------|-------------|
| **T7: Keywords** | Extraction | Query → Keywords | LLM | 1-3s | Local/Global | HIGH |
| **T8: Resolution** | Extraction | Keywords → Entities | Vector DB | 50-150ms | Local/Global | CRITICAL |
| **T9: Assembly** | Aggregation | Entities → Context | Graph | 100-500ms | All | HIGH |
| **T10: Generation** | Synthesis | Context → Answer | LLM | 2-8s | All | CRITICAL |
| **T11: Formatting** | Format | Answer → Structured | Parser | 10-50ms | All | MEDIUM |

---

**Next**: [03-transform-chains.md](03-transform-chains.md) - Цепочки преобразований
