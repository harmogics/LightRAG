# Indexing Transformations: Преобразования при Индексации

## Обзор

Indexing Pipeline выполняет 6 основных семантических преобразований для конвертации неструктурированных документов в структурированный Knowledge Graph.

```
Document → [T1] → Chunks → [T2] → Raw Entities → [T3] → Refined Entities
                                                                ↓
                                                              [T4]
                                                                ↓
                                                    Structured Entities
                                                                ↓
                                                              [T5]
                                                                ↓
                                                      Merged Entities
                                                                ↓
                                                              [T6]
                                                                ↓
                                                    Knowledge Graph + Vectors
```

## T1: Chunking (Text Segmentation)

### Semantic Operation

**Type**: DECOMPOSITION (1 → Many)

**Semantic Change**: Document → Semantically coherent fragments

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Single document (text string) |
| **Output** | List of text chunks with metadata |
| **Input Size** | 1K-1M tokens |
| **Output Count** | 1-1000+ chunks |

### Transformation Details

```python
Input:
{
    "doc_id": "doc-abc123",
    "content": "Apple Inc is a technology company headquartered in Cupertino...",
    "file_path": "/docs/apple.pdf"
}

Transform: chunking_by_token_size()
Parameters:
- max_token_size: 1024
- overlap_token_size: 128
- split_by_character: "\n\n" (optional)

Output:
[
    {
        "chunk_id": "chunk-xyz789",
        "content": "Apple Inc is a technology company...",
        "tokens": 512,
        "chunk_order_index": 0,
        "full_doc_id": "doc-abc123"
    },
    {
        "chunk_id": "chunk-def456",
        "content": "...headquartered in Cupertino. The company...",
        "tokens": 498,
        "chunk_order_index": 1,
        "full_doc_id": "doc-abc123"
    }
]
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Preservation** | 98-99% | Minimal loss (only boundary effects) |
| **Semantic Coherence** | High | Maintains context within chunks |
| **Reversibility** | Yes | Can reconstruct document |
| **Deterministic** | Yes | Same input → same output |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Rule-Based Engine** | Primary | Executes segmentation |
| **Tokenizer** | Support | Counts tokens |
| **LLM** | None | Not involved |

### Information Flow

```
Information Type: Structural + Semantic

Input Information:
- Document text: 100%
- Document structure: 100%
- Metadata: 100%

Output Information:
- Text content: 100% (preserved)
- Chunk boundaries: New (added)
- Positional info: New (added)
- Overlap regions: Redundant (for context)

Net Change: +2% (metadata overhead)
```

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 1-5ms per document |
| **Throughput** | 1000+ docs/sec |
| **Memory** | O(N) where N = doc size |
| **Scalability** | Linear |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Boundary Quality** | >90% | 92-95% |
| **Context Preservation** | >95% | 97-99% |
| **Overlap Effectiveness** | >80% | 85-90% |

### Role in Chain

**Position**: First transformation

**Criticality**: HIGH - affects all downstream transformations

**Dependencies**: None

**Dependents**: T2 (Entity Extraction) - directly uses chunks

**Optimization Impact**:
- Good chunking → Better entity extraction
- Poor chunking → Entity fragmentation

---

## T2: Entity Extraction (Semantic Parsing)

### Semantic Operation

**Type**: EXTRACTION (Unstructured → Structured)

**Semantic Change**: Raw text → Structured entities + relationships

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Text chunk (unstructured) |
| **Output** | Entities + Relationships (structured) |
| **Input Size** | 512-1024 tokens |
| **Output Count** | 5-20 entities, 3-15 relationships |

### Transformation Details

```python
Input:
{
    "chunk_id": "chunk-xyz789",
    "content": "Tim Cook is the CEO of Apple Inc, which is headquartered in Cupertino, California."
}

Transform: extract_entities() with LLM
Prompt: entity_extraction_system_prompt
Model: GPT-4 or similar

Output:
{
    "entities": {
        "Tim Cook": [{
            "entity_name": "Tim Cook",
            "entity_type": "person",
            "description": "Tim Cook is the CEO of Apple Inc",
            "source_id": "chunk-xyz789"
        }],
        "Apple Inc": [{
            "entity_name": "Apple Inc",
            "entity_type": "organization",
            "description": "Apple Inc is a technology company",
            "source_id": "chunk-xyz789"
        }],
        "Cupertino": [{
            "entity_name": "Cupertino",
            "entity_type": "location",
            "description": "Cupertino is in California",
            "source_id": "chunk-xyz789"
        }]
    },
    "relationships": {
        ("Tim Cook", "Apple Inc"): [{
            "source_entity": "Tim Cook",
            "target_entity": "Apple Inc",
            "keywords": "CEO, leadership",
            "description": "Tim Cook serves as CEO of Apple Inc",
            "source_id": "chunk-xyz789"
        }],
        ("Apple Inc", "Cupertino"): [{
            "source_entity": "Apple Inc",
            "target_entity": "Cupertino",
            "keywords": "headquarters, location",
            "description": "Apple Inc is headquartered in Cupertino",
            "source_id": "chunk-xyz789"
        }]
    }
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Preservation** | 50-70% | Extracts key facts, loses details |
| **Information Enrichment** | +30% | Adds structure, types, relationships |
| **Semantic Abstraction** | Medium | From text to entity concepts |
| **Reversibility** | No | Cannot reconstruct original text |
| **Deterministic** | No | LLM may vary |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Entity Extraction Agent** | Primary | Identifies entities/relations |
| **LLM (GPT-4)** | Executor | Performs extraction |
| **Parser** | Support | Structures output |

### Information Flow

```
Information Type: Semantic Extraction

Input Information:
- Raw text: 100%
- Implicit entities: ~60%
- Implicit relations: ~40%

Extracted Information:
- Explicit entities: 80-90% of implicit
- Explicit relations: 70-80% of implicit
- Entity types: NEW (classification)
- Descriptions: NEW (synthesis)

Net Change:
- Factual content: 60% preserved
- Structure: +40% added
- Overall: 100% → 100% (different form)
```

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 2-5s per chunk |
| **Throughput** | 0.2-0.5 chunks/sec |
| **LLM Calls** | 1 per chunk |
| **Token Usage** | ~2500 tokens |
| **Scalability** | Parallel (per chunk) |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Entity Precision** | >85% | 88-92% |
| **Entity Recall** | >75% | 78-85% |
| **Relation Precision** | >80% | 82-88% |
| **Relation Recall** | >70% | 72-80% |
| **Type Accuracy** | >90% | 92-96% |

### Role in Chain

**Position**: Second transformation (after chunking)

**Criticality**: CRITICAL - core knowledge extraction

**Dependencies**: T1 (Chunking) - requires well-formed chunks

**Dependents**:
- T3 (Gleaning) - refines this output
- T5 (Merging) - aggregates across chunks

**Optimization Impact**:
- Better prompts → Higher precision/recall
- Larger context → More relationships detected

---

## T3: Gleaning (Refinement)

### Semantic Operation

**Type**: EXTRACTION (Refinement Pass)

**Semantic Change**: Initial extraction → Refined + Complete extraction

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Initial extraction results + Original text |
| **Output** | Additional/corrected entities + relationships |
| **Output Type** | Delta (additions/corrections) |

### Transformation Details

```python
Input:
{
    "initial_extraction": {
        "entities": ["Tim Cook", "Apple Inc"],
        "relationships": [("Tim Cook", "Apple Inc")]
    },
    "original_text": "Tim Cook is the CEO of Apple Inc, which is headquartered in Cupertino, California. He succeeded Steve Jobs in 2011."
}

Transform: extract_entities() with history
Prompt: entity_continue_extraction_user_prompt
History: previous extraction

Output (Delta):
{
    "new_entities": {
        "Steve Jobs": [{
            "entity_name": "Steve Jobs",
            "entity_type": "person",
            "description": "Steve Jobs was the previous CEO",
            "source_id": "chunk-xyz789"
        }]
    },
    "refined_entities": {
        "Tim Cook": [{
            "entity_name": "Tim Cook",
            "entity_type": "person",
            "description": "Tim Cook is the CEO of Apple Inc since 2011, succeeding Steve Jobs",
            "source_id": "chunk-xyz789"
        }]
    },
    "new_relationships": {
        ("Tim Cook", "Steve Jobs"): [{
            "source_entity": "Tim Cook",
            "target_entity": "Steve Jobs",
            "keywords": "succession, predecessor",
            "description": "Tim Cook succeeded Steve Jobs as CEO",
            "source_id": "chunk-xyz789"
        }]
    }
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Gain** | +10-20% | Captures missed entities |
| **Refinement Quality** | High | Improves descriptions |
| **Completeness** | +15% | Increases coverage |
| **Deterministic** | No | LLM-based |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Gleaning Agent** | Primary | Finds missed items |
| **LLM (GPT-4)** | Executor | Performs gleaning |
| **Comparator** | Support | Compares extractions |

### Information Flow

```
Information Type: Iterative Refinement

Input Information:
- Initial extraction: 75-80% coverage
- Original text: 100% reference

Gleaning Process:
- Identifies gaps: 15-20%
- Refines descriptions: 10-15%
- Corrects errors: 5-10%

Output Information:
- Total coverage: 85-95%
- Description quality: +20%

Net Change: +15% information capture
```

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 2-5s per chunk |
| **LLM Calls** | 1 per chunk (conditional) |
| **Token Usage** | ~3500 tokens (with history) |
| **Activation Rate** | 90% (if enabled) |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Additional Entities** | 10-20% | 12-18% |
| **Description Improvement** | >20% | 22-28% |
| **Error Correction** | >80% | 85-92% |
| **False Positives** | <5% | 2-4% |

### Role in Chain

**Position**: Third transformation (after initial extraction)

**Criticality**: MEDIUM - improves quality but optional

**Dependencies**: T2 (Entity Extraction) - refines its output

**Dependents**: T4 (Parsing) - processes gleaning results

**Optimization Impact**:
- Increases entity coverage by 15%
- Reduces downstream merge conflicts

---

## T4: Parsing (Structuring)

### Semantic Operation

**Type**: TRANSFORMATION (Format Conversion)

**Semantic Change**: Raw LLM output → Structured data objects

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Raw LLM text output |
| **Output** | Validated Python dictionaries |
| **Format Change** | Text → JSON-compatible dicts |

### Transformation Details

```python
Input (Raw LLM Output):
"""
entity<|#|>Tim Cook<|#|>person<|#|>Tim Cook is the CEO of Apple Inc
entity<|#|>Apple Inc<|#|>organization<|#|>Apple Inc is a technology company
relation<|#|>Tim Cook<|#|>Apple Inc<|#|>leadership, CEO<|#|>Tim Cook serves as CEO
<|COMPLETE|>
"""

Transform: _process_extraction_result()
Operations:
1. Split by newlines
2. Parse delimiter-separated fields
3. Validate field counts
4. Sanitize text
5. Normalize names
6. Create structured objects

Output:
{
    "entities": {
        "Tim Cook": [{
            "entity_name": "Tim Cook",
            "entity_type": "person",
            "description": "Tim Cook is the CEO of Apple Inc",
            "source_id": "chunk-xyz789",
            "created_at": 1705012345
        }],
        "Apple Inc": [{
            "entity_name": "Apple Inc",
            "entity_type": "organization",
            "description": "Apple Inc is a technology company",
            "source_id": "chunk-xyz789",
            "created_at": 1705012345
        }]
    },
    "relationships": {
        ("Apple Inc", "Tim Cook"): [{  # Sorted for undirected
            "source_entity": "Tim Cook",
            "target_entity": "Apple Inc",
            "keywords": "leadership, CEO",
            "description": "Tim Cook serves as CEO",
            "source_id": "chunk-xyz789",
            "created_at": 1705012345
        }]
    }
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Preservation** | 95-100% | Format change only |
| **Structure Addition** | High | Adds data structure |
| **Validation** | High | Ensures correctness |
| **Deterministic** | Yes | Same input → same output |
| **Reversibility** | Partial | Can regenerate text |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Parser** | Primary | Executes parsing |
| **Validator** | Support | Validates structure |
| **Sanitizer** | Support | Cleans text |
| **LLM** | None | Not involved |

### Information Flow

```
Information Type: Structural Transformation

Input Information:
- Raw text: 100%
- Delimited fields: Implicit structure

Parsing Process:
- Extract fields: 100%
- Validate format: Filter invalid (~5%)
- Normalize names: Standardize
- Add metadata: Timestamps, IDs

Output Information:
- Structured data: 95% (valid records)
- Type safety: NEW
- Query-ready format: NEW

Net Change: Structure +100%, Content 95%
```

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 1-10ms per chunk |
| **Throughput** | 100+ chunks/sec |
| **Memory** | O(N) |
| **CPU** | Low |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Parse Success Rate** | >95% | 96-98% |
| **Validation Pass Rate** | >90% | 92-96% |
| **Data Integrity** | >99% | 99.5% |
| **Format Errors** | <5% | 2-4% |

### Role in Chain

**Position**: Fourth transformation

**Criticality**: HIGH - ensures data quality

**Dependencies**: T2, T3 - parses their outputs

**Dependents**: T5 (Merging) - requires structured input

**Optimization Impact**:
- Robust parsing → Fewer downstream errors
- Good validation → Higher data quality

---

## T5: Description Merging (Summarization)

### Semantic Operation

**Type**: AGGREGATION (Many → 1)

**Semantic Change**: Multiple descriptions → Single cohesive description

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | List of entity/relation descriptions |
| **Output** | Single merged description |
| **Input Count** | 2-50+ descriptions |
| **Output Count** | 1 description |

### Transformation Details

```python
Input:
{
    "entity_name": "Apple Inc",
    "descriptions": [
        "Apple Inc is a technology company",
        "Apple Inc designs and manufactures consumer electronics",
        "Apple Inc is headquartered in Cupertino",
        "Apple Inc was founded in 1976 by Steve Jobs"
    ]
}

Transform: _handle_entity_relation_summary()
Strategy: Map-Reduce (if large list)
Model: GPT-4
Prompt: summarize_entity_descriptions

Output:
{
    "entity_name": "Apple Inc",
    "merged_description": "Apple Inc is a multinational technology company that designs, develops, and manufactures consumer electronics, computer software, and online services. Founded in 1976 by Steve Jobs, Steve Wozniak, and Ronald Wayne, the company is headquartered in Cupertino, California."
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Preservation** | 80-90% | Key facts preserved |
| **Information Loss** | 10-20% | Redundancy removed |
| **Coherence** | High | Creates narrative flow |
| **Completeness** | High | Combines all sources |
| **Deterministic** | No | LLM-based |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Summary Agent** | Primary | Performs merging |
| **LLM (GPT-4)** | Executor | Generates summary |
| **Map-Reduce Engine** | Support | Handles large lists |

### Information Flow

```
Information Type: Aggregation + Synthesis

Input Information:
- Description 1: 100% info
- Description 2: 80% new, 20% overlap
- Description 3: 70% new, 30% overlap
- Description 4: 60% new, 40% overlap
Total unique info: ~250%

Merging Process:
- Identify overlap: 30-40%
- Remove redundancy
- Resolve conflicts
- Synthesize narrative

Output Information:
- Merged description: 90% of unique info
- Coherent narrative: NEW
- Factual completeness: 85-90%

Net Change: 250% → 100% (deduplication)
```

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 3-8s per entity |
| **LLM Calls** | 1 (simple) or log(N) (map-reduce) |
| **Token Usage** | 500-2000 tokens |
| **Activation Rate** | 40-60% entities |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Factual Accuracy** | >90% | 92-96% |
| **Completeness** | >85% | 87-93% |
| **Coherence Score** | >80% | 82-88% |
| **Hallucination Rate** | <5% | 2-4% |
| **Compression Ratio** | 3-5x | 3.5-4.5x |

### Role in Chain

**Position**: Fifth transformation

**Criticality**: HIGH - ensures consistency

**Dependencies**: T4 (Parsing) - needs structured input

**Dependents**: T6 (Vectorization) - uses merged descriptions

**Optimization Impact**:
- Good merging → Consistent knowledge
- Poor merging → Information loss or conflicts

---

## T6: Vectorization (Embedding)

### Semantic Operation

**Type**: TRANSFORMATION (Semantic Encoding)

**Semantic Change**: Text → Dense vector representation

### Input/Output

| Aspect | Details |
|--------|---------|
| **Input** | Text (entity/chunk/relation) |
| **Output** | Dense vector (typically 1536 dimensions) |
| **Format** | String → float[] |

### Transformation Details

```python
Input:
{
    "entity_name": "Apple Inc",
    "description": "Apple Inc is a multinational technology company that designs, develops, and manufactures consumer electronics..."
}

Transform: embedding_func()
Model: text-embedding-ada-002 or similar
Content: f"{entity_name}\n{description}"

Output:
{
    "entity_id": "ent-abc123",
    "entity_name": "Apple Inc",
    "description": "...",
    "embedding": [
        0.0234, -0.0156, 0.0421, ...,  # 1536 dimensions
        0.0089, -0.0312, 0.0167
    ]
}
```

### Semantic Characteristics

| Characteristic | Value | Description |
|---------------|-------|-------------|
| **Information Preservation** | ~100% | Semantic meaning preserved |
| **Semantic Similarity** | Preserved | Similar texts → similar vectors |
| **Dimensionality** | Fixed (1536) | Independent of text length |
| **Reversibility** | No | Cannot decode to text |
| **Deterministic** | Yes | Same text → same vector |

### Agent Involvement

| Agent | Role | Contribution |
|-------|------|-------------|
| **Embedding Model** | Primary | Generates vectors |
| **Vector DB** | Storage | Stores embeddings |
| **LLM** | None | Separate embedding model |

### Information Flow

```
Information Type: Semantic Encoding

Input Information:
- Text content: 100%
- Semantic meaning: 100%
- Syntactic structure: 100%

Encoding Process:
- Extract semantic features
- Project to fixed dimensions
- Normalize vector

Output Information:
- Semantic representation: ~100%
- Syntactic details: Lost
- Lexical details: Lost
- Similarity relationships: Preserved

Net Change: Form change, meaning preserved
```

### Performance Metrics

| Metric | Value |
|--------|-------|
| **Latency** | 10-50ms per text |
| **Throughput** | 1000+ texts/sec (batch) |
| **Vector Size** | 6KB (1536 × float32) |
| **API Calls** | 1 per batch (up to 100 texts) |

### Quality Metrics

| Metric | Target | Typical |
|--------|--------|---------|
| **Semantic Fidelity** | >95% | 96-98% |
| **Similarity Preservation** | >90% | 92-96% |
| **Retrieval Precision** | >85% | 87-93% |
| **Cosine Similarity** | Context-dependent | 0.7-0.9 for similar |

### Role in Chain

**Position**: Sixth transformation (final)

**Criticality**: HIGH - enables semantic search

**Dependencies**: T5 (Merging) - uses merged descriptions

**Dependents**: Query pipeline - uses vectors for search

**Optimization Impact**:
- Good embeddings → Better retrieval
- Poor embeddings → Irrelevant results

---

## Summary Comparison Table

| Transform | Type | Input → Output | Agent | Latency | Info Preservation | Criticality |
|-----------|------|---------------|-------|---------|------------------|-------------|
| **T1: Chunking** | Decomposition | Doc → Chunks | Rule-based | 1-5ms | 98-99% | HIGH |
| **T2: Extraction** | Extraction | Text → Entities | LLM | 2-5s | 50-70% | CRITICAL |
| **T3: Gleaning** | Refinement | Entities → Refined | LLM | 2-5s | +15% | MEDIUM |
| **T4: Parsing** | Format | Text → Struct | Rule-based | 1-10ms | 95-100% | HIGH |
| **T5: Merging** | Aggregation | Many → One | LLM | 3-8s | 80-90% | HIGH |
| **T6: Vectorization** | Encoding | Text → Vector | Embedding | 10-50ms | ~100% | HIGH |

---

**Next**: [02-query-transforms.md](02-query-transforms.md) - Преобразования при обработке запросов
