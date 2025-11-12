# Document Ingestion: Прием и Валидация Документов

## Обзор

Document Ingestion Layer отвечает за прием документов, их валидацию, дедупликацию и постановку в очередь обработки. Это первый критический этап pipeline, который обеспечивает целостность данных и предотвращает дублирование.

## Архитектура

### Главная Функция: `ainsert()`

**Расположение**: `lightrag/lightrag.py:901`

```python
async def ainsert(
    self,
    input: str | list[str],
    split_by_character: str | None = None,
    split_by_character_only: bool = False,
    ids: str | list[str] | None = None,
    file_paths: str | list[str] | None = None,
    track_id: str | None = None,
) -> str
```

### Параметры

| Параметр | Тип | Описание |
|----------|-----|----------|
| `input` | `str \| list[str]` | Один документ или список документов для индексации |
| `split_by_character` | `str \| None` | Символ для разделения (например, `\n\n`, `.`, `;`) |
| `split_by_character_only` | `bool` | Если True, разделение только по символу без token-based chunking |
| `ids` | `str \| list[str] \| None` | Пользовательские ID документов (если не указаны, генерируются MD5) |
| `file_paths` | `str \| list[str] \| None` | Пути к исходным файлам для цитирования |
| `track_id` | `str \| None` | ID для отслеживания статуса обработки |

### Возвращаемое Значение
- `str`: Track ID для мониторинга процесса обработки

## Двухфазный Процесс

### PHASE 1: Enqueue Phase

**Функция**: `apipeline_enqueue_documents()` в `lightrag.py:1008`

#### Шаги обработки:

```
Input: list[str] documents
    ↓
[1. Нормализация Input]
    ↓
    • Преобразование одиночного документа в список
    • Преобразование одиночного ID в список
    • Преобразование одиночного file_path в список
    ↓
[2. Валидация]
    ↓
    • Проверка соответствия len(file_paths) == len(input)
    • Валидация формата IDs (если предоставлены)
    ↓
[3. Санитизация Текста]
    ↓
    • sanitize_text_for_encoding(text)
    • Удаление проблемных UTF-8 символов
    • Нормализация пробелов и переносов строк
    ↓
[4. Генерация Document IDs]
    ↓
    • Если ids не предоставлены:
        doc_id = compute_mdhash_id(content, prefix="doc-")
    • Если ids предоставлены:
        doc_id = provided_id
    ↓
[5. Дедупликация по Содержимому]
    ↓
    • Группировка документов по MD5 hash содержимого
    • Если несколько IDs → один hash: выбор первого ID
    • content_hash_to_ids mapping
    ↓
[6. Проверка Существующих Документов]
    ↓
    • Запрос к doc_status storage
    • Фильтрация уже обработанных документов
    • new_docs = {id: content} для новых документов
    ↓
[7. Создание Document Metadata]
    ↓
    • Для каждого нового документа:
        {
            doc_id: {
                "content": sanitized_text,
                "file_path": file_path or "",
                "created_at": timestamp
            }
        }
    ↓
[8. Сохранение в Storage]
    ↓
    • await full_docs.upsert(new_docs)
    • await doc_status.upsert({
          doc_id: {
              "status": "PENDING",
              "created_at": timestamp,
              "track_id": track_id
          }
      })
    ↓
[9. Логирование]
    ↓
    • logger.info(f"Enqueued {len(new_docs)} documents")
    • logger.info(f"Track ID: {track_id}")
    ↓
Output: track_id
```

### PHASE 2: Processing Phase

**Функция**: `apipeline_process_enqueue_documents()` в `lightrag.py:1127`

#### Шаги обработки:

```
Input: split_by_character, split_by_character_only
    ↓
[1. Получение Pending Documents]
    ↓
    • docs = await doc_status.get_docs_by_status("PENDING")
    • docs += await doc_status.get_docs_by_status("FAILED")
    ↓
[2. Валидация Согласованности]
    ↓
    • Проверка наличия документов в full_docs
    • Создание doc_id_to_content mapping
    • Валидация file_paths
    ↓
[3. Обновление Статуса]
    ↓
    • await doc_status.update({
          doc_id: {"status": "PROCESSING"}
      })
    ↓
[4. Обработка Каждого Документа]
    ↓
    • Для каждого (doc_id, content, file_path):
        ├─ [4.1] Chunking
        │   └─ chunks = chunking_by_token_size(...)
        │
        ├─ [4.2] Создание Chunk Metadata
        │   └─ для каждого chunk:
        │       chunk_id = compute_mdhash_id(content, prefix="chunk-")
        │       chunk_data = {
        │           "content": chunk["content"],
        │           "tokens": chunk["tokens"],
        │           "chunk_order_index": chunk["chunk_order_index"],
        │           "full_doc_id": doc_id,
        │           "file_path": file_path
        │       }
        │
        ├─ [4.3] Дедупликация Chunks
        │   └─ filter existing chunks from text_chunks storage
        │
        ├─ [4.4] Параллельные Операции
        │   ├─ await chunks_vdb.upsert(chunks)
        │   ├─ await text_chunks.upsert(chunks)
        │   └─ await _process_extract_entities(chunks)
        │
        └─ [4.5] Обновление Статуса
            └─ await doc_status.update({
                   doc_id: {"status": "PROCESSED"}
               })
    ↓
[5. Финализация]
    ↓
    • await _insert_done()
        ├─ await full_entities_storage.persist()
        ├─ await full_relations_storage.persist()
        └─ await all storage backends persist()
    ↓
Output: документы обработаны и проиндексированы
```

## Детали Реализации

### 1. Санитизация Текста

**Функция**: `sanitize_text_for_encoding()` в `utils.py`

```python
def sanitize_text_for_encoding(text: str) -> str:
    """
    Очистка текста от проблемных символов для безопасного кодирования

    Операции:
    1. Удаление невалидных UTF-8 байтов
    2. Замена control characters (кроме \n, \r, \t)
    3. Нормализация множественных пробелов
    4. Удаление leading/trailing whitespace
    """
    # Encode to UTF-8 and decode with 'ignore' errors
    text = text.encode('utf-8', errors='ignore').decode('utf-8')

    # Remove control characters except newline, carriage return, tab
    text = ''.join(char for char in text
                   if ord(char) >= 32 or char in '\n\r\t')

    # Normalize whitespace
    text = ' '.join(text.split())

    return text.strip()
```

### 2. Генерация Document ID

**Функция**: `compute_mdhash_id()` в `utils.py`

```python
def compute_mdhash_id(content: str, prefix: str = "") -> str:
    """
    Генерация детерминированного ID на основе содержимого

    Args:
        content: текст для хеширования
        prefix: префикс для ID (например, "doc-", "chunk-")

    Returns:
        f"{prefix}{md5_hash}"

    Пример:
        compute_mdhash_id("Hello World", "doc-")
        → "doc-b10a8db164e0754105b7a99be72e3fe5"
    """
    hash_object = hashlib.md5(content.encode('utf-8'))
    return f"{prefix}{hash_object.hexdigest()}"
```

### 3. Track ID Generation

**Функция**: `generate_track_id()` в `utils.py`

```python
def generate_track_id(prefix: str = "track") -> str:
    """
    Генерация уникального tracking ID

    Формат: {prefix}-{timestamp}-{random_suffix}

    Пример:
        generate_track_id("insert")
        → "insert-20250112143022-a3f7b9"
    """
    timestamp = datetime.now().strftime("%Y%m%d%H%M%S")
    random_suffix = ''.join(random.choices(
        string.ascii_lowercase + string.digits, k=6
    ))
    return f"{prefix}-{timestamp}-{random_suffix}"
```

### 4. Document Status Storage

**Класс**: `DocStatusStorage` в `base.py:747`

```python
class DocStatus:
    """Статусы обработки документа"""
    PENDING = "PENDING"         # Ожидает обработки
    PROCESSING = "PROCESSING"   # В процессе обработки
    PROCESSED = "PROCESSED"     # Успешно обработан
    FAILED = "FAILED"          # Обработка завершилась с ошибкой

# Структура записи статуса
{
    "doc_id": str,
    "status": DocStatus,
    "track_id": str,
    "created_at": int,
    "updated_at": int,
    "error_message": str | None,
    "retry_count": int
}
```

#### Методы DocStatusStorage:

```python
async def get_docs_by_status(
    self,
    status: str,
    track_id: str | None = None
) -> list[dict]:
    """Получить документы по статусу"""

async def get_docs_paginated(
    self,
    status: str | None = None,
    limit: int = 100,
    offset: int = 0
) -> tuple[list[dict], int]:
    """Постраничная выборка документов"""

async def update_status(
    self,
    doc_id: str,
    status: str,
    error_message: str | None = None
) -> None:
    """Обновить статус документа"""
```

## Обработка Ошибок

### 1. Validation Errors

```python
# Несоответствие количества file_paths и документов
if file_paths is not None and len(file_paths) != len(input):
    raise ValueError(
        "Number of file paths must match the number of documents"
    )

# Несоответствие количества IDs и документов
if ids is not None and len(ids) != len(input):
    raise ValueError(
        "Number of IDs must match the number of documents"
    )
```

### 2. Processing Errors

```python
try:
    # Processing logic
    await self._process_document(doc_id, content)
    await doc_status.update_status(doc_id, "PROCESSED")
except Exception as e:
    logger.error(f"Failed to process document {doc_id}: {e}")
    await doc_status.update_status(
        doc_id,
        "FAILED",
        error_message=str(e)
    )
    # Не прерываем обработку других документов
    continue
```

### 3. Retry Logic

```python
# Автоматический retry для FAILED документов
failed_docs = await doc_status.get_docs_by_status("FAILED")
for doc in failed_docs:
    if doc["retry_count"] < max_retries:
        await doc_status.update_status(doc["doc_id"], "PENDING")
        doc["retry_count"] += 1
```

## Дедупликация Стратегий

### 1. Content-Based Deduplication

```python
# Группировка документов по content hash
content_hash_to_ids = defaultdict(list)
for doc_id, content in docs.items():
    content_hash = compute_mdhash_id(content)
    content_hash_to_ids[content_hash].append(doc_id)

# Выбор одного ID для каждого уникального содержимого
unique_docs = {}
for content_hash, doc_ids in content_hash_to_ids.items():
    # Выбираем первый ID из группы
    selected_id = doc_ids[0]
    unique_docs[selected_id] = docs[selected_id]

    # Логируем дубликаты
    if len(doc_ids) > 1:
        logger.warning(
            f"Duplicate content found: {len(doc_ids)} docs "
            f"with IDs {doc_ids}, using {selected_id}"
        )
```

### 2. ID-Based Deduplication

```python
# Фильтрация уже существующих документов
existing_doc_ids = await doc_status.filter_keys(new_doc_ids)
new_docs = {
    doc_id: content
    for doc_id, content in docs.items()
    if doc_id not in existing_doc_ids
}

if len(existing_doc_ids) > 0:
    logger.info(
        f"Skipped {len(existing_doc_ids)} already indexed documents"
    )
```

## Мониторинг и Tracking

### 1. Track ID Usage

```python
# Создание track_id для batch обработки
track_id = await rag.ainsert(documents)

# Проверка статуса обработки
status = await rag.doc_status.get_docs_by_status(
    "PROCESSING",
    track_id=track_id
)

# Ожидание завершения обработки
while True:
    pending = await rag.doc_status.get_docs_by_status(
        "PENDING",
        track_id=track_id
    )
    processing = await rag.doc_status.get_docs_by_status(
        "PROCESSING",
        track_id=track_id
    )

    if not pending and not processing:
        break

    await asyncio.sleep(1)

# Получение результатов
processed = await rag.doc_status.get_docs_by_status(
    "PROCESSED",
    track_id=track_id
)
failed = await rag.doc_status.get_docs_by_status(
    "FAILED",
    track_id=track_id
)

print(f"Successfully processed: {len(processed)}")
print(f"Failed: {len(failed)}")
```

### 2. Pipeline Status

```python
# Pipeline status dictionary
pipeline_status = {
    "track_id": str,
    "total_docs": int,
    "processed_docs": int,
    "failed_docs": int,
    "current_stage": str,
    "latest_message": str,
    "history_messages": list[str],
    "start_time": float,
    "end_time": float | None
}
```

## Оптимизации Performance

### 1. Batch Processing

```python
# Обработка документов батчами для эффективности
batch_size = 100
for i in range(0, len(documents), batch_size):
    batch = documents[i:i+batch_size]
    await rag.ainsert(batch)
```

### 2. Concurrent Storage Operations

```python
# Параллельное сохранение в разные storage backends
await asyncio.gather(
    full_docs.upsert(new_docs),
    doc_status.upsert(status_updates),
    text_chunks.upsert(chunks)
)
```

### 3. Early Exit on Empty Input

```python
if not new_docs:
    logger.info("No new documents to process")
    return track_id
```

## Best Practices

### 1. Использование File Paths для Citations

```python
# Всегда предоставляйте file_paths для возможности цитирования
await rag.ainsert(
    input=documents,
    file_paths=["doc1.pdf", "doc2.pdf"],
    ids=["doc-1", "doc-2"]
)
```

### 2. Custom IDs для Известных Документов

```python
# Используйте custom IDs для версионирования документов
await rag.ainsert(
    input=updated_content,
    ids=["doc-v2"]  # Явный ID для версии 2
)
```

### 3. Мониторинг Track IDs

```python
# Сохраняйте track_id для long-running operations
track_id = await rag.ainsert(large_document_set)
# Store track_id in database or cache for later monitoring
await redis.set(f"job:{job_id}", track_id)
```

## Следующий Этап

После успешного enqueue и validation документов, система переходит к **[Chunking Strategy](02-chunking-strategy.md)** для разбиения документов на семантически значимые фрагменты.
