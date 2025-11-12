# LLM Agents: Роли и Функции Языковых Агентов

## Обзор

LightRAG использует языковые модели (LLM) в качестве интеллектуальных агентов на критических этапах pipeline. Каждый агент имеет специфическую роль, промпт-инструкции и выполняет определенные функции для преобразования неструктурированного текста в структурированный граф знаний.

## Архитектура LLM Агентов

```
┌─────────────────────────────────────────────────────────┐
│              LLM AGENTS ECOSYSTEM                       │
│                                                         │
│  ┌────────────────────────────────────────────────┐   │
│  │  AGENT 1: Entity Extraction Agent              │   │
│  │  Role: Initial entity and relationship         │   │
│  │        extraction from text chunks             │   │
│  │  Priority: 0-7 (based on chunk order)          │   │
│  │  Prompt: entity_extraction_system_prompt       │   │
│  └────────────────┬───────────────────────────────┘   │
│                   │                                     │
│                   ▼                                     │
│  ┌────────────────────────────────────────────────┐   │
│  │  AGENT 2: Gleaning Agent                       │   │
│  │  Role: Identify missed or incorrectly          │   │
│  │        formatted entities/relationships        │   │
│  │  Priority: 0-7 (based on chunk order)          │   │
│  │  Prompt: entity_continue_extraction_*_prompt   │   │
│  └────────────────┬───────────────────────────────┘   │
│                   │                                     │
│                   ▼                                     │
│  ┌────────────────────────────────────────────────┐   │
│  │  AGENT 3: Description Summarization Agent      │   │
│  │  Role: Merge multiple descriptions into        │   │
│  │        cohesive summary                        │   │
│  │  Priority: 8 (high priority)                   │   │
│  │  Prompt: summarize_entity_descriptions         │   │
│  └────────────────────────────────────────────────┘   │
│                                                         │
│  ┌────────────────────────────────────────────────┐   │
│  │  AGENT 4: Query Processing Agents              │   │
│  │  • Keyword Extraction Agent                    │   │
│  │  • Context Generation Agent                    │   │
│  │  • Response Generation Agent                   │   │
│  │  Priority: Variable                            │   │
│  └────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Agent 1: Entity Extraction Agent

### Роль

**Knowledge Graph Specialist** - извлечение сущностей и отношений из текстовых chunks

### Ответственности

1. **Идентификация сущностей**
   - Распознавание четко определенных entities в тексте
   - Категоризация по типам (person, organization, location, etc.)
   - Создание comprehensive описаний

2. **Извлечение отношений**
   - Идентификация прямых, явных связей между entities
   - Декомпозиция N-арных отношений в бинарные
   - Определение ключевых слов и описания отношений

3. **Форматирование output**
   - Структурированный output с delimiters
   - Консистентная нормализация имен (Title Case)
   - Соблюдение third-person perspective

### Промпт

**System Prompt**: `entity_extraction_system_prompt` (lightrag/prompt.py:10)

**Ключевые Инструкции**:

```
---Role---
You are a Knowledge Graph Specialist responsible for extracting entities
and relationships from the input text.

---Instructions---
1. Entity Extraction & Output:
   * Identification: Identify clearly defined and meaningful entities
   * Entity Details:
     - entity_name: Title case normalized name
     - entity_type: Categorized type from predefined list
     - entity_description: Comprehensive description
   * Output Format:
     entity<|#|>entity_name<|#|>entity_type<|#|>entity_description

2. Relationship Extraction & Output:
   * Identification: Direct, clearly stated relationships
   * N-ary Decomposition: Break down into binary relationships
   * Relationship Details:
     - source_entity: Source entity name (Title Case)
     - target_entity: Target entity name (Title Case)
     - relationship_keywords: High-level keywords (comma separated)
     - relationship_description: Clear rationale for connection
   * Output Format:
     relation<|#|>source<|#|>target<|#|>keywords<|#|>description

3. Delimiter Usage: <|#|> is atomic, must not be filled with content

4. Relationship Direction: Treat as undirected unless explicitly stated

5. Output Order: Entities first, then relationships (by importance)

6. Context & Objectivity: Third person, explicit naming, avoid pronouns

7. Language: Output in {language}, preserve proper nouns

8. Completion Signal: Output <|COMPLETE|> after all extraction
```

### Входные Данные

```python
{
    "input_text": str,              # Текст chunk для обработки
    "entity_types": list[str],       # Допустимые типы сущностей
    "language": str,                 # Язык output
    "tuple_delimiter": str,          # "<|#|>"
    "completion_delimiter": str,     # "<|COMPLETE|>"
    "examples": str                  # Примеры extraction
}
```

### Выходные Данные

```
entity<|#|>Apple Inc<|#|>organization<|#|>Apple Inc is a technology company...
entity<|#|>Tim Cook<|#|>person<|#|>Tim Cook is the CEO of Apple Inc...
entity<|#|>iPhone<|#|>product<|#|>iPhone is a smartphone designed by Apple Inc...
relation<|#|>Tim Cook<|#|>Apple Inc<|#|>leadership, management<|#|>Tim Cook serves as CEO...
relation<|#|>Apple Inc<|#|>iPhone<|#|>product, manufacturing<|#|>Apple Inc designs iPhone...
<|COMPLETE|>
```

### Вызов Агента

```python
# Функция: extract_entities() в operate.py:2010
final_result, timestamp = await use_llm_func_with_cache(
    entity_extraction_user_prompt,
    use_llm_func,
    system_prompt=entity_extraction_system_prompt,
    llm_response_cache=llm_response_cache,
    cache_type="extract",
    chunk_id=chunk_key,
    cache_keys_collector=cache_keys_collector
)
```

### Приоритет Выполнения

```python
# Priority 0-7 based on chunk_order_index
chunk_priority = min(chunk_data["chunk_order_index"], 7)

# Chunks в начале документа имеют больший приоритет
# Важные сущности часто упоминаются в начале
```

### Метрики

- **Execution time**: ~2-5 секунд per chunk (зависит от LLM)
- **Token consumption**: ~800 (prompt) + ~1024 (chunk) + ~500 (output) = ~2324 tokens
- **Cache hit rate**: 70-90% при re-processing

## Agent 2: Gleaning Agent

### Роль

**Quality Assurance Specialist** - выявление пропущенных или некорректных extractions

### Ответственности

1. **Идентификация пропусков**
   - Поиск entities пропущенных в initial extraction
   - Поиск relationships пропущенных в initial extraction

2. **Исправление ошибок**
   - Обнаружение truncated descriptions
   - Исправление missing fields
   - Коррекция incorrect formatting

3. **Улучшение качества**
   - Более детальные descriptions
   - Более точные categorizations

### Промпт

**User Prompt**: `entity_continue_extraction_user_prompt` (lightrag/prompt.py:82)

**Ключевые Инструкции**:

```
---Task---
Based on the last extraction task, identify and extract any **missed or
incorrectly formatted** entities and relationships from the input text.

---Instructions---
1. Focus on Corrections/Additions:
   * Do NOT re-output correctly extracted items
   * If missed: extract and output now
   * If truncated/incorrect: re-output corrected version

2. Output Format: Same format as initial extraction

3. Output Content Only: No explanations or remarks

4. Completion Signal: <|COMPLETE|> after all corrections

5. Language: Output in {language}
```

### Входные Данные

```python
{
    "history_messages": [
        {
            "role": "user",
            "content": entity_extraction_user_prompt
        },
        {
            "role": "assistant",
            "content": initial_extraction_result
        }
    ],
    "system_prompt": entity_extraction_system_prompt
}
```

### Выходные Данные

```
entity<|#|>Steve Jobs<|#|>person<|#|>Steve Jobs was co-founder of Apple Inc...
relation<|#|>Steve Jobs<|#|>Apple Inc<|#|>founder, visionary<|#|>Steve Jobs co-founded Apple...
<|COMPLETE|>
```

### Вызов Агента

```python
# Условие активации
if entity_extract_max_gleaning > 0:
    glean_result, timestamp = await use_llm_func_with_cache(
        entity_continue_extraction_user_prompt,
        use_llm_func,
        system_prompt=entity_extraction_system_prompt,
        llm_response_cache=llm_response_cache,
        history_messages=history,  # История с initial extraction
        cache_type="extract",
        chunk_id=chunk_key,
        cache_keys_collector=cache_keys_collector
    )
```

### Merging Strategy

```python
# Сравнение description lengths
for entity_name, glean_entities in glean_nodes.items():
    if entity_name in maybe_nodes:
        original_desc_len = len(
            maybe_nodes[entity_name][0].get("description", "") or ""
        )
        glean_desc_len = len(glean_entities[0].get("description", "") or "")

        # Выбор более длинного (более информативного) описания
        if glean_desc_len > original_desc_len:
            maybe_nodes[entity_name] = list(glean_entities)
    else:
        # Новая сущность из gleaning
        maybe_nodes[entity_name] = list(glean_entities)
```

### Метрики

- **Execution time**: ~2-5 секунд per chunk
- **Token consumption**: ~3000-4000 tokens (включая history)
- **Improvement rate**: 10-20% дополнительных entities
- **Activation rate**: ~90% chunks (если enabled)

## Agent 3: Description Summarization Agent

### Роль

**Knowledge Curator** - синтез множественных descriptions в cohesive summary

### Ответственности

1. **Агрегация информации**
   - Объединение descriptions из разных chunks
   - Сохранение всех ключевых фактов
   - Устранение дублирования

2. **Разрешение конфликтов**
   - Идентификация conflicting descriptions
   - Определение distinct entities с одинаковым именем
   - Reconciliation или presentation обоих viewpoints

3. **Оптимизация длины**
   - Соблюдение token limits
   - Сохранение информативности
   - Приоритизация важной информации

### Промпт

**System Prompt**: `summarize_entity_descriptions` (lightrag/prompt.py:174)

**Ключевые Инструкции**:

```
---Role---
You are a Knowledge Graph Specialist, proficient in data curation and synthesis.

---Task---
Synthesize a list of descriptions of a given entity or relation into a
single, comprehensive, and cohesive summary.

---Instructions---
1. Input Format: JSON objects, one per line

2. Output Format: Plain text, multiple paragraphs

3. Comprehensiveness: Integrate ALL key information from every description

4. Context & Objectivity:
   - Third-person perspective
   - Explicitly mention entity/relation name at beginning

5. Conflict Handling:
   - If distinct entities share name: separate summaries
   - If historical discrepancies: reconcile or present both viewpoints

6. Length Constraint: Must not exceed {summary_length} tokens

7. Language: Output in {language}
```

### Входные Данные

```python
{
    "description_type": str,           # "Entity" or "Relationship"
    "description_name": str,           # Entity/relation name
    "description_list": list[str],     # List of descriptions to merge
    "summary_length": int,             # Max tokens (default: 500)
    "language": str                    # Output language
}
```

### Выходные Данные

```
Apple Inc is a multinational technology company headquartered in Cupertino,
California. Founded in 1976 by Steve Jobs, Steve Wozniak, and Ronald Wayne,
the company has become a global leader in consumer electronics, computer
software, and online services. Apple Inc is particularly known for its
flagship products including iPhone, iPad, Mac computers, Apple Watch, and
AirPods. The company revolutionized multiple industries with its innovative
products and user-centric design philosophy. Under the leadership of CEO
Tim Cook since 2011, Apple Inc has continued to expand its ecosystem of
products and services, maintaining its position as one of the world's most
valuable companies.
```

### Decision Tree

```
Input: list[description]
    ↓
[1. Empty Check]
if len == 0: return ("", False)
    ↓
[2. Single Description]
if len == 1: return (descriptions[0], False)
    ↓
[3. Token Count Check]
total_tokens = count_tokens(combined_text)
if total_tokens < summary_context_size AND
   len < force_llm_summary_on_merge:
    return (combined_text, False)  # No LLM needed
    ↓
[4. Single LLM Call]
if total_tokens < summary_max_tokens:
    return await _single_summary_call(...)
    ↓
[5. Map-Reduce Strategy]
return await _map_reduce_summary(...)
```

### Map-Reduce Implementation

```python
async def _map_reduce_summary(description_list, entity_name):
    """
    Map-Reduce для больших списков descriptions

    MAP Phase:
    - Разбить на chunks по token limit
    - Summary каждого chunk параллельно

    REDUCE Phase:
    - Рекурсивно объединить summaries
    """

    # === MAP ===
    chunks = split_into_chunks(description_list, summary_context_size)

    chunk_summaries = await asyncio.gather(*[
        _single_summary_call(chunk, entity_name)
        for chunk in chunks
    ])

    # === REDUCE ===
    if len(chunk_summaries) == 1:
        return chunk_summaries[0]
    else:
        # Рекурсивный вызов
        return await _handle_entity_relation_summary(
            entity_name,
            chunk_summaries,  # Summaries как новые descriptions
            global_config,
            llm_response_cache
        )
```

### Вызов Агента

```python
# Функция: _handle_entity_relation_summary() в operate.py:121
merged_description, llm_used = await _handle_entity_relation_summary(
    description_type="Entity",
    entity_or_relation_name=entity_name,
    description_list=all_descriptions,
    seperator="\n\n",
    global_config=global_config,
    llm_response_cache=llm_response_cache
)
```

### Приоритет Выполнения

```python
# Priority 8 (высокий приоритет)
# Summary критичен для консистентности графа
# Выполняется после extraction агентов (priority 0-7)
```

### Метрики

- **Execution time**: ~3-8 секунд (зависит от количества descriptions)
- **Token consumption**: ~500-2000 tokens per call
- **Compression ratio**: ~3-5x (3-5 descriptions → 1 summary)
- **Activation rate**: ~40-60% entities (требующих summarization)

## Agent 4: Query Processing Agents

### Роль

**Query Understanding & Response Generation Specialists**

### 4.1 Keyword Extraction Agent

**Ответственности**:
- Извлечение high-level и low-level keywords из query
- Понимание intent пользователя
- Формирование search terms для graph traversal

**Промпт**: `keywords_extraction` (lightrag/prompt.py)

**Пример**:
```
Input: "What products does Apple Inc manufacture?"

Output:
High-level: ["Apple Inc", "products", "manufacturing"]
Low-level: ["iPhone", "iPad", "Mac", "consumer electronics"]
```

### 4.2 Context Generation Agent

**Ответственности**:
- Формирование контекста из retrieved chunks и entities
- Ранжирование релевантности
- Создание structured context для response generation

**Процесс**:
1. Vector search по chunks
2. Graph traversal от релевантных entities
3. Формирование unified context

### 4.3 Response Generation Agent

**Ответственности**:
- Генерация ответа на основе context
- Включение citations к source documents
- Обеспечение factuality и relevance

**Промпт**: `rag_response` или `naive_rag_response`

## LLM Wrapper Function

### Функция: `use_llm_func_with_cache()`

**Расположение**: `lightrag/utils.py:1594`

```python
async def use_llm_func_with_cache(
    prompt: str,
    llm_func: callable,
    system_prompt: str | None = None,
    llm_response_cache: BaseKVStorage | None = None,
    history_messages: list | None = None,
    cache_type: str = "extract",
    chunk_id: str | None = None,
    cache_keys_collector: list | None = None,
) -> tuple[str, int]:
    """
    LLM вызов с кешированием и санитизацией

    Features:
    - MD5-based cache keys
    - Automatic text sanitization
    - Priority-based execution
    - Streaming support (optional)
    - Error handling with retries

    Returns:
        (response_text, timestamp)
    """
```

### Кеширование

```python
# Cache key generation
cache_content = system_prompt + prompt
if history_messages:
    cache_content += json.dumps(history_messages)

cache_key = compute_mdhash_id(cache_content)
cache_key = f"{cache_type}:{chunk_id}:{cache_key}"

# Cache lookup
cached_data = await llm_response_cache.get_by_id(cache_key)
if cached_data:
    return (cached_data["response"], cached_data["timestamp"])

# Cache MISS - call LLM
response = await llm_func(prompt, system_prompt=system_prompt)

# Cache store
await llm_response_cache.upsert({
    cache_key: {
        "response": response,
        "timestamp": int(time.time()),
        "cache_type": cache_type,
        "chunk_id": chunk_id
    }
})
```

### Санитизация

```python
# Text sanitization для безопасного UTF-8 encoding
response = sanitize_text_for_encoding(response)

# Удаление проблемных символов:
# - Invalid UTF-8 bytes
# - Control characters (кроме \n, \r, \t)
# - Нормализация whitespace
```

## LLM Provider: OpenAI Implementation

**Расположение**: `lightrag/llm/openai.py`

```python
async def openai_complete_if_cache(
    prompt: str,
    system_prompt: str | None = None,
    history_messages: list | None = None,
    **kwargs
) -> str:
    """
    OpenAI-compatible LLM call

    Features:
    - Async API calls
    - Retry logic with exponential backoff
    - Rate limit handling
    - Streaming support
    - Custom base URL support
    """

    openai_async_client = AsyncOpenAI(
        api_key=kwargs.get("api_key"),
        base_url=kwargs.get("base_url")
    )

    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    if history_messages:
        messages.extend(history_messages)
    messages.append({"role": "user", "content": prompt})

    # Retry logic
    for attempt in range(max_retries):
        try:
            response = await openai_async_client.chat.completions.create(
                model=kwargs.get("model", "gpt-4"),
                messages=messages,
                temperature=kwargs.get("temperature", 0.0),
                max_tokens=kwargs.get("max_tokens", 1000)
            )

            return response.choices[0].message.content

        except OpenAIError as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                await asyncio.sleep(wait_time)
            else:
                raise
```

## Agent Configuration

### Global Config

```python
global_config = {
    "llm_model_func": openai_complete_if_cache,
    "llm_model_name": "gpt-4-turbo-preview",
    "llm_model_max_async": 16,        # Concurrent LLM calls
    "llm_model_max_token_size": 32768,
    "llm_model_kwargs": {
        "temperature": 0.0,
        "max_tokens": 1000
    },

    "entity_extract_max_gleaning": 1, # Enable gleaning

    "summary_max_tokens": 500,
    "summary_context_size": 4096,
    "force_llm_summary_on_merge": 3,

    "addon_params": {
        "language": "English",
        "entity_types": [
            "person", "organization", "location",
            "event", "product", "concept"
        ]
    }
}
```

## Best Practices

### ✅ DO

1. **Используйте кеширование** для избежания duplicate LLM calls
2. **Настраивайте temperature=0** для детерминированности
3. **Мониторьте token usage** для cost optimization
4. **Включайте gleaning** для improved extraction quality
5. **Используйте system prompts** для consistent behavior

### ❌ DON'T

1. **Не игнорируйте rate limits** - используйте retry logic
2. **Не используйте высокий temperature** для extraction tasks
3. **Не пропускайте sanitization** - риск encoding errors
4. **Не устанавливайте слишком низкий llm_model_max_async** - underutilization
5. **Не забывайте про cache invalidation** при изменении prompts

## Следующий Этап

Теперь перейдем к описанию **[Semantic Layer](07-semantic-layer.md)** - концептуального слоя с векторными представлениями и graph-based семантикой.
