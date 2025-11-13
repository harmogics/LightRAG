# Agent Pattern (LLM Agents)

## Overview

**Pattern Type**: Behavioral  
**Category**: Semantic Agent Pattern  
**Purpose**: Инкапсуляция LLM behavior для специфичных semantic tasks

---

## Concept

LLM Agents = Specialized semantic processors with:
- Defined **role** (system prompt)
- Specific **task** (user prompt)
- Consistent **interface** (input → LLM → output)
- **Stateful** behavior (multi-turn capable)

---

## Agent Types in LightRAG

### 1. Entity Extraction Agent

**Role**: Knowledge Graph Specialist  
**Task**: Extract entities + relationships from text  
**Input**: Text chunk  
**Output**: Structured entities/relations list  

**Implementation**:
```python
class EntityExtractionAgent:
    """Agent for extracting entities from chunks"""

    def __init__(self, system_prompt, user_prompt_template):
        self.system_prompt = system_prompt  # P1
        self.user_prompt_template = user_prompt_template  # P2

    async def extract(
        self,
        chunk_content: str,
        llm_func: callable,
        **kwargs
    ) -> tuple[list[dict], list[dict]]:
        """
        Extract entities and relations from chunk

        Returns:
            (entities, relations)
        """

        # Format prompts
        user_prompt = self.user_prompt_template.format(
            input_text=chunk_content,
            **kwargs
        )

        # LLM call
        response = await llm_func(
            user_prompt,
            system_prompt=self.system_prompt,
            temperature=0.0  # Deterministic
        )

        # Parse output
        entities, relations = self._parse_output(response)

        return entities, relations
```

**Usage in Pipeline**:
```python
# T2: Entity Extraction
agent = EntityExtractionAgent(
    system_prompt=PROMPTS["entity_extraction_system_prompt"],
    user_prompt_template=PROMPTS["entity_extraction_user_prompt"]
)

for chunk in chunks:
    entities, relations = await agent.extract(
        chunk["content"],
        llm_func=llm_model_func
    )
```

---

### 2. Gleaning Agent

**Role**: Knowledge Graph Specialist (refinement)  
**Task**: Find missed entities in previous extraction  
**Input**: Previous extraction + original text  
**Output**: Additional entities/relations  

**Multi-Turn Capability**:
```python
class GleaningAgent:
    """Multi-turn agent for finding missed entities"""

    async def glean(
        self,
        chunk_content: str,
        initial_extraction: str,
        llm_func: callable,
        max_rounds: int = 2
    ) -> list[dict]:
        """
        Iterative gleaning with conversation history
        """

        history = []
        all_results = []

        # Round 1: Initial extraction (already done, part of history)
        history.append({
            "role": "user",
            "content": f"Extract entities from: {chunk_content}"
        })
        history.append({
            "role": "assistant",
            "content": initial_extraction
        })

        # Rounds 2+: Gleaning
        for round_num in range(max_rounds):
            gleaning_prompt = self._format_gleaning_prompt()

            # Multi-turn LLM call with history
            response = await llm_func(
                gleaning_prompt,
                system_prompt=self.system_prompt,
                history_messages=history
            )

            # Parse gleaned entities
            new_entities = self._parse_output(response)

            if not new_entities:
                break  # No more to glean

            all_results.extend(new_entities)

            # Update history for next round
            history.append({"role": "user", "content": gleaning_prompt})
            history.append({"role": "assistant", "content": response})

        return all_results
```

**Stateful Behavior**:
- Maintains conversation history
- Self-correction mechanism
- Iterative refinement

---

### 3. Summarization Agent

**Role**: Data Curator and Synthesizer  
**Task**: Merge multiple descriptions into coherent summary  
**Input**: List of descriptions  
**Output**: Unified summary  

**Map-Reduce Strategy**:
```python
class SummarizationAgent:
    """Agent for merging entity descriptions"""

    async def summarize(
        self,
        entity_name: str,
        descriptions: list[str],
        llm_func: callable,
        max_tokens: int = 500
    ) -> str:
        """
        Summarize descriptions using map-reduce
        """

        total_tokens = self._count_tokens(descriptions)

        if total_tokens <= self.max_context_size:
            # Direct summarization
            return await self._summarize_direct(
                entity_name,
                descriptions,
                llm_func
            )
        else:
            # Map-reduce: split, summarize chunks, recurse
            chunks = self._split_descriptions(descriptions)

            # Map: Summarize each chunk
            summaries = [
                await self._summarize_direct(entity_name, chunk, llm_func)
                for chunk in chunks
            ]

            # Reduce: Recursively summarize summaries
            return await self.summarize(
                entity_name,
                summaries,
                llm_func,
                max_tokens
            )

    async def _summarize_direct(
        self,
        entity_name: str,
        descriptions: list[str],
        llm_func: callable
    ) -> str:
        """Single LLM call for summarization"""

        prompt = self._format_prompt(entity_name, descriptions)

        summary = await llm_func(
            "",  # No user prompt, all in system
            system_prompt=prompt,
            temperature=0.1  # Slight creativity for coherence
        )

        return summary
```

---

### 4. Query Agent (Answer Generation)

**Role**: Expert AI Assistant with Retrieval Augmentation  
**Task**: Generate grounded answers from context  
**Input**: Query + Context (KG + chunks)  
**Output**: Answer + citations  

**Grounded Generation**:
```python
class QueryAgent:
    """Agent for answer generation with grounding"""

    async def generate_answer(
        self,
        query: str,
        context: dict,  # Entities, relations, chunks
        llm_func: callable,
        mode: str = "local"
    ) -> dict:
        """
        Generate grounded answer from context
        """

        # Select prompt based on mode
        if mode == "naive":
            system_prompt = PROMPTS["naive_rag_response"]
        else:
            system_prompt = PROMPTS["rag_response"]

        # Format context
        formatted_context = self._format_context(context)

        # Grounded generation with low temperature
        answer = await llm_func(
            query,
            system_prompt=system_prompt.format(
                context_data=formatted_context
            ),
            temperature=0.1  # Low creativity, high grounding
        )

        # Extract citations
        citations = self._extract_citations(answer, context)

        return {
            "answer": answer,
            "citations": citations,
            "context": context
        }
```

---

## Agent Pattern Structure

```python
class BaseAgent:
    """Base class for LLM agents"""

    def __init__(
        self,
        role_description: str,
        system_prompt_template: str,
        temperature: float = 0.0
    ):
        self.role_description = role_description
        self.system_prompt_template = system_prompt_template
        self.temperature = temperature

    async def execute(
        self,
        input_data: Any,
        llm_func: callable,
        **kwargs
    ) -> Any:
        """
        Execute agent task

        Args:
            input_data: Agent-specific input
            llm_func: LLM function to use
            **kwargs: Additional parameters

        Returns:
            Agent-specific output
        """
        raise NotImplementedError()

    def _format_prompt(self, **kwargs) -> str:
        """Format prompt with input data"""
        return self.system_prompt_template.format(**kwargs)

    def _parse_output(self, response: str) -> Any:
        """Parse LLM response into structured format"""
        raise NotImplementedError()
```

---

## Agent Characteristics

### 1. Role-Based Behavior

Each agent has clear role:
```python
agent_roles = {
    "ExtractionAgent": "Knowledge Graph Specialist",
    "GleaningAgent": "Knowledge Graph Specialist (Refinement)",
    "SummarizationAgent": "Data Curator and Synthesizer",
    "QueryAgent": "Expert AI Assistant with RAG"
}
```

### 2. Task Specialization

```python
agent_tasks = {
    "ExtractionAgent": "Extract entities and relationships",
    "GleaningAgent": "Find missed or incorrectly formatted entities",
    "SummarizationAgent": "Synthesize multiple descriptions",
    "QueryAgent": "Generate comprehensive answers from context"
}
```

### 3. Consistent Interface

All agents follow pattern:
```python
input → format_prompt → llm_call → parse_output → return_result
```

### 4. Stateful/Stateless

```python
agent_statefulness = {
    "ExtractionAgent": "Stateless (single-turn)",
    "GleaningAgent": "Stateful (multi-turn, maintains history)",
    "SummarizationAgent": "Stateless (or map-reduce state)",
    "QueryAgent": "Stateful (conversation history support)"
}
```

---

## Benefits

### 1. Encapsulation

Agent encapsulates:
- Prompt engineering
- LLM interaction
- Output parsing
- Error handling

### 2. Reusability

```python
# Same agent, different inputs
extraction_agent = EntityExtractionAgent(...)

entities1 = await extraction_agent.extract(chunk1, llm_func)
entities2 = await extraction_agent.extract(chunk2, llm_func)
entities3 = await extraction_agent.extract(chunk3, llm_func)
```

### 3. Testability

```python
# Mock LLM for testing
class MockLLM:
    async def __call__(self, prompt, **kwargs):
        return "entity<|#|>Test<|#|>person<|#|>Test entity"

@pytest.mark.asyncio
async def test_extraction_agent():
    agent = EntityExtractionAgent(...)
    mock_llm = MockLLM()

    entities, _ = await agent.extract("Test text", mock_llm)

    assert len(entities) > 0
    assert entities[0]["name"] == "Test"
```

### 4. Composability

```python
# Chain agents
extraction_agent = EntityExtractionAgent(...)
gleaning_agent = GleaningAgent(...)

# Extract
entities1 = await extraction_agent.extract(chunk, llm)

# Glean (refine)
entities2 = await gleaning_agent.glean(chunk, entities1, llm)

# Merge
all_entities = entities1 + entities2
```

---

## Agent Coordination

### Sequential Execution

```python
# T2 → T3 → T5 chain
chunk_content = "..."

# Agent 1: Extract
entities_raw = await extraction_agent.extract(chunk_content, llm)

# Agent 2: Glean
entities_refined = await gleaning_agent.glean(
    chunk_content,
    entities_raw,
    llm
)

# Agent 3: Summarize (later, after merging from multiple chunks)
entity_descriptions = collect_descriptions(entities_refined)
summary = await summarization_agent.summarize(
    "EntityName",
    entity_descriptions,
    llm
)
```

### Parallel Execution (Map Phase)

```python
# Process multiple chunks in parallel
extraction_agent = EntityExtractionAgent(...)

tasks = [
    extraction_agent.extract(chunk["content"], llm)
    for chunk in chunks
]

results = await asyncio.gather(*tasks)

# Merge results (Reduce phase)
all_entities = merge_entity_results(results)
```

---

## Related Patterns

- **Strategy Pattern**: Different agents = different strategies
- **Chain of Responsibility**: Agents in sequence
- **Observer Pattern**: Monitoring agent execution

---

## See Also

- [Pipeline Pattern](05-pipeline-pattern.md) - Agents in pipelines
- [Multi-Turn Dialog](11-multi-turn-dialog.md) - Stateful agents
- [Prompts Documentation](../prompts/README.md) - Agent prompts

