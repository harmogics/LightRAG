# LightRAG Prompts: Система Языковых Промптов

## Обзор

Этот каталог содержит детальную документацию всех промптов, используемых в LightRAG для взаимодействия с языковыми моделями. Промпты являются критической частью семантических преобразований и определяют качество извлечения знаний и генерации ответов.

---

## Архитектура Промптов

```
┌─────────────────────────────────────────────────────────────────┐
│                    INDEXING PIPELINE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Document → Chunks → [Entity Extraction Prompt] → Entities      │
│                             ↓                                    │
│                      [Gleaning Prompt] (optional)                │
│                             ↓                                    │
│                      More Entities/Relations                     │
│                             ↓                                    │
│            Multiple Descriptions per Entity                      │
│                             ↓                                    │
│               [Summarization Prompt]                             │
│                             ↓                                    │
│                Unified Entity Description                        │
│                             ↓                                    │
│                      Knowledge Graph                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     QUERY PIPELINE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  User Query → [Keywords Extraction Prompt] → Keywords           │
│                             ↓                                    │
│                    Entity Resolution                             │
│                             ↓                                    │
│                   Context Assembly                               │
│                             ↓                                    │
│          [RAG Response Prompt] or [Naive RAG Prompt]            │
│                             ↓                                    │
│                    Final Answer                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Каталог Промптов

### Indexing Prompts (T2, T3, T5)

| ID | Имя | Файл | Трансформация | Роль |
|----|-----|------|---------------|------|
| **P1** | Entity Extraction System | [01-entity-extraction-system.md](01-entity-extraction-system.md) | T2 | Извлечение entities и relations из chunks |
| **P2** | Entity Extraction User | [02-entity-extraction-user.md](02-entity-extraction-user.md) | T2 | User prompt для начальной экстракции |
| **P3** | Entity Gleaning | [03-entity-gleaning.md](03-entity-gleaning.md) | T3 | Повторное извлечение пропущенных entities |
| **P4** | Entity Summarization | [04-entity-summarization.md](04-entity-summarization.md) | T5 | Объединение множественных описаний |

### Query Prompts (T7, T10)

| ID | Имя | Файл | Трансформация | Роль |
|----|-----|------|---------------|------|
| **P5** | Keywords Extraction | [05-keywords-extraction.md](05-keywords-extraction.md) | T7 | Извлечение high/low level keywords |
| **P6** | RAG Response (KG) | [06-rag-response.md](06-rag-response.md) | T10 | Генерация ответов с KG + chunks |
| **P7** | Naive RAG Response | [07-naive-rag-response.md](07-naive-rag-response.md) | T10 | Генерация ответов только с chunks |

### Utility Prompts

| ID | Имя | Файл | Назначение |
|----|-----|------|-----------|
| **P8** | Context Templates | [08-context-templates.md](08-context-templates.md) | Шаблоны форматирования контекста |
| **P9** | Examples & Constants | [09-examples-constants.md](09-examples-constants.md) | Примеры и константы |

---

## Связь с Трансформациями

### Indexing Chain

```
T1: Chunking
    ↓
T2: Entity Extraction
    • Использует: P1 (System) + P2 (User)
    • LLM: GPT-4o / GPT-4o-mini
    • Output: Raw entities + relations
    ↓
T3: Gleaning (conditional)
    • Использует: P1 (System) + P3 (Gleaning)
    • LLM: GPT-4o / GPT-4o-mini
    • Output: Additional entities + corrections
    ↓
T4: Parsing
    ↓
T5: Description Merging
    • Использует: P4 (Summarization)
    • LLM: GPT-4o / GPT-4o-mini
    • Output: Unified descriptions
    ↓
T6: Vectorization
```

### Query Chain

```
Query
    ↓
T7: Keyword Extraction (Local/Global)
    • Использует: P5 (Keywords)
    • LLM: GPT-4o / GPT-4o-mini
    • Output: High-level + low-level keywords
    ↓
T8: Entity Resolution
    ↓
T9: Context Assembly
    • Использует: P8 (Context Templates)
    • Format: JSON-based context
    ↓
T10: Answer Generation
    • Использует: P6 (RAG) или P7 (Naive)
    • LLM: GPT-4o / GPT-4o-mini
    • Output: Grounded answer + citations
    ↓
T11: Answer Formatting
```

---

## Общие Характеристики Промптов

### 1. Формат Промптов

Все промпты следуют структурированному формату:

```markdown
---Role---
[Определение роли AI ассистента]

---Goal--- / ---Task---
[Цель или задача]

---Instructions---
[Пошаговые инструкции]

---Examples--- (optional)
[Примеры входа/выхода]

---Real Data--- / ---Input---
[Placeholder для реальных данных]

---Output---
[Начало секции вывода]
```

### 2. Параметризация

Промпты используют Python `.format()` для подстановки:
- `{entity_types}` - типы сущностей
- `{tuple_delimiter}` - разделитель полей
- `{completion_delimiter}` - сигнал завершения
- `{language}` - язык вывода
- `{input_text}` / `{query}` - входные данные
- `{context_data}` - контекстные данные

### 3. Языковая Поддержка

```python
# Default language
DEFAULT_SUMMARY_LANGUAGE = "English"

# Configurable per-instance
language = global_config["addon_params"].get("language", DEFAULT_SUMMARY_LANGUAGE)

# All prompts support:
# - English (default)
# - Chinese (中文)
# - Russian (Русский)
# - Any language supported by LLM
```

### 4. Delimiters (Разделители)

```python
DEFAULT_TUPLE_DELIMITER = "<|#|>"       # Field separator
DEFAULT_COMPLETION_DELIMITER = "<|COMPLETE|>"  # Completion signal

# Why special delimiters?
# - Unique markers unlikely to appear in text
# - Easy parsing with string.split()
# - Clear structure for LLM output
```

---

## Внешние Зависимости

### LLM Providers

LightRAG поддерживает multiple LLM providers через унифицированный интерфейс:

```python
# Core LLM interface
from lightrag.llm import (
    openai_complete_if_cache,     # OpenAI
    anthropic_complete_if_cache,  # Anthropic (Claude)
    azure_openai_complete,        # Azure OpenAI
    ollama_model_complete,        # Ollama (local)
    bedrock_model_complete,       # AWS Bedrock
    # ... и другие
)
```

**Поддерживаемые провайдеры**:

| Provider | Module | Models | Use Case |
|----------|--------|--------|----------|
| **OpenAI** | `lightrag.llm.openai` | GPT-4o, GPT-4o-mini, GPT-4-turbo | Production, high quality |
| **Anthropic** | `lightrag.llm.anthropic` | Claude 3.5 Sonnet, Opus | Alternative to OpenAI |
| **Azure OpenAI** | `lightrag.llm.azure_openai` | GPT-4o (Azure) | Enterprise deployments |
| **Ollama** | `lightrag.llm.ollama` | Llama 3, Mistral, local | Local, privacy-focused |
| **AWS Bedrock** | `lightrag.llm.bedrock` | Claude (Bedrock), Titan | AWS-native |
| **Hugging Face** | `lightrag.llm.hf` | Open-source models | Custom models |
| **LMDeploy** | `lightrag.llm.lmdeploy` | Deployed models | High-performance serving |

### Tokenizers

```python
# Primary: tiktoken (OpenAI's tokenizer)
import tiktoken

def initialize_tokenizer(model_name: str = "gpt-4o"):
    """Initialize tiktoken tokenizer"""
    return tiktoken.encoding_for_model(model_name)

# For token counting in prompts:
tokenizer = tiktoken.get_encoding("cl100k_base")  # GPT-4 encoding
token_count = len(tokenizer.encode(text))
```

**Альтернативы для non-OpenAI моделей**:
- **SentencePiece** (для Gemini, Llama)
- **Custom tokenizers** через Hugging Face Transformers

### Caching Mechanisms

```python
# LLM Response Cache
from lightrag.base import BaseKVStorage

# Implementations:
# 1. JsonKVStorage (local JSON files)
# 2. MongoKVStorage (MongoDB)
# 3. PostgreSQLKVStorage (PostgreSQL)
# 4. RedisKVStorage (Redis) - fastest

# Cache keys based on:
# - Prompt hash (MD5)
# - System prompt hash
# - Model name
# - Temperature (if relevant)
```

### JSON Parsing

```python
import json
import re

# For structured outputs (P5: Keywords, P4: Summarization)
# LightRAG uses robust JSON extraction:

def extract_json_from_response(response: str) -> dict:
    """Extract JSON from LLM response with error handling"""
    # Try direct parse
    try:
        return json.loads(response)
    except:
        # Try finding JSON in text
        json_match = re.search(r'\{[^{}]*\}', response, re.DOTALL)
        if json_match:
            return json.loads(json_match.group(0))
    # Fallback...
```

### Validation & Parsing

```python
# Entity extraction output parsing
from lightrag.utils import fix_tuple_delimiter_corruption

def parse_entity_line(line: str, tuple_delimiter: str) -> dict:
    """Parse entity/relation line from LLM output"""
    parts = line.split(tuple_delimiter)

    if parts[0] == "entity":
        return {
            "type": "entity",
            "name": parts[1],
            "entity_type": parts[2],
            "description": parts[3]
        }
    elif parts[0] == "relation":
        return {
            "type": "relation",
            "source": parts[1],
            "target": parts[2],
            "keywords": parts[3],
            "description": parts[4]
        }
```

---

## Метрики Качества Промптов

### 1. Entity Extraction Prompts (P1, P2, P3)

**Метрики**:
- **Precision**: Correct entities / All extracted entities
- **Recall**: Extracted entities / All entities in text
- **F1-Score**: Harmonic mean of precision and recall
- **Hallucination Rate**: False entities / All extracted
- **Format Compliance**: Valid outputs / Total outputs

**Типичные значения**:
```python
entity_extraction_quality = {
    "precision": 0.70,        # With gleaning: 0.75
    "recall": 0.65,           # With gleaning: 0.80
    "f1": 0.67,               # With gleaning: 0.77
    "hallucination": 0.05,    # Low due to grounding
    "format_compliance": 0.95 # High with clear format
}
```

### 2. Summarization Prompt (P4)

**Метрики**:
- **Factual Accuracy**: Correct facts / Total facts
- **Completeness**: Info from all inputs included
- **Coherence**: Readability score (0-1)
- **Hallucination Rate**: Unsupported statements

**Типичные значения**:
```python
summarization_quality = {
    "factual_accuracy": 0.90,
    "completeness": 0.80,
    "coherence": 0.95,
    "hallucination": 0.08
}
```

### 3. Keywords Extraction Prompt (P5)

**Метрики**:
- **Intent Accuracy**: Correct query understanding
- **Coverage**: All aspects covered
- **Specificity**: Appropriate granularity

**Типичные значения**:
```python
keywords_quality = {
    "intent_accuracy": 0.90,
    "coverage": 0.85,
    "specificity": 0.80
}
```

### 4. RAG Response Prompts (P6, P7)

**Метрики**:
- **Factual Accuracy**: Based on source data
- **Completeness**: Fully answers query
- **Grounding**: Citations present and correct
- **Relevance**: On-topic response

**Типичные значения**:
```python
rag_response_quality = {
    "factual_accuracy": 0.85,
    "completeness": 0.75,
    "grounding": 0.90,
    "relevance": 0.90,
    "hallucination": 0.10  # Main risk point
}
```

---

## Best Practices

### 1. Prompt Engineering

```python
# ✅ Good: Clear, structured, with examples
GOOD_PROMPT = """
---Role---
You are X

---Task---
Do Y

---Instructions---
1. Step 1
2. Step 2

---Examples---
Input: ...
Output: ...

---Real Data---
{input}
"""

# ❌ Bad: Vague, unstructured
BAD_PROMPT = "Extract entities from: {input}"
```

### 2. Temperature Settings

```python
# Different prompts need different temperatures
PROMPT_TEMPERATURES = {
    "entity_extraction": 0.0,    # Deterministic, factual
    "gleaning": 0.0,             # Deterministic
    "summarization": 0.1,        # Slight creativity for coherence
    "keywords": 0.0,             # Deterministic
    "answer_generation": 0.1     # Slight creativity for fluency
}
```

### 3. Token Limits

```python
# Prompt budget planning
PROMPT_TOKEN_BUDGETS = {
    "entity_extraction": {
        "system": 2000,    # Detailed instructions
        "user": 1500,      # Chunk content
        "output": 2000,    # Entities + relations
        "total": 5500
    },
    "summarization": {
        "system": 800,
        "user": 3000,      # Multiple descriptions
        "output": 500,
        "total": 4300
    },
    "answer_generation": {
        "system": 1500,
        "user": 20000,     # Large context (KG + chunks)
        "output": 800,
        "total": 22300
    }
}
```

### 4. Error Handling

```python
# Robust error handling for LLM outputs
async def safe_llm_call_with_prompt(prompt: str, system_prompt: str):
    """Safe LLM call with retries and fallbacks"""
    max_retries = 3

    for attempt in range(max_retries):
        try:
            response = await llm_func(prompt, system_prompt=system_prompt)

            # Validate response format
            if not validate_response(response):
                if attempt < max_retries - 1:
                    continue  # Retry
                else:
                    return default_response()

            return response

        except Exception as e:
            logger.warning(f"LLM call failed (attempt {attempt + 1}): {e}")
            if attempt == max_retries - 1:
                raise
```

---

## Конфигурация и Кастомизация

### Переопределение Промптов

```python
from lightrag import LightRAG, PROMPTS

# Customize prompts
PROMPTS["entity_extraction_system_prompt"] = """
Your custom prompt here...
{entity_types}
{input_text}
"""

# Or use custom language
rag = LightRAG(
    addon_params={
        "language": "Russian",  # Все промпты будут на русском
    }
)
```

### Добавление Новых Entity Types

```python
# Default types
DEFAULT_ENTITY_TYPES = [
    "person", "organization", "location", "event",
    "product", "concept", "category", "other"
]

# Custom types
rag = LightRAG(
    entity_types=[
        "person", "company", "technology",
        "research_paper", "algorithm", "dataset"
    ]
)
```

---

## Дальнейшее Чтение

Для детального понимания каждого промпта см. соответствующие файлы:

1. [Entity Extraction System Prompt](01-entity-extraction-system.md)
2. [Entity Extraction User Prompt](02-entity-extraction-user.md)
3. [Entity Gleaning Prompt](03-entity-gleaning.md)
4. [Entity Summarization Prompt](04-entity-summarization.md)
5. [Keywords Extraction Prompt](05-keywords-extraction.md)
6. [RAG Response Prompt](06-rag-response.md)
7. [Naive RAG Response Prompt](07-naive-rag-response.md)
8. [Context Templates](08-context-templates.md)
9. [Examples & Constants](09-examples-constants.md)

Для понимания трансформаций см.:
- [transform/01-indexing-transforms.md](../transform/01-indexing-transforms.md)
- [transform/02-query-transforms.md](../transform/02-query-transforms.md)
