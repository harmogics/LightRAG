# Utilities: Вспомогательные Библиотеки

## python-dotenv

**Package**: `python-dotenv`
**Purpose**: Load environment variables from .env files

```python
from dotenv import load_dotenv
import os

load_dotenv(dotenv_path=".env", override=False)

api_key = os.getenv("OPENAI_API_KEY")
```

## configparser

**Package**: `configparser`
**Purpose**: INI configuration files

```python
import configparser

config = configparser.ConfigParser()
config.read("config.ini")

host = config["database"]["host"]
```

## pypinyin

**Package**: `pypinyin`
**Purpose**: Chinese pinyin conversion (optional)

```python
try:
    import pypinyin

    # Sort Chinese entities by pinyin
    entities_sorted = sorted(
        entities,
        key=lambda e: pypinyin.lazy_pinyin(e["entity_name"])
    )
except ImportError:
    # Fallback to regular sorting
    entities_sorted = sorted(entities, key=lambda e: e["entity_name"])
```

## psutil

**Package**: `psutil`
**Purpose**: System monitoring

```python
import psutil

# Check memory usage
memory = psutil.virtual_memory()
print(f"Memory: {memory.percent}% used")

# Check CPU
cpu_percent = psutil.cpu_percent(interval=1)
```

## pytz

**Package**: `pytz`
**Purpose**: Timezone handling

```python
from datetime import datetime
import pytz

utc_time = datetime.now(pytz.UTC)
local_time = utc_time.astimezone(pytz.timezone("America/New_York"))
```

---

**Version**: 1.0
