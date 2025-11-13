# API & Server: REST API и Веб-Сервер

## FastAPI

**Package**: `fastapi`
**Purpose**: Modern web framework for APIs

```python
from fastapi import FastAPI, HTTPException

app = FastAPI(title="LightRAG API")

@app.post("/documents")
async def insert_document(content: str):
    await lightrag.ainsert(content)
    return {"status": "success"}

@app.post("/query")
async def query(query: str, mode: str = "local"):
    result = await lightrag.aquery(query, param=QueryParam(mode=mode))
    return {"answer": result}
```

## uvicorn

**Package**: `uvicorn`
**Purpose**: ASGI server

```bash
uvicorn lightrag.api.lightrag_server:app --host 0.0.0.0 --port 8020
```

## Authentication

### PyJWT

**Package**: `PyJWT`
**Purpose**: JWT token generation/validation

```python
import jwt

token = jwt.encode(
    {"user_id": user_id, "exp": expiration},
    secret_key,
    algorithm="HS256"
)

payload = jwt.decode(token, secret_key, algorithms=["HS256"])
```

### passlib

**Package**: `passlib[bcrypt]`
**Purpose**: Password hashing

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Hash password
hashed = pwd_context.hash("my_password")

# Verify
is_valid = pwd_context.verify("my_password", hashed)
```

---

**Version**: 1.0
