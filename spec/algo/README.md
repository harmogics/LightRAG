# ML Algorithms: Машинное Обучение в LightRAG

## Обзор

Документация всех ML алгоритмов, применяемых в LightRAG для semantic understanding, text generation, и knowledge graph construction.

## ML Pipeline в LightRAG

```
┌────────────────────────────────────────────────────┐
│              INPUT TEXT                            │
└────────────────────┬───────────────────────────────┘
                     │
          ┌──────────┼──────────┐
          ↓                     ↓
   [LLM: Entity Extraction] [Embedding Model]
          │                     │
          ↓                     ↓
   Entities/Relations      Vector Embeddings
          │                     │
          ↓                     ↓
   Graph Construction    Vector Storage
          │                     │
          ↓                     ↓
   [Community Detection] [Similarity Search]
   (Louvain Algorithm)   (Cosine)
          │                     │
          └──────────┬──────────┘
                     ↓
            Query Processing
                     │
          ┌──────────┼──────────┐
          ↓                     ↓
   [Vector Retrieval]    [Graph Traversal]
   (Top-K + Reranking)   (BFS + Centrality)
          │                     │
          └──────────┬──────────┘
                     ↓
         [LLM: Answer Generation]
                     │
                     ↓
                 ANSWER
```

## Категории ML Алгоритмов

### 1. Language Models (Большие Языковые Модели)

**Файл**: [01-language-models.md](01-language-models.md)

Transformer-based autoregressive models для генерации текста:
- GPT-4, GPT-3.5-turbo (OpenAI)
- Claude 3.5 Sonnet, Claude 3 Opus (Anthropic)
- Llama 3, Mistral, Qwen (Open-source)
- Chain of Thought (CoT) reasoning

**Применение**:
- Entity/Relation extraction (NER)
- Text summarization (gleaning)
- Answer generation (RAG response)
- Keyword extraction (high/low level)

**Зависимости**: `openai`, `anthropic`, `transformers` (HuggingFace)

### 2. Embedding Models (Модели Векторизации)

**Файл**: [02-embedding-models.md](02-embedding-models.md)

Transformer encoders для преобразования text → dense vectors:
- text-embedding-3-small/large (OpenAI)
- sentence-transformers (all-mpnet-base-v2, bge-large)
- jina-embeddings-v2 (Jina AI)

**Применение**:
- Entity embeddings (semantic search)
- Relation embeddings (similarity matching)
- Chunk embeddings (document retrieval)
- Query embeddings (search key)

**Зависимости**: `openai`, `sentence-transformers`, `transformers`

### 3. Cross-Encoder Reranking Models

**Файл**: [03-reranking-models.md](03-reranking-models.md)

Cross-attention models для precise relevance scoring:
- Cohere rerank-v3.5 (Cohere API)
- jina-reranker-v2-base-multilingual (Jina AI)

**Применение**:
- Second-stage refinement после vector search
- Chunk reranking (top-100 → top-20)
- Entity/relation reranking

**Зависимости**: `cohere`, `jina-ai` (API calls)

### 4. Community Detection Algorithms

**Файл**: [04-community-detection.md](04-community-detection.md)

Graph ML для unsupervised clustering:
- Louvain algorithm (modularity optimization)

**Применение**:
- Entity clustering (semantic communities)
- Graph visualization (color-coding)
- Global query mode (community-aware context)

**Зависимости**: `python-louvain`, `networkx`

### 5. Prompt-Based Entity Extraction

**Файл**: [05-entity-extraction.md](05-entity-extraction.md)

LLM prompting для structured information extraction:
- Few-shot prompting (in-context learning)
- Chain extraction (iterative gleaning)
- Structured output parsing

**Применение**:
- Named Entity Recognition (NER)
- Relation extraction (RE)
- Entity type classification

**Зависимости**: LLM providers (GPT, Claude, etc.)

---

## Сравнение Архитектур

| Model Type | Architecture | Input | Output | Training |
|------------|-------------|-------|--------|----------|
| **Language Model** | Decoder-only Transformer | Text → | → Text | Autoregressive (next token prediction) |
| **Embedding Model** | Encoder-only Transformer | Text → | → Vector (ℝⁿ) | Contrastive learning (SimCSE, InfoNCE) |
| **Cross-Encoder** | Full Transformer | [Text1, Text2] → | → Score | Classification (relevance labels) |
| **Community Detection** | Graph algorithm | Graph → | → Clusters | Unsupervised (modularity) |

---

## Связь с System Components

| ML Algorithm | Vector Space | Graph Space | Research Concept |
|--------------|--------------|-------------|------------------|
| **Embedding Models** | Generate vectors | - | [Dual-Space Architecture](../research/03-dual-space-architecture.md) |
| **Language Models** | - | Extract entities/relations | [LLM Semantic Processing](../research/06-llm-semantic-processing.md) |
| **Cross-Encoders** | Refine similarity | - | [Semantic Traversal](../research/04-semantic-traversal-methods.md) |
| **Community Detection** | - | Cluster entities | [Star-Attractor Pattern](../research/02-star-attractor-pattern.md) |

---

## Dependency Summary

### Core ML Libraries

| Library | Purpose | Installation |
|---------|---------|--------------|
| **openai** | GPT models + embeddings | `pip install openai` |
| **anthropic** | Claude models | `pip install anthropic` |
| **transformers** | HuggingFace models | `pip install transformers torch` |
| **sentence-transformers** | Embedding models | `pip install sentence-transformers` |
| **python-louvain** | Community detection | `pip install python-louvain` |
| **cohere** | Reranking API | `pip install cohere` |

### Model Selection Guide

**Embedding Models**:
- **API (recommended)**: OpenAI text-embedding-3-small (cost-effective, high quality)
- **Local**: sentence-transformers/all-mpnet-base-v2 (privacy, no cost)
- **Multilingual**: intfloat/e5-large-v2 (100+ languages)

**Language Models**:
- **Best quality**: GPT-4, Claude 3.5 Sonnet
- **Cost-effective**: GPT-3.5-turbo, Claude 3 Haiku
- **Local**: Llama 3 8B, Mistral 7B (HuggingFace)

**Reranking**:
- **Best quality**: Cohere rerank-v3.5
- **Cost-effective**: Jina reranker-v2 (100x cheaper)
- **Local**: cross-encoder/ms-marco-MiniLM-L-12-v2

---

## Performance Characteristics

### Latency Comparison (Single Request)

| Model Type | API | Local (GPU) | Local (CPU) |
|------------|-----|-------------|-------------|
| **Embedding** (32 texts) | 200ms | 80ms | 500ms |
| **LLM Generation** (100 tokens) | 2-5s | 1-3s | 10-30s |
| **Reranking** (20 docs) | 150ms | 50ms | 300ms |
| **Community Detection** (10K nodes) | - | 500ms | 500ms |

### Cost Comparison (per 1M tokens)

| Model | Provider | Cost |
|-------|----------|------|
| **text-embedding-3-small** | OpenAI | $0.02 |
| **text-embedding-3-large** | OpenAI | $0.13 |
| **GPT-3.5-turbo** | OpenAI | $0.50 (input) + $1.50 (output) |
| **GPT-4** | OpenAI | $30 (input) + $60 (output) |
| **Claude 3.5 Sonnet** | Anthropic | $3 (input) + $15 (output) |
| **Cohere rerank-v3.5** | Cohere | $2 per 1K reranks |
| **Jina reranker-v2** | Jina | $0.02 per 1K reranks |

---

## Training Paradigms

### 1. Supervised Fine-Tuning (SFT)

**Used in**: Language models, cross-encoders

```
Training Data: (input, output) pairs
Loss: Cross-entropy (next token prediction)

Example (LLM):
Input: "Extract entities from: Apple Inc produces iPhones"
Output: "entity<|#|>Apple Inc<|#|>organization<|#|>..."
```

### 2. Contrastive Learning

**Used in**: Embedding models

```
Training Data: (anchor, positive, negative) triplets
Loss: InfoNCE (contrastive loss)

Example:
Anchor: "Apple company"
Positive: "Apple Inc produces iPhones"
Negative: "Apples are fruits"

→ Learn: sim(anchor, positive) > sim(anchor, negative)
```

### 3. Unsupervised Clustering

**Used in**: Community detection (Louvain)

```
Objective: Maximize modularity Q
Q = Σ [A_ij - (k_i * k_j)/(2m)] * δ(c_i, c_j)

Greedy optimization:
1. Each node = own community
2. Move nodes to maximize Q
3. Aggregate communities
4. Repeat until convergence
```

---

## Example: Full ML Pipeline

```python
# Step 1: Entity Extraction (LLM)
entities_text = await llm_generate(
    model="gpt-4",
    prompt=entity_extraction_prompt(text),
)
entities = parse_entities(entities_text)

# Step 2: Embedding Generation (Encoder)
entity_embeddings = await embedding_model(
    texts=[e["name"] for e in entities],
    model="text-embedding-3-small"
)

# Step 3: Community Detection (Graph ML)
communities = community.best_partition(entity_graph)

# Step 4: Query Processing
query_embedding = await embedding_model([query])

# Step 5: Vector Retrieval + Reranking
candidates = await vector_db.query(query_embedding, top_k=100)
reranked = await rerank(query, candidates, top_n=20)

# Step 6: Answer Generation (LLM)
answer = await llm_generate(
    model="gpt-4",
    prompt=rag_prompt(query, reranked, entities, graph)
)
```

---

**Версия**: 1.0
**Дата**: 2025-01-13
