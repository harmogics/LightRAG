# Chunking Strategies: Концептуальный Процесс Формирования Фрагментов

## Обзор

Chunking - это фундаментальный процесс разбиения документов на семантически связанные фрагменты оптимального размера для обработки языковыми моделями. Качество chunking напрямую влияет на полноту и точность извлечения сущностей, формирование графа знаний и качество последующего поиска.

**Концептуальная цель**: Максимизировать semantic coherence внутри chunks при минимизации потери контекста на границах фрагментов.

---

## Концептуальная Архитектура

### Уровни Абстракции

```
┌─────────────────────────────────────────────────────────────────┐
│                    LEVEL 1: Document Level                       │
│  Концепция: Документ как единое семантическое целое             │
│  Проблема: Превышение context window LLM (обычно 8K-128K токенов)│
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   LEVEL 2: Chunking Strategy                     │
│  Решение: Декомпозиция с сохранением семантической связности   │
│  Методы: Token-based, Character-based, Semantic-aware           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    LEVEL 3: Chunk Level                          │
│  Результат: Фрагменты размером 200-1500 токенов                │
│  Свойства: Semantic coherence, Contextual overlap               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  LEVEL 4: Processing Level                       │
│  Применение: Entity extraction, Relationship detection          │
│  Требование: Достаточный контекст для понимания сущностей      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Концептуальные Основы Chunking

### 1. Семантическая Когерентность (Semantic Coherence)

**Определение**: Степень семантической связности текста внутри chunk.

**Метрика**:
```python
semantic_coherence = measure_topic_continuity(chunk) * measure_entity_completeness(chunk)

# Идеальный chunk:
# - Содержит завершенные предложения (синтаксическая полнота)
# - Покрывает одну логическую единицу (топикальная связность)
# - Включает полный контекст упоминаемых сущностей (сущностная полнота)
```

**Примеры**:

✅ **Высокая когерентность** (Good chunk):
```
"Alice Smith joined TechCorp in 2020 as a Senior Engineer. She led the
development of the new AI platform, collaborating closely with the Product
team. Under her leadership, the project was completed ahead of schedule."

Анализ:
- Единая тема: Alice's role at TechCorp
- Полный контекст: кто (Alice), где (TechCorp), когда (2020), что (AI platform)
- Завершенность: полная история одного события
```

❌ **Низкая когерентность** (Bad chunk):
```
"development of the new AI platform, collaborating closely with the Product
team. Under her leadership, the project was completed ahead of schedule.
Meanwhile, Bob Johnson from the Marketing department announced"

Проблемы:
- Обрывается предложение о Alice
- Неясно, кто "she" (нет начала контекста)
- Резкий переход к Bob без связи
- Фрагментация сущностей
```

### 2. Контекстное Окно (Context Window)

**Концепция**: Каждый chunk существует в контексте соседних chunks.

```
Контекстная модель:

Prior Context        Current Chunk        Future Context
[...previous...]  →  [PROCESSING]  →      [...next...]
                      ↑          ↑
                      |          |
              Backward    Forward
               Context     Context

Overlap Zone:
┌─────────────┐
│   Chunk 1   │
│  [........] │
│  [....⟋⟋⟋⟋]│  ← Overlap (128 tokens)
└─────⟋⟋⟋⟋───┘
      ⟋⟋⟋⟋
    ┌─⟋⟋⟋⟋─────┐
    │ [⟋⟋⟋⟋....] │
    │ [........] │
    │   Chunk 2  │
    └────────────┘
```

**Функции Overlap**:
1. **Continuity preservation**: Сохранение нарративной связности
2. **Entity context**: Обеспечение полного контекста для entities на границах
3. **Reference resolution**: Разрешение местоимений и ссылок
4. **Redundancy buffer**: Буфер против потери критической информации

### 3. Информационная Плотность (Information Density)

**Определение**: Количество семантически значимой информации на токен.

```python
information_density = semantic_entities_per_token(chunk)

# Оптимальная плотность:
# - Слишком низкая (<0.05): много "шума", мало полезной информации
# - Слишком высокая (>0.30): перегрузка, сложность обработки
# - Оптимальная (0.10-0.20): баланс контекста и сущностей
```

**Примеры**:

📉 **Низкая плотность** (может быть OK для контекста):
```
"In the beginning, there was an idea. This idea would eventually grow and
develop over time into something much more significant than anyone could
have initially imagined or anticipated."

Entities: 0-1 (idea)
Density: ~0.01-0.02 (много общих слов, мало конкретики)
```

📈 **Высокая плотность** (challenge для LLM):
```
"Alice(CEO,TechCorp) met Bob(CTO,InnovateLab) regarding ProjectX timeline.
Sarah(PM) discussed API integration with Chen(DevLead) involving Redis,
PostgreSQL, Neo4j architecture decisions."

Entities: 10+ (people, roles, companies, projects, technologies)
Density: ~0.25-0.35 (очень плотная информация)
```

✅ **Оптимальная плотность**:
```
"Alice Smith, CEO of TechCorp, held a strategic meeting with Bob Chen,
CTO of InnovateLab. They discussed the timeline for ProjectX, a joint
initiative to develop a new AI platform. The meeting focused on technical
architecture decisions, including the choice of databases."

Entities: 6-8 (Alice, Bob, TechCorp, InnovateLab, ProjectX, AI platform)
Density: ~0.12-0.18 (хороший баланс)
```

---

## Стратегии Chunking: Концептуальный Анализ

### Strategy 1: Pure Token-Based Chunking

**Философия**: "Uniform division with mechanical precision"

**Концептуальная модель**:
```
Document → Tokenize → Fixed-size windows → Chunks

Принцип: Математически равномерное разбиение токенового потока
```

#### Процесс Принятия Решения

**Шаг 1: Анализ входных данных**
```python
Input: document_text
       ↓
Question 1: Есть ли естественные границы?
    ├─ NO → Pure token-based (эта стратегия)
    └─ YES → Рассмотреть character-based или hybrid
       ↓
Question 2: Какой размер chunk оптимален?
    Факторы:
    • LLM context window (обычно используем ~15-25% от максимума)
    • Entity extraction prompt size (~2KB)
    • Target chunk content (1024 tokens - good default)
    • Compute budget (меньше chunks = быстрее, но хуже гранулярность)
       ↓
Decision: max_token_size = 1024 (by default)
```

**Шаг 2: Определение параметров overlap**
```python
Question: Сколько overlap нужно для сохранения контекста?

Анализ trade-offs:
┌──────────────────┬─────────────────────┬──────────────────────┐
│   Overlap Size   │   Context Quality   │   Storage Cost       │
├──────────────────┼─────────────────────┼──────────────────────┤
│  0 tokens        │  Poor (boundaries)  │  Minimal (1.0x)      │
│  64 tokens       │  Fair (partial)     │  Low (1.07x)         │
│  128 tokens      │  Good (sentences)   │  Moderate (1.14x)    │
│  256 tokens      │  Excellent (para)   │  High (1.33x)        │
│  512 tokens      │  Redundant          │  Very High (2x)      │
└──────────────────┴─────────────────────┴──────────────────────┘

Эмпирическое правило:
overlap_size ≈ 10-15% of max_token_size

Обоснование:
• 128 tokens ≈ 2-3 предложения (достаточно для контекста)
• Покрывает большинство entity mentions (обычно < 100 tokens)
• Разрешает большинство local references (pronouns, demonstratives)

Decision: overlap_token_size = 128 (by default)
```

**Шаг 3: Токенизация и разбиение**

```python
def conceptual_process_pure_token_based(document: str) -> list[Chunk]:
    """
    Концептуальный алгоритм pure token-based chunking
    """

    # Phase 1: TOKENIZATION
    # Концепция: Преобразование текста в дискретные единицы
    tokens = tokenizer.encode(document)
    # tokens = [1234, 5678, 9012, ...]  # token IDs

    total_tokens = len(tokens)

    # Phase 2: WINDOW CALCULATION
    # Концепция: Определение sliding window параметров
    stride = max_token_size - overlap_token_size
    # stride = 1024 - 128 = 896 токенов между началами chunks

    # Phase 3: CHUNK GENERATION
    # Концепция: Скользящее окно по токеновому потоку
    chunks = []

    for index, start_position in enumerate(range(0, total_tokens, stride)):
        # Определение границ window
        window_start = start_position
        window_end = min(start_position + max_token_size, total_tokens)

        # Извлечение токенов для текущего chunk
        chunk_tokens = tokens[window_start:window_end]

        # Декодирование обратно в текст
        chunk_text = tokenizer.decode(chunk_tokens)

        # Создание chunk с метаданными
        chunk = {
            "tokens": len(chunk_tokens),
            "content": chunk_text.strip(),
            "chunk_order_index": index,

            # Концептуальные метаданные (не в реальной реализации)
            "_conceptual": {
                "token_range": (window_start, window_end),
                "has_complete_sentences": analyze_sentence_completeness(chunk_text),
                "overlap_with_prev": overlap_token_size if index > 0 else 0,
                "overlap_with_next": overlap_token_size if window_end < total_tokens else 0
            }
        }

        chunks.append(chunk)

    return chunks
```

#### Концептуальная Визуализация

**Пример: Документ на 3500 токенов**

```
Document Token Stream:
[════════════════════════════════════════════════] (3500 tokens)
 0        1000      2000      3000      3500

Chunking Process:

Chunk 0: Window [0:1024]
[████████████████████████████████] (1024 tokens)
 0                              1024
                                 ↑
                         Overlap начинается здесь

Chunk 1: Window [896:1920]
         [████████████████████████████████] (1024 tokens)
         896                           1920
        ↑                                ↑
     Overlap                      Overlap начинается
  с Chunk 0                          для Chunk 2
  (128 tok)

Chunk 2: Window [1792:2816]
                  [████████████████████████████████] (1024 tokens)
                  1792                          2816
                 ↑
              Overlap
           с Chunk 1

Chunk 3: Window [2688:3500]
                           [████████████████████] (812 tokens - последний)
                           2688                3500
                          ↑
                       Overlap
                    с Chunk 2

Результат: 4 chunks с контекстным перекрытием
```

#### Семантический Анализ Результата

**Что происходит с семантикой?**

```
Сценарий 1: Предложение разрывается на границе
────────────────────────────────────────────────
Original text:
"...Alice completed the project. Bob started a new initiative..."
                                ↑ chunk boundary здесь

Chunk N ending:   "...Alice completed the project. Bob star"
Chunk N+1 start:  "Bob started a new initiative involving..."
                   ↑ overlap сохраняет полное предложение

Эффект: ✅ Overlap компенсирует разрыв, оба chunks имеют полный контекст о Bob


Сценарий 2: Entity mention пересекает границу
────────────────────────────────────────────────
Original text:
"...The CEO of TechCorp, Alice Smith, announced..."
                        ↑ chunk boundary здесь

Chunk N ending:   "...The CEO of TechCorp, Alice Sm"
Chunk N+1 start:  "Alice Smith, announced the new strategy..."
                   ↑ overlap сохраняет полное имя

Эффект: ✅ Overlap включает полное имя entity в обоих chunks


Сценарий 3: Контекстная зависимость
────────────────────────────────────────────────
Original text:
"Alice leads the team. She has 10 years of experience. Her achievements..."
                       ↑ chunk boundary здесь

Chunk N ending:   "Alice leads the team. She has 10"
Chunk N+1 start:  "She has 10 years of experience. Her achievements..."
                   ↑ "She", "Her" требуют контекста

Проблема: ⚠️ Без overlap, "She" в Chunk N+1 неясна
Решение: ✅ Overlap 128 tokens включает "Alice" в оба chunks
```

#### Преимущества и Недостатки

**✅ Преимущества**:

1. **Простота и детерминизм**
   - Алгоритм полностью предсказуем
   - Одинаковый input → одинаковый output
   - Нет необходимости в сложном анализе структуры

2. **Универсальность**
   - Работает с любым типом текста
   - Не зависит от языка (при наличии токенизатора)
   - Не требует parsing или NLP

3. **Контроль размера**
   - Точный контроль над размером chunks (± несколько токенов)
   - Предсказуемый memory footprint
   - Легко оптимизировать под LLM context window

4. **Производительность**
   - O(n) сложность (линейная)
   - Минимальные вычисления (encode → slice → decode)
   - Легко параллелизуется

**❌ Недостатки**:

1. **Игнорирование структуры**
   - Может разрывать параграфы, разделы
   - Не учитывает заголовки, списки
   - Теряет document-level структуру

2. **Разрыв предложений**
   - Sentences могут быть разорваны на границах
   - Требует overlap для компенсации
   - Неестественные точки разделения

3. **Семантическая фрагментация**
   - Логические единицы (stories, arguments) могут быть разделены
   - Entities могут терять контекст
   - Relationships могут быть менее очевидны

4. **Неоптимальная информационная плотность**
   - Некоторые chunks могут быть informationally dense
   - Другие могут быть sparse
   - Нет балансировки complexity

**Когда использовать**:
- Документы без явной структуры (continuous prose)
- Когда нужна максимальная предсказуемость
- Когда производительность критична
- Когда overlap может компенсировать разрывы

---

### Strategy 2: Character-Based Chunking (Pure)

**Философия**: "Respect document structure and natural boundaries"

**Концептуальная модель**:
```
Document → Split by structure → Segments → Chunks

Принцип: Следование естественным границам документа
```

#### Процесс Принятия Решения

**Шаг 1: Анализ структуры документа**

```python
Input: document_text
       ↓
Question: Какова структура документа?

Структурные паттерны:
┌──────────────────────┬────────────────────┬──────────────────┐
│   Document Type      │   Split Character  │   Chunk Level    │
├──────────────────────┼────────────────────┼──────────────────┤
│  Essays, Articles    │   "\n\n" (para)    │   Paragraph      │
│  Scripts, Dialogue   │   "\n---\n" (scene)│   Scene          │
│  Transcripts         │   "\n\n" (speaker) │   Turn           │
│  Code documentation  │   "\n## " (heading)│   Section        │
│  Structured reports  │   "\n===\n" (div)  │   Division       │
│  Chat logs           │   "\n[" (message)  │   Message        │
└──────────────────────┴────────────────────┴──────────────────┘

Decision factors:
1. Есть ли повторяющийся разделитель?
2. Соответствуют ли сегменты семантическим единицам?
3. Подходит ли размер сегментов для обработки?
```

**Шаг 2: Выбор режима (pure vs hybrid)**

```python
Question: Используем ли character-only или hybrid режим?

Режим 1: CHARACTER-ONLY (split_by_character_only=True)
────────────────────────────────────────────────────────
Концепция: Чистое разделение по границам, без token chunking

Когда использовать:
• Сегменты уже оптимального размера (~200-1500 tokens)
• Структура важнее единообразия размера
• Каждый сегмент - это завершенная семантическая единица

Example: Email threads, chat conversations, Q&A pairs
────────────────────────────────────────────────────────

Режим 2: HYBRID (split_by_character_only=False)
────────────────────────────────────────────────────────
Концепция: Структурное + токеновое разделение

Процесс:
1. Разделить по character → структурные сегменты
2. Для каждого сегмента:
   IF length <= max_token_size:
       Использовать сегмент как chunk (сохранить структуру)
   ELSE:
       Применить token-based chunking к сегменту

Когда использовать:
• Неоднородные размеры сегментов
• Некоторые сегменты слишком большие
• Нужен баланс структуры и размера

Example: Books (chapters), Long articles (sections)
────────────────────────────────────────────────────────
```

#### Концептуальный Алгоритм (Character-Only)

```python
def conceptual_process_character_based_pure(
    document: str,
    split_character: str = "\n\n"
) -> list[Chunk]:
    """
    Концептуальный алгоритм character-only chunking

    Философия: Сохранить естественную структуру документа
    """

    # Phase 1: STRUCTURAL SEGMENTATION
    # Концепция: Разделение по семантическим границам
    raw_segments = document.split(split_character)

    # Phase 2: SEGMENT ANALYSIS
    # Концепция: Анализ каждого сегмента
    chunks = []

    for index, segment in enumerate(raw_segments):
        # Пропуск пустых сегментов
        if not segment.strip():
            continue

        # Токенизация для подсчета размера
        segment_tokens = tokenizer.encode(segment)
        token_count = len(segment_tokens)

        # Создание chunk из сегмента "as is"
        chunk = {
            "tokens": token_count,
            "content": segment.strip(),
            "chunk_order_index": index,

            # Концептуальные метаданные
            "_conceptual": {
                "source": "structural_segment",
                "boundary_type": "natural",  # Естественная граница
                "preserves_structure": True,
                "size_category": categorize_size(token_count)
                # small (<300), medium (300-1000), large (>1000)
            }
        }

        chunks.append(chunk)

    return chunks


def categorize_size(token_count: int) -> str:
    """Категоризация размера chunk для концептуального анализа"""
    if token_count < 300:
        return "small"  # Может быть слишком мало контекста
    elif token_count <= 1500:
        return "optimal"  # Хороший размер для LLM
    else:
        return "large"  # Может быть challenging для LLM
```

#### Концептуальный Алгоритм (Hybrid)

```python
def conceptual_process_character_based_hybrid(
    document: str,
    split_character: str = "\n\n",
    max_token_size: int = 1024,
    overlap_token_size: int = 128
) -> list[Chunk]:
    """
    Концептуальный алгоритм hybrid chunking

    Философия: Уважать структуру, но обеспечить оптимальный размер
    """

    # Phase 1: STRUCTURAL SEGMENTATION
    raw_segments = document.split(split_character)

    # Phase 2: ADAPTIVE CHUNKING
    # Концепция: Разная стратегия для разных размеров сегментов
    chunks = []
    chunk_index = 0

    for segment in raw_segments:
        if not segment.strip():
            continue

        segment_tokens = tokenizer.encode(segment)
        token_count = len(segment_tokens)

        # Decision: Нужно ли дальнейшее разбиение?
        if token_count <= max_token_size:
            # Case 1: Сегмент подходящего размера
            # Action: Использовать как есть (сохранить структуру)
            chunk = {
                "tokens": token_count,
                "content": segment.strip(),
                "chunk_order_index": chunk_index,
                "_conceptual": {
                    "source": "structural_segment_intact",
                    "reason": f"Within size limit ({token_count} <= {max_token_size})"
                }
            }
            chunks.append(chunk)
            chunk_index += 1

        else:
            # Case 2: Сегмент слишком большой
            # Action: Применить token-based chunking к сегменту

            # Sub-chunking with overlap
            stride = max_token_size - overlap_token_size

            for start in range(0, token_count, stride):
                end = min(start + max_token_size, token_count)
                sub_chunk_tokens = segment_tokens[start:end]
                sub_chunk_text = tokenizer.decode(sub_chunk_tokens)

                chunk = {
                    "tokens": len(sub_chunk_tokens),
                    "content": sub_chunk_text.strip(),
                    "chunk_order_index": chunk_index,
                    "_conceptual": {
                        "source": "structural_segment_subdivided",
                        "parent_segment_size": token_count,
                        "reason": f"Segment too large ({token_count} > {max_token_size})",
                        "sub_chunk_position": f"{start}-{end} of {token_count}"
                    }
                }
                chunks.append(chunk)
                chunk_index += 1

    return chunks
```

#### Концептуальная Визуализация (Hybrid Mode)

**Пример: Статья с параграфами разного размера**

```
Original Document Structure:
┌────────────────────────────────────────────────────────────┐
│ Paragraph 1: Introduction (400 tokens)                     │
├────────────────────────────────────────────────────────────┤
│ Paragraph 2: Background (600 tokens)                       │
├────────────────────────────────────────────────────────────┤
│ Paragraph 3: Main Content (2000 tokens) [TOO LARGE!]      │
├────────────────────────────────────────────────────────────┤
│ Paragraph 4: Discussion (500 tokens)                       │
├────────────────────────────────────────────────────────────┤
│ Paragraph 5: Conclusion (300 tokens)                       │
└────────────────────────────────────────────────────────────┘

Split by "\n\n" → 5 segments

Chunking Process:
─────────────────

Segment 1 (400 tokens): ✓ OK size
→ Chunk 0: [Paragraph 1 - intact] (400 tokens)
  Status: Structural boundary preserved

Segment 2 (600 tokens): ✓ OK size
→ Chunk 1: [Paragraph 2 - intact] (600 tokens)
  Status: Structural boundary preserved

Segment 3 (2000 tokens): ✗ TOO LARGE
→ Apply token-based sub-chunking:
  → Chunk 2: [Para 3, tokens 0:1024] (1024 tokens)
  → Chunk 3: [Para 3, tokens 896:1920] (1024 tokens)  ← overlap 128
  → Chunk 4: [Para 3, tokens 1792:2000] (208 tokens)
  Status: Structural boundary broken by necessity

Segment 4 (500 tokens): ✓ OK size
→ Chunk 5: [Paragraph 4 - intact] (500 tokens)
  Status: Structural boundary preserved

Segment 5 (300 tokens): ✓ OK size
→ Chunk 6: [Paragraph 5 - intact] (300 tokens)
  Status: Structural boundary preserved

Result: 7 chunks
- 5 структурно целостных (Paragraphs 1,2,4,5)
- 2 разбитых из большого параграфа (Paragraph 3)
```

#### Семантический Анализ

**Преимущества структурного подхода**:

```
Сценарий 1: Параграф как семантическая единица
────────────────────────────────────────────────

Document:
¶1: "Alice Smith was appointed CEO of TechCorp in January 2020.
     She brought extensive experience from her previous role at InnovateLab."

¶2: "Under Alice's leadership, TechCorp launched three major products.
     The company's revenue grew by 150% in two years."

Character-based (split by "\n\n"):
→ Chunk 0: [¶1 complete]
→ Chunk 1: [¶2 complete]

Семантическая целостность: ✅ ОТЛИЧНО
- Каждый chunk содержит законченную мысль
- Context complete: вся информация об Alice в первом chunk
- Natural boundaries: читатель ожидает break между параграфами


Token-based (no structure awareness):
→ Chunk 0: "Alice Smith was appointed CEO of TechCorp in January 2020.
           She brought extensive experience from her previous role at
           InnovateLab. Under Alice's leadership, TechCor"
→ Chunk 1: "TechCorp launched three major products. The company's..."

Семантическая целостность: ⚠️ СРЕДНЕ
- Разрыв посреди "TechCorp"
- Потеря контекста о "Alice's leadership" в начале Chunk 1
- Неестественная точка разделения
```

**Недостатки при неоднородных размерах**:

```
Сценарий 2: Очень короткие сегменты
────────────────────────────────────

Chat log example:
[User]: "Hi"                          (2 tokens)
[Bot]: "Hello!"                       (3 tokens)
[User]: "What's the weather?"         (6 tokens)
[Bot]: "It's sunny today."           (6 tokens)

Character-based (split by "\n"):
→ Chunk 0: "Hi" (2 tokens)           ⚠️ Слишком мало контекста
→ Chunk 1: "Hello!" (3 tokens)       ⚠️ Слишком мало
→ Chunk 2: "What's the weather?" (6 tokens) ⚠️ Слишком мало
→ Chunk 3: "It's sunny today." (6 tokens)   ⚠️ Слишком мало

Проблемы:
• Недостаточный контекст для entity extraction
• Слишком много маленьких chunks (overhead)
• LLM не видит conversation flow

Решение: Group multiple messages into chunks
```

#### Преимущества и Недостатки

**✅ Преимущества**:

1. **Структурная целостность**
   - Сохраняет natural boundaries документа
   - Respect author's intended structure
   - Семантически связные единицы

2. **Семантическая когерентность**
   - Высокая внутренняя связность
   - Полный контекст в рамках структурной единицы
   - Меньше fragmented entities

3. **Читаемость и интерпретируемость**
   - Chunks easier to understand для человека
   - Более естественные точки разделения
   - Лучше для debugging и inspection

4. **Оптимизация для structured content**
   - Идеально для well-structured documents
   - Эффективно для documents с ясной иерархией
   - Preserves document semantics

**❌ Недостатки**:

1. **Неоднородность размеров**
   - Chunks могут сильно варьироваться по размеру
   - Некоторые могут быть слишком маленькими (мало контекста)
   - Другие могут быть слишком большими (превышают LLM limit)

2. **Зависимость от качества структуры**
   - Требует well-structured input
   - Плохая структура → плохие chunks
   - Не работает для unstructured text

3. **Необходимость domain knowledge**
   - Нужно знать правильный split character
   - Разные типы документов требуют разных разделителей
   - Нет универсального решения

4. **Потенциальная потеря контекста между сегментами**
   - Нет overlap между структурными единицами (в pure mode)
   - Cross-segment relationships могут быть потеряны
   - Requires careful consideration of boundaries

**Когда использовать**:
- Well-structured documents (параграфы, секции, главы)
- Когда структура соответствует семантическим единицам
- Conversations, transcripts, scripts
- Когда важна интерпретируемость chunks

---

### Strategy 3: Hybrid Adaptive Chunking

**Философия**: "Best of both worlds - structure-aware with size guarantees"

**Концептуальная модель**:
```
Document → Identify structure → Split by structure → Validate size
         ↓                                              ↓
    No structure?                               Too large?
         ↓                                              ↓
    Token-based                              Token-based sub-chunking

Принцип: Адаптивный подход на основе характеристик документа
```

Это уже описано выше в Character-Based Hybrid режиме, но рассмотрим концептуально более глубоко.

#### Концептуальная Диаграмма Принятия Решений

```
                    ┌─────────────────┐
                    │  Input Document │
                    └────────┬────────┘
                             ↓
                    ┌─────────────────┐
                    │ Analyze Structure│
                    └────────┬────────┘
                             ↓
                    ╔════════╧════════╗
                    ║ Has Natural     ║
                    ║ Boundaries?     ║
                    ╚════════╤════════╝
                   YES ←─────┴─────→ NO
                    ↓                 ↓
          ┌──────────────────┐   ┌──────────────────┐
          │ Character-Based  │   │ Token-Based      │
          │ Initial Split    │   │ Direct Chunking  │
          └─────────┬────────┘   └──────────────────┘
                    ↓                     (Strategy 1)
          ┌──────────────────┐
          │ For Each Segment │
          └─────────┬────────┘
                    ↓
           ╔════════╧════════╗
           ║ Segment Size    ║
           ║ Acceptable?     ║
           ╚════════╤════════╝
          YES ←─────┴─────→ NO
           ↓                 ↓
  ┌────────────────┐   ┌──────────────────┐
  │ Keep Segment   │   │ Sub-chunk with   │
  │ as Chunk       │   │ Token-Based      │
  │ (preserve!)    │   │ + Overlap        │
  └────────────────┘   └──────────────────┘
           ↓                 ↓
           └────────┬────────┘
                    ↓
          ┌──────────────────┐
          │ All Chunks Ready │
          └──────────────────┘
```

#### Концептуальные Правила Размера

```python
def conceptual_size_validation(segment_tokens: int,
                               max_token_size: int) -> dict:
    """
    Концептуальная валидация размера сегмента

    Возвращает решение о необходимости sub-chunking
    """

    # Определение порогов
    min_threshold = max_token_size * 0.2   # 20% от max (например, 204)
    optimal_min = max_token_size * 0.3     # 30% от max (например, 307)
    optimal_max = max_token_size * 1.0     # 100% от max (например, 1024)
    max_threshold = max_token_size * 1.5   # 150% от max (например, 1536)

    if segment_tokens < min_threshold:
        return {
            "decision": "too_small",
            "action": "consider_merging",
            "reason": f"Segment ({segment_tokens} tokens) may lack context",
            "recommendation": "Merge with adjacent segments if semantically appropriate"
        }

    elif min_threshold <= segment_tokens < optimal_min:
        return {
            "decision": "acceptable_small",
            "action": "keep_as_is",
            "reason": f"Small but acceptable ({segment_tokens} tokens)",
            "recommendation": "Keep as separate chunk, monitor extraction quality"
        }

    elif optimal_min <= segment_tokens <= optimal_max:
        return {
            "decision": "optimal",
            "action": "keep_as_is",
            "reason": f"Perfect size ({segment_tokens} tokens)",
            "recommendation": "Use as-is, excellent for LLM processing"
        }

    elif optimal_max < segment_tokens <= max_threshold:
        return {
            "decision": "acceptable_large",
            "action": "keep_or_split",
            "reason": f"Large but manageable ({segment_tokens} tokens)",
            "recommendation": "Can keep if structure is important, or split if prefer uniformity"
        }

    else:  # segment_tokens > max_threshold
        return {
            "decision": "too_large",
            "action": "must_split",
            "reason": f"Exceeds limit ({segment_tokens} > {max_threshold} tokens)",
            "recommendation": "Apply token-based sub-chunking with overlap"
        }
```

#### Концептуальный Пример: Academic Paper

```
Academic Paper Structure:
══════════════════════════

Abstract (150 tokens)                 → Optimal size, keep as Chunk 0
Introduction (800 tokens)             → Optimal size, keep as Chunk 1
Related Work (1200 tokens)            → Large but acceptable, keep as Chunk 2
Methodology (2500 tokens)             → TOO LARGE, must split:
    → Chunk 3: Method tokens [0:1024]
    → Chunk 4: Method tokens [896:1920]  ← overlap
    → Chunk 5: Method tokens [1792:2500]
Experiments (3000 tokens)             → TOO LARGE, must split:
    → Chunk 6: Exp tokens [0:1024]
    → Chunk 7: Exp tokens [896:1920]
    → Chunk 8: Exp tokens [1792:2816]
    → Chunk 9: Exp tokens [2688:3000]
Results (600 tokens)                  → Optimal size, keep as Chunk 10
Discussion (900 tokens)               → Optimal size, keep as Chunk 11
Conclusion (200 tokens)               → Small but acceptable, keep as Chunk 12
References (1500 tokens)              → Large, consider splitting or keeping
    → Decision: Keep as Chunk 13 (references are list-like, OK to be large)

Final Chunks: 14 chunks
- 9 structure-preserving (Abstract, Intro, Related, Results, Discussion, Conclusion, Refs)
- 5 sub-chunked from large sections (Methodology: 3, Experiments: 4)

Semantic Quality: ✅ EXCELLENT
- Preserves paper structure
- All sections either intact or carefully sub-divided
- Natural reading flow maintained
```

---

## Концептуальная Оптимизация: Выбор Параметров

### Матрица Принятия Решений

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Document Characteristics                             │
├────────────────┬────────────────┬────────────────┬──────────────────┤
│  Property      │  Value         │  Recommended   │  Rationale       │
│                │                │  Strategy      │                  │
├────────────────┼────────────────┼────────────────┼──────────────────┤
│ Structure      │ Well-defined   │ Character      │ Preserve meaning │
│                │ (sections)     │ -based         │ units            │
│                ├────────────────┼────────────────┼──────────────────┤
│                │ None/unclear   │ Token-based    │ Uniform chunks   │
├────────────────┼────────────────┼────────────────┼──────────────────┤
│ Segment Size   │ Uniform        │ Character-only │ No need to split │
│  (if struct)   │ (~300-1000 tok)│                │                  │
│                ├────────────────┼────────────────┼──────────────────┤
│                │ Varied         │ Hybrid         │ Adapt per segment│
│                │ (50-5000 tok)  │                │                  │
├────────────────┼────────────────┼────────────────┼──────────────────┤
│ Info Density   │ High           │ Smaller chunks │ Easier to process│
│                │ (>0.25)        │ (512-768 tok)  │                  │
│                ├────────────────┼────────────────┼──────────────────┤
│                │ Medium         │ Standard chunks│ Good balance     │
│                │ (0.1-0.25)     │ (1024 tok)     │                  │
│                ├────────────────┼────────────────┼──────────────────┤
│                │ Low            │ Larger chunks  │ More context     │
│                │ (<0.1)         │ (1536-2048 tok)│ needed           │
├────────────────┼────────────────┼────────────────┼──────────────────┤
│ Entity Density │ High           │ More overlap   │ Preserve context │
│                │ (many entities)│ (256 tok)      │                  │
│                ├────────────────┼────────────────┼──────────────────┤
│                │ Low            │ Less overlap   │ Save storage     │
│                │ (few entities) │ (64-128 tok)   │                  │
├────────────────┼────────────────┼────────────────┼──────────────────┤
│ LLM Context    │ Large          │ Larger chunks  │ Utilize capacity │
│  Window        │ (128K+)        │ (2048+ tok)    │                  │
│                ├────────────────┼────────────────┼──────────────────┤
│                │ Standard       │ Standard chunks│ Fit prompt space │
│                │ (8K-32K)       │ (1024 tok)     │                  │
│                ├────────────────┼────────────────┼──────────────────┤
│                │ Small          │ Smaller chunks │ Leave room for   │
│                │ (<8K)          │ (512 tok)      │ prompt+response  │
└────────────────┴────────────────┴────────────────┴──────────────────┘
```

### Параметры Overlap: Концептуальный Анализ

**Формула для оптимального overlap**:

```python
def calculate_optimal_overlap(
    max_chunk_size: int,
    entity_density: float,       # entities per 100 tokens
    avg_entity_length: int,      # average tokens per entity mention
    sentence_length: int = 20    # average sentence length
) -> int:
    """
    Концептуальный расчет оптимального overlap

    Factors:
    1. Entity coverage: overlap должен покрывать typical entity mention
    2. Sentence coverage: overlap должен включать полные предложения
    3. Context sufficiency: overlap должен давать semantic context
    """

    # Factor 1: Entity-based minimum
    # Если entity_density = 0.15 (15 entities per 100 tokens)
    # И avg_entity_length = 3 tokens
    # То в chunk вероятно ~15-20 entities, некоторые близко к boundaries
    entity_based_min = avg_entity_length * 2  # Safety margin

    # Factor 2: Sentence-based minimum
    # Должно покрывать хотя бы 2-3 предложения для context
    sentence_based_min = sentence_length * 2.5

    # Factor 3: Proportion-based
    # Overlap обычно 10-15% от chunk size
    proportion_based = max_chunk_size * 0.125  # 12.5%

    # Take maximum to ensure all factors satisfied
    calculated_overlap = max(
        entity_based_min,
        sentence_based_min,
        proportion_based
    )

    # Round to reasonable value
    overlap = round(calculated_overlap / 16) * 16  # Round to 16-token increments

    # Clamp to reasonable bounds
    min_overlap = 64   # Absolute minimum
    max_overlap = max_chunk_size // 3  # Don't exceed 33% of chunk

    return max(min_overlap, min(overlap, max_overlap))


# Examples:
# ─────────
# Standard document (max_chunk=1024, entity_density=0.12, avg_entity=3, sentence=20):
#   entity_min = 6, sentence_min = 50, proportion = 128
#   → optimal_overlap = 128 ✓

# Entity-heavy document (max_chunk=1024, entity_density=0.30, avg_entity=4, sentence=25):
#   entity_min = 8, sentence_min = 62.5, proportion = 128
#   → optimal_overlap = 128 ✓

# Long-sentence literature (max_chunk=1024, entity_density=0.08, avg_entity=4, sentence=40):
#   entity_min = 8, sentence_min = 100, proportion = 128
#   → optimal_overlap = 128 ✓ (proportion wins)

# Large chunks (max_chunk=2048, entity_density=0.12, avg_entity=3, sentence=20):
#   entity_min = 6, sentence_min = 50, proportion = 256
#   → optimal_overlap = 256 ✓ (scales with chunk size)
```

---

## Влияние Chunking на Downstream Процессы

### Entity Extraction Quality

**Концептуальная зависимость**:

```
Entity Extraction Quality = f(chunk_coherence, chunk_size, context_completeness)

Факторы влияния:
────────────────

1. Chunk Coherence (Semantic Unity)
   ↓
   High coherence → Entities в related context → Better extraction
   Low coherence → Entities без связи → Worse extraction

   Example:
   ✅ Good: "Alice (CEO of TechCorp) launched ProductX in 2020."
       → Clear relationships: Alice-CEO, Alice-TechCorp, Alice-ProductX

   ❌ Bad: "...CEO of TechCorp) launched Produc"
       → Incomplete entities, broken context

2. Chunk Size (Information Availability)
   ↓
   Too small (<200 tok) → Insufficient context → Low recall
   Optimal (500-1500 tok) → Good balance → Best quality
   Too large (>2000 tok) → Information overload → Processing issues

   Empirical curve:

   Quality
     │
   1.0├─────────────╭──────────────╮─────────
      │           ╱                  ╲
   0.8├─────────╱                      ╲─────
      │       ╱                          ╲
   0.6├─────╱                              ╲─
      │   ╱                                  ╲
   0.4├─╱
      │╱
   0.2├──────────────────────────────────────
      └───────────────────────────────────────→ Size
          200    500   1000  1500  2000   2500

3. Context Completeness (Entity Boundaries)
   ↓
   Complete mentions → High precision
   Truncated mentions → Low precision + hallucinations

   Example:
   ✅ Complete: "Dr. Alice Smith" → Correct entity
   ❌ Truncated: "Dr. Alice Sm" → May hallucinate "Dr. Alice Smith" or miss
```

### Graph Construction Quality

**Impact of chunking on knowledge graph**:

```
Graph Quality Metrics:
──────────────────────

1. Entity Coverage (Recall)
   = (Entities found) / (Entities in document)

   Influenced by:
   • Chunk boundaries cutting entity mentions
   • Overlap covering boundary entities
   • Chunk size providing sufficient context

   Example scenario:
   Document: 100 unique entities

   Poor chunking (no overlap, bad boundaries):
   → Found: 75 entities (25 lost at boundaries)
   → Coverage = 75%

   Good chunking (128-token overlap, structure-aware):
   → Found: 95 entities (5 missed due to ambiguity)
   → Coverage = 95%

2. Relationship Accuracy (Precision)
   = (Correct relationships) / (All extracted relationships)

   Influenced by:
   • Whether related entities are in same chunk
   • Context sufficiency for relationship inference
   • Chunk coherence supporting relationship understanding

   Example scenario:
   Relationship: Alice-worksAt-TechCorp

   ✅ Both entities in same chunk:
      "Alice Smith joined TechCorp as CEO in 2020..."
      → Relationship clearly stated → High precision

   ⚠️ Entities in different chunks:
      Chunk 1: "Alice Smith was promoted..."
      Chunk 2: "TechCorp announced new leadership..."
      → Relationship unclear → May be missed or incorrect

   ❌ Entity context broken:
      Chunk 1: "Alice Smith works at Tech"
      Chunk 2: "Corp as CEO..."
      → Cannot extract relationship → Lost

3. Description Quality
   = Semantic richness of entity descriptions

   Influenced by:
   • How much context about entity is in chunk
   • Whether entity's full context is preserved
   • Chunk structure preserving entity narratives

   Example:

   Full context in one chunk:
   "Alice Smith, CEO of TechCorp, has 15 years of experience in AI.
    She previously led research at MIT and holds a PhD in Computer Science."
   → Rich description: role, experience, background, education

   Fragmented across chunks:
   Chunk 1: "Alice Smith, CEO of..."
   Chunk 2: "...experience in AI. She previously..."
   Chunk 3: "...led research at MIT..."
   → Description merging needed, may lose details
```

### Query Performance

**Impact on search and retrieval**:

```
Query Performance = f(chunk_granularity, chunk_overlap, chunk_coherence)

Scenarios:
──────────

Scenario 1: Naive Search (chunk-level vector search)
─────────────────────────────────────────────────────
Query: "What did Alice accomplish at TechCorp?"

Good chunking (coherent chunks):
Chunk X: "Alice Smith joined TechCorp as CEO. She led development of
          three major products, growing revenue by 150%."
→ Vector similarity high → Relevant chunk retrieved → Good answer ✅

Poor chunking (fragmented):
Chunk Y: "...as CEO. She led development of thr"
Chunk Z: "ee major products, growing revenue..."
→ Neither chunk fully answers query
→ Lower similarity scores
→ May need to retrieve both (more complexity) ⚠️

Scenario 2: Local Search (entity + neighborhood)
─────────────────────────────────────────────────
Query: "Describe Alice's relationships"

Good chunking (entities well extracted):
→ "Alice" entity found with relationships:
   - worksAt: TechCorp
   - leads: AI Platform Team
   - reportsTo: Board of Directors
→ Rich relationship graph → Comprehensive answer ✅

Poor chunking (relationships missed):
→ "Alice" entity found but sparse relationships:
   - worksAt: TechCorp (only one found)
→ Other relationships lost due to chunking → Incomplete answer ⚠️

Scenario 3: Global Search (community detection)
────────────────────────────────────────────────
Query: "What are the main themes in the document?"

Good chunking (semantic coherence):
→ Chunks align with topics
→ Communities detected: Leadership, Technology, Growth
→ Clear thematic structure → Coherent summary ✅

Poor chunking (arbitrary splits):
→ Topics scattered across chunks
→ Community detection noisy
→ Unclear thematic structure → Confused summary ⚠️
```

---

## Рекомендации по Выбору Стратегии

### Decision Tree

```
START: Analyze your document corpus
│
├─ Question 1: Do documents have clear structure?
│  │
│  ├─ YES: Do structural units have uniform size?
│  │  │
│  │  ├─ YES: Use CHARACTER-ONLY chunking
│  │  │      • Best semantic coherence
│  │  │      • Preserves document structure
│  │  │      • Example: Chat logs, Q&A pairs, scripts
│  │  │
│  │  └─ NO: Use HYBRID chunking
│  │         • Respects structure where possible
│  │         • Ensures size constraints
│  │         • Example: Academic papers, reports, books
│  │
│  └─ NO: Use TOKEN-BASED chunking
│     │    • Uniform chunk sizes
│     │    • Predictable processing
│     │    • Example: Continuous prose, novels, essays
│     │
│     └─ Consider: Can you create synthetic structure?
│            • Paragraph detection
│            • Sentence boundary detection
│            • Then use hybrid approach
│
└─ Question 2: What is your primary optimization target?
   │
   ├─ Precision (accurate entities)
   │  → Smaller chunks (512-768 tokens)
   │  → More overlap (192-256 tokens)
   │  → Structure-aware splitting
   │
   ├─ Recall (complete coverage)
   │  → Larger chunks (1024-1536 tokens)
   │  → More overlap (192-256 tokens)
   │  → Ensure entity contexts preserved
   │
   ├─ Speed (fast processing)
   │  → Standard chunks (1024 tokens)
   │  → Less overlap (64-96 tokens)
   │  → Token-based (simpler)
   │
   └─ Balance
      → Standard chunks (1024 tokens)
      → Standard overlap (128 tokens)
      → Hybrid approach
```

### Configuration Recommendations by Document Type

```python
# Configurations for different document types

CHUNKING_CONFIGS = {
    "scientific_papers": {
        "strategy": "hybrid",
        "split_by_character": "\n## ",  # Split by sections
        "max_token_size": 1200,         # Larger for detailed content
        "overlap_token_size": 192,      # More overlap for complex terms
        "rationale": "Papers have clear sections; preserve structure "
                     "but ensure no section exceeds LLM capacity"
    },

    "news_articles": {
        "strategy": "character",
        "split_by_character": "\n\n",   # Paragraphs
        "split_by_character_only": True,
        "rationale": "Paragraphs are semantic units; usually well-sized"
    },

    "novels": {
        "strategy": "token",
        "max_token_size": 1536,         # Larger for narrative flow
        "overlap_token_size": 256,      # More overlap for story continuity
        "rationale": "Continuous prose without clear structure; "
                     "larger chunks preserve narrative context"
    },

    "technical_documentation": {
        "strategy": "hybrid",
        "split_by_character": "\n### ",  # Split by subsections
        "max_token_size": 1024,
        "overlap_token_size": 128,
        "rationale": "Hierarchical structure; respect it but ensure size limits"
    },

    "chat_conversations": {
        "strategy": "character",
        "split_by_character": "\n---\n",  # Conversation turns
        "split_by_character_only": False,  # Some turns may be long
        "max_token_size": 800,            # Smaller for focused context
        "overlap_token_size": 96,         # Less overlap (turns are self-contained)
        "rationale": "Conversations have natural turn boundaries; "
                     "preserve speaker context"
    },

    "legal_documents": {
        "strategy": "hybrid",
        "split_by_character": "\n\n",    # Clauses/paragraphs
        "max_token_size": 1536,          # Larger for complete clauses
        "overlap_token_size": 256,       # More overlap for references
        "rationale": "Legal text requires complete context; "
                     "clauses reference each other frequently"
    },

    "social_media": {
        "strategy": "token",
        "max_token_size": 512,           # Smaller for short posts
        "overlap_token_size": 64,        # Minimal overlap
        "rationale": "Short, informal text; no structure to preserve; "
                     "smaller chunks match content granularity"
    },

    "email_threads": {
        "strategy": "character",
        "split_by_character": "\n\nFrom:",  # Email boundaries
        "split_by_character_only": False,
        "max_token_size": 1024,
        "overlap_token_size": 128,
        "rationale": "Each email is a semantic unit; thread context important"
    },

    "code_documentation": {
        "strategy": "hybrid",
        "split_by_character": "\n## ",   # Markdown sections
        "max_token_size": 1200,
        "overlap_token_size": 192,       # More overlap for API references
        "rationale": "Clear sections; code examples may be long; "
                     "cross-references common"
    },
}
```

---

## Метрики Качества Chunking

### Evaluation Framework

```python
def evaluate_chunking_quality(chunks: list[Chunk],
                              original_document: str) -> dict:
    """
    Концептуальная оценка качества chunking

    Returns comprehensive quality metrics
    """

    metrics = {}

    # 1. SIZE UNIFORMITY
    # Насколько uniform размеры chunks?
    chunk_sizes = [chunk["tokens"] for chunk in chunks]
    metrics["size_uniformity"] = {
        "mean": np.mean(chunk_sizes),
        "std": np.std(chunk_sizes),
        "cv": np.std(chunk_sizes) / np.mean(chunk_sizes),  # Coefficient of variation
        "min": min(chunk_sizes),
        "max": max(chunk_sizes),
        "assessment": "uniform" if np.std(chunk_sizes) / np.mean(chunk_sizes) < 0.3 else "varied"
    }

    # 2. SEMANTIC COHERENCE
    # Насколько связны chunks внутри?
    coherence_scores = []
    for chunk in chunks:
        # Measure topic continuity within chunk
        sentences = split_sentences(chunk["content"])
        if len(sentences) > 1:
            # Compute semantic similarity between consecutive sentences
            similarities = [
                cosine_similarity(embed(s1), embed(s2))
                for s1, s2 in zip(sentences[:-1], sentences[1:])
            ]
            coherence = np.mean(similarities)
            coherence_scores.append(coherence)

    metrics["semantic_coherence"] = {
        "mean": np.mean(coherence_scores),
        "assessment": "high" if np.mean(coherence_scores) > 0.7 else
                     "medium" if np.mean(coherence_scores) > 0.5 else "low"
    }

    # 3. BOUNDARY QUALITY
    # Насколько естественны границы chunks?
    boundary_quality = []
    for i in range(len(chunks) - 1):
        chunk_end = chunks[i]["content"][-100:]  # Last 100 chars
        chunk_start = chunks[i+1]["content"][:100]  # First 100 chars

        # Check if boundary is at sentence end
        ends_with_sentence = chunk_end.rstrip().endswith(('.', '!', '?'))
        starts_with_capital = chunk_start.lstrip()[0].isupper() if chunk_start.strip() else False

        # Check if boundary breaks entity mention
        breaks_entity = detect_broken_entity(chunk_end, chunk_start)

        score = 1.0
        if not ends_with_sentence:
            score -= 0.3
        if not starts_with_capital:
            score -= 0.2
        if breaks_entity:
            score -= 0.5

        boundary_quality.append(max(0, score))

    metrics["boundary_quality"] = {
        "mean": np.mean(boundary_quality),
        "assessment": "good" if np.mean(boundary_quality) > 0.7 else
                     "fair" if np.mean(boundary_quality) > 0.5 else "poor"
    }

    # 4. COVERAGE EFFICIENCY
    # Насколько эффективно покрывается документ?
    total_chunk_tokens = sum(chunk["tokens"] for chunk in chunks)
    doc_tokens = len(tokenizer.encode(original_document))

    overlap_ratio = (total_chunk_tokens - doc_tokens) / doc_tokens

    metrics["coverage_efficiency"] = {
        "document_tokens": doc_tokens,
        "total_chunk_tokens": total_chunk_tokens,
        "overlap_ratio": overlap_ratio,
        "redundancy": f"{overlap_ratio*100:.1f}%",
        "assessment": "efficient" if 0.1 <= overlap_ratio <= 0.2 else
                     "acceptable" if overlap_ratio < 0.4 else "redundant"
    }

    # 5. ENTITY PRESERVATION
    # Сохранены ли entity mentions полностью?
    doc_entities = extract_entities_simple(original_document)

    preserved_entities = 0
    for entity in doc_entities:
        # Check if entity appears completely in at least one chunk
        for chunk in chunks:
            if entity in chunk["content"]:
                preserved_entities += 1
                break

    metrics["entity_preservation"] = {
        "total_entities": len(doc_entities),
        "preserved": preserved_entities,
        "preservation_rate": preserved_entities / len(doc_entities) if doc_entities else 1.0,
        "assessment": "excellent" if preservation_rate > 0.95 else
                     "good" if preservation_rate > 0.85 else "needs improvement"
    }

    # 6. OVERALL QUALITY SCORE
    quality_score = (
        metrics["semantic_coherence"]["mean"] * 0.3 +
        metrics["boundary_quality"]["mean"] * 0.3 +
        (1 - abs(metrics["coverage_efficiency"]["overlap_ratio"] - 0.125) / 0.125) * 0.2 +
        metrics["entity_preservation"]["preservation_rate"] * 0.2
    )

    metrics["overall_quality"] = {
        "score": quality_score,
        "grade": "A" if quality_score > 0.85 else
                "B" if quality_score > 0.70 else
                "C" if quality_score > 0.55 else
                "D" if quality_score > 0.40 else "F"
    }

    return metrics
```

---

## Заключение

### Ключевые Принципы

1. **Semantic Coherence > Uniform Size**
   - Предпочитайте семантическую связность единообразию размера
   - Используйте overlap для компенсации boundary effects
   - Respect natural structure когда возможно

2. **Context is King**
   - Достаточный overlap критичен для entity extraction
   - Chunks должны быть self-contained semantic units
   - Cross-chunk references сложны для LLM

3. **One Size Doesn't Fit All**
   - Разные типы документов требуют разных стратегий
   - Адаптируйте параметры под ваш corpus
   - Мониторьте downstream quality metrics

4. **Balance Trade-offs**
   - Size vs Coherence
   - Overlap vs Storage
   - Speed vs Quality
   - Precision vs Recall

### Практические Рекомендации

```
Default Configuration (good starting point):
─────────────────────────────────────────────
• Strategy: Hybrid
• max_token_size: 1024
• overlap_token_size: 128
• split_by_character: "\n\n" (if structured)

Then optimize based on:
─────────────────────
1. Monitor entity extraction quality
2. Analyze boundary breaks
3. Evaluate semantic coherence
4. Measure downstream task performance

Iterate:
────────
↻ Adjust parameters
↻ Try different strategies
↻ A/B test configurations
↻ Converge on optimal setup
```

### Будущие Направления

- **Semantic-Aware Chunking**: Использование NLP для определения semantic boundaries
- **Adaptive Chunking**: Динамическая адаптация размера под information density
- **Multi-Level Chunking**: Иерархические chunks (document → section → paragraph → sentence)
- **Query-Aware Chunking**: Оптимизация chunks под типичные query patterns

---

**Файлы для дополнительной информации**:
- [02-chunking-strategy.md](02-chunking-strategy.md) - Базовое описание и визуализации
- [03-entity-extraction.md](03-entity-extraction.md) - Как chunking влияет на extraction
- [transform/01-indexing-transforms.md](transform/01-indexing-transforms.md) - T1 transformation детали
