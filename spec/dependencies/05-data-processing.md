# Data Processing: Обработка и Валидация Данных

## pandas

**Package**: `pandas >= 2.0.0`
**Purpose**: DataFrame operations, data export

```python
import pandas as pd

# Export knowledge graph to Excel
entities_df = pd.DataFrame([
    {"entity_name": e["entity_name"], "type": e["entity_type"], ...}
    for e in entities
])

entities_df.to_excel("knowledge_graph.xlsx", sheet_name="Entities")
```

## pydantic

**Package**: `pydantic`
**Purpose**: Data validation, type safety

```python
from pydantic import BaseModel, Field

class QueryParam(BaseModel):
    mode: str = Field(default="local", pattern="^(local|global|naive|hybrid|mix)$")
    top_k: int = Field(default=10, gt=0, le=100)
    max_entity_tokens: int = Field(default=1000, gt=0)
    # Automatic validation on initialization
```

## json_repair

**Package**: `json_repair`
**Purpose**: Repair malformed JSON from LLM outputs

```python
import json_repair

# LLM may return invalid JSON
llm_response = '{"high_level_keywords": ["Apple"], "low_level": ["iPhone]}'  # Missing quote

# Standard json.loads() would fail
try:
    data = json.loads(llm_response)
except json.JSONDecodeError:
    # json_repair fixes it
    data = json_repair.loads(llm_response)
    # Successfully parsed!
```

## xlsxwriter

**Package**: `xlsxwriter >= 3.1.0`
**Purpose**: Excel file generation

```python
import xlsxwriter

workbook = xlsxwriter.Workbook("knowledge_graph.xlsx")
worksheet = workbook.add_worksheet("Entities")

# Write headers
worksheet.write_row(0, 0, ["Entity", "Type", "Description"])

# Write data
for i, entity in enumerate(entities):
    worksheet.write_row(i+1, 0, [entity["entity_name"], entity["entity_type"], entity["description"]])

workbook.close()
```

---

**Version**: 1.0
