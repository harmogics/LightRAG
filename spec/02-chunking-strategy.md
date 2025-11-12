# Chunking Strategy: Разбиение Документов на Семантические Фрагменты

## Обзор

Chunking Layer отвечает за разбиение длинных документов на управляемые фрагменты (chunks), которые могут быть эффективно обработаны языковыми моделями. Качественная стратегия chunking критически важна для извлечения точных сущностей и сохранения семантического контекста.

## Главная Функция

**Расположение**: `lightrag/operate.py:66`

```python
def chunking_by_token_size(
    tokenizer: Tokenizer,
    content: str,
    split_by_character: str | None = None,
    split_by_character_only: bool = False,
    overlap_token_size: int = 128,
    max_token_size: int = 1024,
) -> list[dict[str, Any]]
```

## Параметры Конфигурации

| Параметр | Тип | Default | Описание |
|----------|-----|---------|----------|
| `tokenizer` | `Tokenizer` | - | Токенизатор для подсчета токенов (tiktoken) |
| `content` | `str` | - | Текст документа для разбиения |
| `split_by_character` | `str \| None` | `None` | Символ/строка для предварительного разделения |
| `split_by_character_only` | `bool` | `False` | Только разделение по символу без token chunking |
| `overlap_token_size` | `int` | `128` | Размер перекрытия между chunks (в токенах) |
| `max_token_size` | `int` | `1024` | Максимальный размер chunk (в токенах) |

## Стратегии Chunking

### Strategy 1: Pure Token-Based Chunking

**Когда использовать**: документы без естественных границ (continuous text)

```python
# Пример использования
chunks = chunking_by_token_size(
    tokenizer=tiktoken_tokenizer,
    content=document_text,
    max_token_size=1024,
    overlap_token_size=128
)
```

**Алгоритм**:

```
Input: content, max_token_size, overlap_token_size
    ↓
[1. Токенизация Документа]
    ↓
    tokens = tokenizer.encode(content)
    total_tokens = len(tokens)
    ↓
[2. Вычисление Stride]
    ↓
    stride = max_token_size - overlap_token_size
    # Пример: 1024 - 128 = 896 токенов на шаг
    ↓
[3. Создание Chunks с Перекрытием]
    ↓
    for index, start in enumerate(range(0, total_tokens, stride)):
        # Извлечение токенов для chunk
        chunk_tokens = tokens[start : start + max_token_size]

        # Декодирование обратно в текст
        chunk_content = tokenizer.decode(chunk_tokens)

        # Подсчет фактического размера
        actual_size = min(max_token_size, total_tokens - start)

        # Создание chunk metadata
        chunk = {
            "tokens": actual_size,
            "content": chunk_content.strip(),
            "chunk_order_index": index
        }
    ↓
Output: list[dict] chunks
```

**Визуализация Overlap**:

```
Document: [======================================] (3000 tokens)

Chunk 0: [████████████████] (1024 tokens)
         [0          1024]

Chunk 1:          [████████████████] (1024 tokens)
                  [896         1920]
                  ↑
                  128 tokens overlap

Chunk 2:                    [████████████████] (1024 tokens)
                            [1792        2816]
                            ↑
                            128 tokens overlap

Chunk 3:                              [████] (184 tokens)
                                      [2688  2872]
                                      ↑
                                      128 tokens overlap
```

### Strategy 2: Character-Based Pre-Splitting

**Когда использовать**: документы со структурой (параграфы, секции, предложения)

```python
# Разделение по параграфам
chunks = chunking_by_token_size(
    tokenizer=tiktoken_tokenizer,
    content=document_text,
    split_by_character="\n\n",  # Двойной перенос строки
    max_token_size=1024,
    overlap_token_size=128
)

# Разделение по предложениям
chunks = chunking_by_token_size(
    tokenizer=tiktoken_tokenizer,
    content=document_text,
    split_by_character=". ",  # Точка с пробелом
    max_token_size=1024,
    overlap_token_size=128
)
```

**Алгоритм**:

```
Input: content, split_by_character, max_token_size, overlap_token_size
    ↓
[1. Предварительное Разделение по Символу]
    ↓
    raw_chunks = content.split(split_by_character)
    # Пример: ["Paragraph 1", "Paragraph 2", ...]
    ↓
[2. Обработка Каждого Raw Chunk]
    ↓
    new_chunks = []
    for raw_chunk in raw_chunks:
        chunk_tokens = tokenizer.encode(raw_chunk)

        if len(chunk_tokens) > max_token_size:
            # Случай A: chunk слишком большой, нужно разбить
            # Применяем token-based chunking с overlap
            for start in range(0, len(chunk_tokens), stride):
                sub_chunk = tokenizer.decode(
                    chunk_tokens[start : start + max_token_size]
                )
                new_chunks.append((
                    min(max_token_size, len(chunk_tokens) - start),
                    sub_chunk
                ))
        else:
            # Случай B: chunk помещается целиком
            new_chunks.append((len(chunk_tokens), raw_chunk))
    ↓
[3. Создание Final Chunks]
    ↓
    for index, (token_count, chunk_text) in enumerate(new_chunks):
        chunk = {
            "tokens": token_count,
            "content": chunk_text.strip(),
            "chunk_order_index": index
        }
    ↓
Output: list[dict] chunks
```

**Преимущества**:
- ✅ Сохраняет естественные границы текста
- ✅ Chunks завершаются на законченных мыслях
- ✅ Улучшает качество entity extraction
- ✅ Минимизирует разрыв контекста

### Strategy 3: Character-Only Splitting

**Когда использовать**: точный контроль над границами, специфические форматы

```python
# Только разделение по разделителю, без token chunking
chunks = chunking_by_token_size(
    tokenizer=tiktoken_tokenizer,
    content=document_text,
    split_by_character="\n---\n",  # Markdown разделитель
    split_by_character_only=True
)
```

**Алгоритм**:

```
Input: content, split_by_character, split_by_character_only=True
    ↓
[1. Разделение по Символу]
    ↓
    raw_chunks = content.split(split_by_character)
    ↓
[2. Создание Chunks БЕЗ Дополнительного Разбиения]
    ↓
    for index, raw_chunk in enumerate(raw_chunks):
        chunk_tokens = tokenizer.encode(raw_chunk)

        chunk = {
            "tokens": len(chunk_tokens),
            "content": raw_chunk.strip(),
            "chunk_order_index": index
        }
        # ВАЖНО: не разбиваем дальше, даже если > max_token_size
    ↓
Output: list[dict] chunks
```

**⚠️ Внимание**: может создать chunks больше `max_token_size`!

## Chunk Metadata Structure

### Базовая Структура

```python
{
    "tokens": int,              # Количество токенов в chunk
    "content": str,             # Текстовое содержимое
    "chunk_order_index": int    # Порядковый номер (0-based)
}
```

### Расширенная Структура (После Storage)

```python
{
    "chunk_id": str,            # "chunk-{md5_hash}"
    "content": str,             # Текстовое содержимое
    "tokens": int,              # Количество токенов
    "chunk_order_index": int,   # Порядковый номер
    "full_doc_id": str,         # "doc-{md5_hash}"
    "file_path": str,           # Путь к исходному файлу
    "created_at": int           # Unix timestamp
}
```

## Семантические Соображения

### 1. Overlap для Сохранения Контекста

**Проблема**: сущности на границах chunks теряют контекст

```
Without Overlap:
Chunk 1: "...The company announced"
Chunk 2: "a new product line..."
         ↑
         Потерян контекст: какая компания?

With Overlap (128 tokens):
Chunk 1: "...The company announced"
Chunk 2: "The company announced a new product line..."
         ↑
         Контекст сохранен!
```

**Рекомендации**:
- Minimum overlap: 64 tokens (сохраняет 1-2 предложения)
- Optimal overlap: 128 tokens (сохраняет 2-3 предложения)
- Maximum overlap: 256 tokens (для сложных доменов)

### 2. Chunk Size и Context Window

**Соображения**:

```
LLM Context Window: 4096 tokens

Available for chunk: 4096
    - System Prompt: ~800 tokens
    - Examples: ~600 tokens
    - Output Buffer: ~600 tokens
    - Safety Margin: ~200 tokens
    --------------------------------
    = ~1900 tokens maximum для chunk
```

**Рекомендуемые размеры**:
- **Small chunks** (512 tokens): точная локализация, но фрагментированный контекст
- **Medium chunks** (1024 tokens): ✅ **оптимальный баланс**
- **Large chunks** (2048 tokens): богатый контекст, но риск пропуска деталей

### 3. Character-Based Boundaries

**Рекомендуемые разделители по типу документа**:

| Тип Документа | Разделитель | Пример |
|---------------|-------------|--------|
| Научные статьи | `\n\n` | Параграфы |
| Документация | `\n## ` | Markdown секции |
| Юридические тексты | `\n\n[0-9]+\.` | Нумерованные пункты |
| Книги | `\n\nChapter ` | Главы |
| Диалоги | `\n\n` | Реплики |
| Код | `\n\ndef ` или `\nclass ` | Функции/классы |

## Интеграция с Pipeline

### Вызов в Processing Phase

**Расположение**: `lightrag/lightrag.py` в `apipeline_process_enqueue_documents()`

```python
# Для каждого документа
for doc_id, content in documents.items():
    file_path = doc_metadata[doc_id]["file_path"]

    # === CHUNKING ===
    chunks = chunking_by_token_size(
        tokenizer=self.tokenizer,
        content=content,
        split_by_character=split_by_character,
        split_by_character_only=split_by_character_only,
        overlap_token_size=self.chunk_overlap_token_size,
        max_token_size=self.chunk_token_size
    )

    # Создание chunk metadata
    inserting_chunks = {}
    for chunk_data in chunks:
        chunk_key = compute_mdhash_id(
            chunk_data["content"],
            prefix="chunk-"
        )
        inserting_chunks[chunk_key] = {
            **chunk_data,
            "full_doc_id": doc_id,
            "file_path": file_path
        }

    # Дедупликация chunks
    existing_chunk_keys = await self.text_chunks.filter_keys(
        set(inserting_chunks.keys())
    )
    new_chunks = {
        k: v for k, v in inserting_chunks.items()
        if k not in existing_chunk_keys
    }

    # Параллельное сохранение и обработка
    await asyncio.gather(
        self.chunks_vdb.upsert(new_chunks),
        self.text_chunks.upsert(new_chunks),
        self._process_extract_entities(new_chunks)
    )
```

## Оптимизация Performance

### 1. Efficient Tokenization

```python
# ❌ Плохо: повторная токенизация
for chunk in chunks:
    tokens = tokenizer.encode(chunk)
    # ...

# ✅ Хорошо: токенизация один раз
tokens = tokenizer.encode(full_content)
for start in range(0, len(tokens), stride):
    chunk_tokens = tokens[start : start + max_token_size]
    # ...
```

### 2. Lazy Content Decoding

```python
# Декодируем только когда необходимо
chunk_tokens = tokens[start : start + max_token_size]

# Откладываем декодирование до использования
if need_text:
    chunk_content = tokenizer.decode(chunk_tokens)
```

### 3. Batch Processing

```python
# Обработка нескольких документов параллельно
async def chunk_documents(documents: list[str]) -> list[list[dict]]:
    tasks = [
        chunking_by_token_size(tokenizer, doc)
        for doc in documents
    ]
    return await asyncio.gather(*tasks)
```

## Advanced Techniques

### 1. Semantic Boundary Detection

**Концепция**: использование NLP для определения смысловых границ

```python
def semantic_chunking(
    content: str,
    tokenizer: Tokenizer,
    max_token_size: int = 1024
) -> list[dict]:
    """
    Chunking с учетом семантических границ

    1. Разбиение на предложения
    2. Группировка предложений до max_token_size
    3. Сохранение целостности семантических блоков
    """
    import nltk

    # Разбиение на предложения
    sentences = nltk.sent_tokenize(content)

    chunks = []
    current_chunk = []
    current_tokens = 0

    for sentence in sentences:
        sentence_tokens = len(tokenizer.encode(sentence))

        if current_tokens + sentence_tokens > max_token_size:
            # Завершаем текущий chunk
            if current_chunk:
                chunks.append({
                    "content": " ".join(current_chunk),
                    "tokens": current_tokens,
                    "chunk_order_index": len(chunks)
                })

            # Начинаем новый chunk
            current_chunk = [sentence]
            current_tokens = sentence_tokens
        else:
            current_chunk.append(sentence)
            current_tokens += sentence_tokens

    # Добавляем последний chunk
    if current_chunk:
        chunks.append({
            "content": " ".join(current_chunk),
            "tokens": current_tokens,
            "chunk_order_index": len(chunks)
        })

    return chunks
```

### 2. Hierarchical Chunking

**Концепция**: создание chunks разных уровней детализации

```python
def hierarchical_chunking(
    content: str,
    tokenizer: Tokenizer
) -> dict[str, list[dict]]:
    """
    Многоуровневое chunking

    Levels:
    - paragraph: ~256 tokens (детальный контекст)
    - section: ~1024 tokens (основной уровень)
    - document: ~4096 tokens (широкий контекст)
    """
    return {
        "paragraph": chunking_by_token_size(
            tokenizer, content, max_token_size=256
        ),
        "section": chunking_by_token_size(
            tokenizer, content, max_token_size=1024
        ),
        "document": chunking_by_token_size(
            tokenizer, content, max_token_size=4096
        )
    }
```

### 3. Content-Aware Chunking

**Концепция**: адаптация стратегии к типу контента

```python
def content_aware_chunking(
    content: str,
    tokenizer: Tokenizer,
    content_type: str
) -> list[dict]:
    """
    Chunking адаптированный к типу контента
    """
    strategies = {
        "code": {
            "split_by": "\ndef ",
            "max_size": 512,
            "overlap": 64
        },
        "academic": {
            "split_by": "\n\n",
            "max_size": 1024,
            "overlap": 128
        },
        "legal": {
            "split_by": "\n\n[0-9]+\.",
            "max_size": 2048,
            "overlap": 256
        },
        "dialogue": {
            "split_by": "\n\n",
            "max_size": 512,
            "overlap": 64
        }
    }

    config = strategies.get(content_type, {
        "split_by": None,
        "max_size": 1024,
        "overlap": 128
    })

    return chunking_by_token_size(
        tokenizer=tokenizer,
        content=content,
        split_by_character=config["split_by"],
        max_token_size=config["max_size"],
        overlap_token_size=config["overlap"]
    )
```

## Best Practices

### ✅ DO

1. **Используйте overlap** для сохранения контекста между chunks
2. **Предпочитайте character-based splitting** для структурированных документов
3. **Настраивайте размер chunk** под конкретный домен
4. **Сохраняйте метаданные** (order, source, file path) для каждого chunk
5. **Тестируйте разные стратегии** на вашем корпусе документов

### ❌ DON'T

1. **Не используйте слишком маленькие chunks** (<256 tokens) - потеря контекста
2. **Не используйте слишком большие chunks** (>2048 tokens) - перегрузка LLM
3. **Не игнорируйте overlap** для длинных документов
4. **Не используйте arbitrary boundaries** - нарушение семантики
5. **Не забывайте про performance** - кешируйте токенизацию

## Метрики Качества Chunking

### 1. Context Preservation Score

```python
def context_preservation_score(chunks: list[dict]) -> float:
    """
    Оценка сохранения контекста между chunks

    Метрика: average overlap content similarity
    """
    scores = []
    for i in range(len(chunks) - 1):
        overlap = compute_overlap_similarity(
            chunks[i]["content"],
            chunks[i + 1]["content"]
        )
        scores.append(overlap)

    return sum(scores) / len(scores) if scores else 0.0
```

### 2. Semantic Boundary Score

```python
def semantic_boundary_score(chunks: list[dict]) -> float:
    """
    Оценка качества семантических границ

    Проверяет завершенность предложений на границах
    """
    complete_boundaries = 0
    for chunk in chunks:
        content = chunk["content"]
        # Проверка завершения предложения
        if content.rstrip().endswith(('.', '!', '?', '\n')):
            complete_boundaries += 1

    return complete_boundaries / len(chunks)
```

## Следующий Этап

После chunking документы готовы к **[Entity Extraction](03-entity-extraction.md)** - процессу извлечения сущностей и отношений с помощью языковых моделей.
