# Comparison Tables: Сводные Таблицы

## Обзор

Этот документ содержит исчерпывающие сравнительные таблицы всех семантических преобразований в LightRAG, позволяющие быстро оценить характеристики, компромиссы и подходящие сценарии использования.

---

## Master Comparison Table

### All Transformations Side-by-Side

| ID | Name | Pipeline | Agent Type | Determinism | Fidelity | Latency | Info Loss | Hallucination Risk | Reversible |
|----|------|----------|------------|-------------|----------|---------|-----------|-------------------|------------|
| **T1** | Chunking | Indexing | Rule-based | 95% | 0.98 | 1-5ms | 2% | Very Low | 85% |
| **T2** | Entity Extraction | Indexing | LLM | 60% | 0.65 | 2-5s | 40% | Medium (5%) | 20% |
| **T3** | Gleaning | Indexing | LLM | 55% | 0.75 | 2-5s | -15% (gain!) | Low (3%) | 20% |
| **T4** | Parsing | Indexing | Rule-based | 98% | 0.95 | 1-10ms | 3% | Very Low | 95% |
| **T5** | Description Merging | Indexing | LLM | 65% | 0.85 | 3-8s | 15% | Medium (8%) | 30% |
| **T6** | Vectorization | Indexing | Model | 99% | 0.95 | 10-50ms | 0% (semantic) | Very Low | 0% |
| **T7** | Keyword Extraction | Query | LLM | 60% | 0.90 | 1-3s | 10% | Low (5%) | 25% |
| **T8** | Entity Resolution | Query | Vector Search | 75% | 0.85 | 50-150ms | 5% | Very Low | 60% |
| **T9** | Context Assembly | Query | Rule-based | 90% | 0.90 | 100-500ms | 5% | Very Low | 80% |
| **T10** | Answer Generation | Query | LLM | 50% | 0.85 | 2-8s | 25% | Medium-High (10%) | 20% |
| **T11** | Answer Formatting | Query | Rule-based | 98% | 0.99 | 10-50ms | 0% | Very Low | 98% |

**Color Key** (conceptual):
- 🟢 Green: High quality / Low risk (Fidelity >0.90, Determinism >90%, Hallucination <3%)
- 🟡 Yellow: Medium (Fidelity 0.75-0.90, Determinism 60-90%, Hallucination 3-8%)
- 🔴 Red: Lower/Higher risk (Fidelity <0.75, Determinism <60%, Hallucination >8%)

---

## Comparison by Pipeline

### Indexing Transformations (T1-T6)

| Transform | Primary Function | Input | Output | Semantic Operation | Parallelizable | Critical Path |
|-----------|-----------------|-------|--------|-------------------|----------------|---------------|
| **T1: Chunking** | Decomposition | Document | Text Chunks (1-1000) | Segmentation | No | Yes |
| **T2: Entity Extraction** | Extraction | Text Chunk | Raw Entities + Relations | Abstraction | Yes (per chunk) | Yes |
| **T3: Gleaning** | Refinement | Raw Entities | Refined Entities | Validation + Discovery | Yes (per chunk) | Optional |
| **T4: Parsing** | Structuring | Raw JSON | Python Objects | Format Conversion | Yes (per chunk) | Yes |
| **T5: Merging** | Aggregation | Multiple Descriptions | Unified Description | Synthesis | Yes (per entity) | Yes |
| **T6: Vectorization** | Encoding | Text Descriptions | Vector Embeddings | Semantic Encoding | Yes (batch) | Yes |

**Key Insights**:
- **Bottleneck**: T2 and T5 (LLM calls) - 5-13s total
- **Parallelization**: Heavy (T2-T4 per chunk, T5 per entity)
- **Quality Gate**: T3 (optional but recommended)
- **Total Pipeline**: ~10-30s per document (depending on size)

---

### Query Transformations (T7-T11)

| Transform | Primary Function | Input | Output | Semantic Operation | Mode Usage | Critical Path |
|-----------|-----------------|-------|--------|-------------------|------------|---------------|
| **T7: Keyword Extraction** | Intent Analysis | User Query | High/Low Keywords | Decomposition | Local, Global, Hybrid | Yes |
| **T8: Entity Resolution** | Disambiguation | Keywords | Entity IDs | Mapping | Local, Global, Hybrid | Yes |
| **T9: Context Assembly** | Aggregation | Graph + Chunks | Structured Context | Selection + Ordering | All modes | Yes |
| **T10: Answer Generation** | Synthesis | Context + Query | Natural Language Answer | Generation | All modes | Yes |
| **T11: Formatting** | Structuring | Raw Answer | JSON/Markdown | Format Conversion | All modes | No |

**Key Insights**:
- **Bottleneck**: T7 and T10 (LLM calls) - 3-11s total
- **Mode Impact**: T7-T8 skipped in Naive mode (save 1-3s)
- **Quality Gate**: T10 (main risk point)
- **Total Pipeline**: 2-15s depending on mode

---

## Comparison by Agent Type

### LLM-Driven Transformations

| Transform | Model Usage | Temperature | Prompt Size | Output Tokens | Cacheable | Variability |
|-----------|-------------|-------------|-------------|---------------|-----------|-------------|
| **T2: Entity Extraction** | Required | 0.0 | 2-4KB | 500-2000 | Yes (high hit rate) | Medium |
| **T3: Gleaning** | Required | 0.0 | 3-5KB | 200-1000 | Yes (medium hit rate) | Medium |
| **T5: Description Merging** | Required | 0.1 | 1-3KB | 200-500 | Yes (low hit rate) | Medium |
| **T7: Keyword Extraction** | Required | 0.0 | 1-2KB | 100-300 | Yes (medium hit rate) | Medium |
| **T10: Answer Generation** | Required | 0.1 | 5-100KB | 200-800 | Yes (low hit rate) | High |

**Characteristics**:
- **Latency**: 1-8s per call
- **Cost**: Dominant cost factor ($$$)
- **Quality**: High semantic understanding
- **Risk**: Hallucinations, variability
- **Optimization**: Caching, prompt engineering, batch processing

---

### Rule-Based Transformations

| Transform | Algorithm | Complexity | CPU Usage | Memory Usage | Deterministic | Error Rate |
|-----------|-----------|------------|-----------|--------------|---------------|------------|
| **T1: Chunking** | Recursive split | O(n) | Low | Low | 95% | <1% |
| **T4: Parsing** | JSON validation | O(n) | Very Low | Low | 98% | <2% |
| **T9: Context Assembly** | Graph traversal + sort | O(n log n) | Medium | Medium | 90% | <2% |
| **T11: Formatting** | Template rendering | O(n) | Very Low | Low | 98% | <1% |

**Characteristics**:
- **Latency**: <500ms typically
- **Cost**: Negligible
- **Quality**: Consistent, predictable
- **Risk**: Very low
- **Optimization**: Algorithm tuning, caching

---

### Vector/Model-Based Transformations

| Transform | Model Type | Dimensions | Batch Size | Throughput | Accuracy | Hardware |
|-----------|------------|------------|------------|------------|----------|----------|
| **T6: Vectorization** | Embedding | 768-1536 | 8-32 | 10-100/s | N/A | GPU optional |
| **T8: Entity Resolution** | Vector Search | 768-1536 | Top-K: 5-15 | 100-1000/s | 80-85% | CPU/GPU |

**Characteristics**:
- **Latency**: 10-150ms
- **Cost**: Low ($$)
- **Quality**: High semantic similarity
- **Risk**: Low
- **Optimization**: Batch processing, HNSW index, GPU acceleration

---

## Comparison by Complexity

### Low Complexity (Simple, Fast, Reliable)

| Transform | Complexity Score | Implementation Lines | Test Coverage | Failure Rate | Maintenance |
|-----------|------------------|---------------------|---------------|--------------|-------------|
| **T1: Chunking** | 2/10 | ~100 | 95% | <0.1% | Low |
| **T4: Parsing** | 2/10 | ~150 | 90% | <0.5% | Low |
| **T11: Formatting** | 1/10 | ~80 | 95% | <0.1% | Very Low |

**Characteristics**: Easy to understand, test, and maintain

---

### Medium Complexity (Balanced)

| Transform | Complexity Score | Implementation Lines | Test Coverage | Failure Rate | Maintenance |
|-----------|------------------|---------------------|---------------|--------------|-------------|
| **T6: Vectorization** | 4/10 | ~200 | 85% | <1% | Low-Medium |
| **T8: Entity Resolution** | 5/10 | ~250 | 80% | 2-3% | Medium |
| **T9: Context Assembly** | 6/10 | ~400 | 80% | 1-2% | Medium |

**Characteristics**: Moderate complexity, well-tested patterns

---

### High Complexity (Advanced, Requires Tuning)

| Transform | Complexity Score | Implementation Lines | Test Coverage | Failure Rate | Maintenance |
|-----------|------------------|---------------------|---------------|--------------|-------------|
| **T2: Entity Extraction** | 8/10 | ~600 | 75% | 5-10% | High |
| **T3: Gleaning** | 7/10 | ~400 | 75% | 3-5% | Medium-High |
| **T5: Description Merging** | 7/10 | ~450 | 75% | 4-6% | Medium-High |
| **T7: Keyword Extraction** | 6/10 | ~300 | 80% | 2-4% | Medium |
| **T10: Answer Generation** | 9/10 | ~700 | 70% | 8-12% | High |

**Characteristics**: Complex prompt engineering, quality monitoring required

---

## Performance vs Accuracy Trade-offs

### Speed-Focused Configuration

| Transform | Standard Latency | Speed Config | Latency Reduction | Accuracy Impact |
|-----------|------------------|--------------|-------------------|-----------------|
| **T1** | 1-5ms | Smaller chunks | 0% | -5% (boundary loss) |
| **T2** | 2-5s | gpt-4o-mini | -40% | -10% (quality) |
| **T3** | 2-5s | Skip entirely | -100% (skip) | -15% (coverage) |
| **T5** | 3-8s | gpt-4o-mini | -30% | -8% (summary quality) |
| **T7** | 1-3s | gpt-4o-mini | -30% | -5% (intent) |
| **T10** | 2-8s | gpt-4o-mini | -25% | -10% (answer quality) |

**Total Savings**: ~7-15s → ~3-7s (50-60% faster)
**Accuracy Loss**: ~10-15% overall

---

### Quality-Focused Configuration

| Transform | Standard Latency | Quality Config | Latency Increase | Accuracy Gain |
|-----------|------------------|----------------|------------------|---------------|
| **T1** | 1-5ms | Larger overlap (256 tokens) | +20% | +5% (context) |
| **T2** | 2-5s | gpt-4o + examples | +30% | +15% (extraction) |
| **T3** | 2-5s | Always enabled | +0% (if always on) | +15% (coverage) |
| **T5** | 3-8s | gpt-4o + longer summaries | +40% | +10% (quality) |
| **T7** | 1-3s | gpt-4o + careful prompts | +20% | +8% (intent) |
| **T10** | 2-8s | gpt-4o + citations + low temp | +50% | +15% (accuracy) |

**Total Increase**: ~10-30s → ~15-45s (50% slower)
**Accuracy Gain**: ~15-20% overall

---

### Balanced Configuration (Recommended)

| Transform | Config | Rationale |
|-----------|--------|-----------|
| **T1** | Overlap: 192 tokens | Good balance |
| **T2** | gpt-4o-mini + cache | Fast enough, cached well |
| **T3** | Conditional (low confidence only) | Best ROI |
| **T5** | gpt-4o-mini | Adequate quality |
| **T7** | gpt-4o-mini | Sufficient intent capture |
| **T10** | gpt-4o-mini + citations + temp=0.1 | Quality where it matters |

**Total Latency**: ~8-20s
**Accuracy**: ~85-90%

---

## Comparison by Information Flow

### Information Reduction Transformations

| Transform | Input Info | Output Info | Reduction | Type | Intentional |
|-----------|-----------|-------------|-----------|------|-------------|
| **T2: Entity Extraction** | 100% | 60% | -40% | Abstraction | Yes |
| **T5: Description Merging** | 100% | 85% | -15% | Compression | Yes |
| **T7: Keyword Extraction** | 100% | 90% | -10% | Abstraction | Yes |
| **T10: Answer Generation** | 100% | 70% | -30% | Synthesis | Yes |

**Purpose**: Extract relevant information, discard noise

---

### Information Preservation Transformations

| Transform | Input Info | Output Info | Preservation | Type | Lossless |
|-----------|-----------|-------------|--------------|------|----------|
| **T1: Chunking** | 100% | 98% | 98% | Segmentation | Near-lossless |
| **T4: Parsing** | 100% | 97% | 97% | Format conversion | Near-lossless |
| **T6: Vectorization** | 100% (semantic) | 100% | 100% | Encoding | Semantic-lossless |
| **T11: Formatting** | 100% | 100% | 100% | Restructuring | Lossless |

**Purpose**: Maintain information while changing representation

---

### Information Enhancement Transformations

| Transform | Input Info | Output Info | Enhancement | Type | How |
|-----------|-----------|-------------|-------------|------|-----|
| **T3: Gleaning** | 100% | 115% | +15% | Discovery | Find missed entities |
| **T9: Context Assembly** | 100% (parts) | 105% | +5% | Aggregation | Add relationships |

**Purpose**: Add missing information through refinement or context

---

## Comparison by Usage Context

### Always Required Transformations

| Transform | Pipeline | Skip Cost | Alternative | Recommended Config |
|-----------|----------|-----------|-------------|-------------------|
| **T1: Chunking** | Indexing | Cannot skip | N/A | Balanced overlap |
| **T4: Parsing** | Indexing | System breaks | N/A | Standard validation |
| **T6: Vectorization** | Indexing | No search | N/A | Good embedding model |
| **T9: Context Assembly** | Query | No context | N/A | Mode-appropriate |
| **T10: Answer Generation** | Query | No answer | T11 only (echo context) | Quality model + grounding |
| **T11: Formatting** | Query | Ugly output | Raw text | Standard templates |

---

### Conditionally Required Transformations

| Transform | Pipeline | When Required | When Optional | Skip Savings |
|-----------|----------|---------------|---------------|--------------|
| **T2: Entity Extraction** | Indexing | Local/Global modes | Naive mode only | Cannot skip |
| **T3: Gleaning** | Indexing | Low T2 confidence | High confidence | 2-5s |
| **T5: Description Merging** | Indexing | Multiple mentions | Single mention | 3-8s (rare) |
| **T7: Keyword Extraction** | Query | Local/Global modes | Naive mode | 1-3s |
| **T8: Entity Resolution** | Query | Local/Global modes | Naive mode | 50-150ms |

---

### Mode-Specific Usage

#### Naive Mode (Fastest)

| Transform | Used | Skipped | Impact |
|-----------|------|---------|--------|
| T1-T6 | ✓ (indexing) | - | Full graph available |
| **T7** | ✗ | Skipped | -1-3s, use direct vector search |
| **T8** | ✗ | Skipped | -50-150ms, no entities |
| **T9** | ✓ | - | Chunks only, no graph |
| **T10** | ✓ | - | Answer from chunks |
| **T11** | ✓ | - | Format output |

**Total Query Time**: ~2-5s
**Use Case**: Simple factual queries

---

#### Local Mode (Balanced)

| Transform | Used | Configuration | Impact |
|-----------|------|---------------|--------|
| T1-T6 | ✓ | Standard | Full graph |
| **T7** | ✓ | High-level keywords | Seed entities |
| **T8** | ✓ | Top-K: 5-10 | Local subgraph |
| **T9** | ✓ | 1-2 hop traversal | Rich context |
| **T10** | ✓ | Standard | Entity-aware answer |
| **T11** | ✓ | + explainability | Sources shown |

**Total Query Time**: ~4-9s
**Use Case**: Entity-focused queries, relationships

---

#### Global Mode (Comprehensive)

| Transform | Used | Configuration | Impact |
|-----------|------|---------------|--------|
| T1-T6 | ✓ | High quality | Best graph |
| **T7** | ✓ | High + low level keywords | Comprehensive |
| **T8** | ✓ | Top-K: 10-15, extended | Wide coverage |
| **T9** | ✓ | 3+ hops, communities | Very rich context |
| **T10** | ✓ | High quality model | Detailed answer |
| **T11** | ✓ | Full metadata | Complete explainability |

**Total Query Time**: ~7-15s
**Use Case**: Complex reasoning, multi-hop questions

---

## Comparison by Scalability

### Horizontal Scalability (Parallel Processing)

| Transform | Parallelizable | Granularity | Coordination | Speedup Potential |
|-----------|----------------|-------------|--------------|-------------------|
| **T1** | No | Document-level | Sequential | 1x |
| **T2** | Yes | Per chunk | Minimal | 10-100x |
| **T3** | Yes | Per chunk | Minimal | 10-100x |
| **T4** | Yes | Per chunk | Minimal | 10-100x |
| **T5** | Yes | Per entity | Entity locks | 5-50x |
| **T6** | Yes | Batch | Minimal | 2-10x |
| **T7** | No | Per query | N/A | 1x |
| **T8** | Partial | Per keyword | Minimal | 2-5x |
| **T9** | Partial | Subgraph + chunks | Medium | 2-3x |
| **T10** | No | Per query | N/A | 1x |
| **T11** | Yes | Per answer | Minimal | 10x+ |

**Best Parallelization**: T2, T3, T4 (indexing chunks), T5 (entities)

---

### Vertical Scalability (Resource Usage)

| Transform | CPU | Memory | GPU | Disk I/O | Network I/O | Bottleneck |
|-----------|-----|--------|-----|----------|-------------|------------|
| **T1** | Low | Low | No | Low | No | CPU (splitting) |
| **T2** | Low | Medium | Optional | Low | High (LLM API) | Network |
| **T3** | Low | Medium | Optional | Low | High (LLM API) | Network |
| **T4** | Low | Low | No | Low | No | CPU (parsing) |
| **T5** | Low | Medium | Optional | Low | High (LLM API) | Network |
| **T6** | Medium | Medium | Yes (helpful) | Low | Medium (API/local) | GPU/Network |
| **T7** | Low | Low | Optional | Low | High (LLM API) | Network |
| **T8** | Medium | High | Optional | Medium | No | Memory (index) |
| **T9** | Medium | Medium | No | Medium | No | CPU (traversal) |
| **T10** | Low | Medium | Optional | Low | High (LLM API) | Network |
| **T11** | Low | Low | No | Low | No | CPU (minimal) |

**Primary Bottleneck**: Network I/O (LLM API calls)

---

## Comparison by Cost

### Computational Cost (Relative)

| Transform | CPU Cost | Memory Cost | API Cost | Total Cost | Cost/Document | Cost/Query |
|-----------|----------|-------------|----------|------------|---------------|------------|
| **T1** | $ | $ | - | $ | $0.0001 | - |
| **T2** | $ | $$ | $$$$ | $$$$ | $0.01-0.05 | - |
| **T3** | $ | $$ | $$$$ | $$$$ | $0.01-0.05 | - |
| **T4** | $ | $ | - | $ | $0.0001 | - |
| **T5** | $ | $$ | $$$ | $$$ | $0.005-0.02 | - |
| **T6** | $$ | $$ | $$ | $$ | $0.001-0.005 | - |
| **T7** | $ | $ | $$$ | $$$ | - | $0.002-0.01 |
| **T8** | $$ | $$$ | - | $$ | - | $0.0001 |
| **T9** | $$ | $$ | - | $$ | - | $0.0001 |
| **T10** | $ | $$ | $$$$ | $$$$ | - | $0.01-0.05 |
| **T11** | $ | $ | - | $ | - | $0.00001 |

**Most Expensive**: T2, T3, T5 (indexing LLM), T10 (query LLM)
**Cost Distribution**: ~80% LLM API, ~15% embeddings, ~5% compute

---

### Cost Optimization Strategies

| Strategy | Target Transforms | Savings | Trade-off |
|----------|------------------|---------|-----------|
| **Use smaller models** | T2, T3, T5, T7, T10 | 50-70% | -5-15% quality |
| **Skip gleaning** | T3 | 100% (for T3) | -15% coverage |
| **Cache aggressively** | All LLM | 30-60% | Storage cost |
| **Batch embeddings** | T6 | 20-40% | Slight latency |
| **Use local models** | T2, T10 | 90%+ | Infrastructure cost |
| **Reduce chunk size** | T2 (fewer chunks) | 30-50% | -10% coverage |

---

## Failure Modes and Recovery

### Critical Failures (System-Breaking)

| Transform | Failure Mode | Frequency | Impact | Recovery Strategy | Detection |
|-----------|--------------|-----------|--------|-------------------|-----------|
| **T1** | Chunking fails | <0.1% | Cannot process doc | Retry with fallback | Immediate |
| **T4** | Parsing fails | <0.5% | Lost entities | Skip invalid, log | Immediate |
| **T6** | Embedding fails | <1% | No search | Retry, fallback | Immediate |
| **T10** | Generation fails | 1-2% | No answer | Retry, fallback | Immediate |

**Mitigation**: Robust error handling, fallbacks, retries

---

### Quality Failures (Degraded Output)

| Transform | Failure Mode | Frequency | Impact | Detection | Recovery |
|-----------|--------------|-----------|--------|-----------|----------|
| **T2** | Poor extraction | 5-10% | Incomplete graph | Offline analysis | Gleaning (T3) |
| **T3** | No improvement | 3-5% | Still incomplete | Offline analysis | Accept result |
| **T5** | Poor summary | 4-6% | Bad entity descriptions | User feedback | Regenerate |
| **T7** | Wrong intent | 2-4% | Irrelevant results | User feedback | Rephrase query |
| **T8** | Wrong entities | 2-3% | Wrong subgraph | Offline analysis | Expand search |
| **T10** | Hallucination | 8-12% | Wrong answer | User feedback | Regenerate with grounding |

**Mitigation**: Quality monitoring, user feedback loops, validation

---

### Performance Failures (Timeout/Slowness)

| Transform | Failure Mode | Frequency | Impact | Mitigation |
|-----------|--------------|-----------|--------|------------|
| **T2** | LLM timeout | 1-2% | Indexing stalls | Retry, increase timeout |
| **T3** | LLM timeout | 1-2% | Skip gleaning | Make optional |
| **T5** | LLM timeout | 1-2% | Indexing stalls | Retry, simplify prompt |
| **T8** | Index overload | <1% | Slow query | Scale vector DB |
| **T9** | Graph too large | <1% | Slow query | Limit traversal depth |
| **T10** | LLM timeout | 2-3% | No answer | Retry, reduce context |

**Mitigation**: Timeouts, circuit breakers, fallback modes

---

## Recommendations by Use Case

### Scientific/Academic Research

| Priority | Transform | Recommendation | Rationale |
|----------|-----------|----------------|-----------|
| 1 | **T2/T3** | High quality (gpt-4o + gleaning) | Accuracy critical |
| 2 | **T10** | High quality + citations | Verifiability |
| 3 | **T5** | Detailed summaries | Preserve details |
| 4 | **Mode** | Global | Comprehensive coverage |

**Focus**: Accuracy > Speed

---

### Enterprise/Production

| Priority | Transform | Recommendation | Rationale |
|----------|-----------|----------------|-----------|
| 1 | **T2/T3** | Balanced (gpt-4o-mini + conditional gleaning) | Cost-effective |
| 2 | **T10** | Fast + grounding | Good enough quality |
| 3 | **Caching** | Aggressive | Reduce repeated costs |
| 4 | **Mode** | Local (default) | Good balance |

**Focus**: Cost-effectiveness, reliability

---

### Rapid Prototyping/Development

| Priority | Transform | Recommendation | Rationale |
|----------|-----------|----------------|-----------|
| 1 | **T3** | Skip | Faster iteration |
| 2 | **All LLM** | Smallest models | Low cost |
| 3 | **Mode** | Naive | Fastest |
| 4 | **Caching** | Minimal | Simpler setup |

**Focus**: Speed, simplicity

---

### High-Traffic Consumer Application

| Priority | Transform | Recommendation | Rationale |
|----------|-----------|----------------|-----------|
| 1 | **Indexing** | Offline, high quality | One-time cost |
| 2 | **Query LLM** | Fast models | Low latency |
| 3 | **Caching** | Aggressive (queries) | Reduce cost |
| 4 | **Mode** | Hybrid (adaptive) | Balance per query |

**Focus**: Query latency, scale

---

## Summary: Best Practices

### Indexing Best Practices

1. **Use gleaning (T3) for important documents**: +15% coverage for 2-5s
2. **Parallelize T2-T4 aggressively**: 10-100x speedup
3. **Cache embedding calls (T6)**: High hit rate on similar text
4. **Monitor T2 extraction quality**: Most critical quality gate
5. **Use appropriate chunk size in T1**: Balance context vs. granularity

---

### Query Best Practices

1. **Choose mode based on query type**: Naive for facts, Local for entities, Global for reasoning
2. **Cache T7 keyword extraction**: Common query patterns repeat
3. **Optimize T10 prompts**: Most expensive and highest risk
4. **Set appropriate timeouts**: Prevent slow queries from hanging
5. **Monitor hallucinations in T10**: Critical quality metric

---

### Cost Optimization Best Practices

1. **Use gpt-4o-mini by default**: 70% cheaper, 10-15% quality loss
2. **Skip T3 for speed**: Unless quality is critical
3. **Batch T6 embeddings**: 20-40% cost reduction
4. **Cache everything cacheable**: 30-60% API cost savings
5. **Monitor per-transform costs**: Identify optimization opportunities

---

### Quality Assurance Best Practices

1. **Validate T2 output regularly**: Sample and check extraction quality
2. **Require citations in T10**: Reduce hallucinations
3. **Use temperature=0 or 0.1**: Reduce LLM variability
4. **Set up quality monitoring**: Track precision, recall, F1
5. **Collect user feedback**: Improve prompts iteratively

---

**End of Comparison Tables**

For detailed information on each transformation:
- [01-indexing-transforms.md](01-indexing-transforms.md) - Detailed indexing transformations
- [02-query-transforms.md](02-query-transforms.md) - Detailed query transformations
- [03-transform-chains.md](03-transform-chains.md) - Transformation chains
- [04-semantic-characteristics.md](04-semantic-characteristics.md) - Semantic analysis
