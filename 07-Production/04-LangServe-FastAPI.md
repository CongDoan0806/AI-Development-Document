# LangServe & FastAPI - Deploy LangChain App

## 1. Tổng Quan

| Option | Use case |
|--------|----------|
| **LangServe** | LangChain → REST API tự động |
| **FastAPI** | Tự build API, flexible |
| **LangGraph Platform** | Deploy LangGraph apps |

---

## 2. LangServe - Auto API

### 2.1 Setup

```bash
pip install langserve[all] fastapi uvicorn
```

### 2.2 Hello World

```python
# main.py
from fastapi import FastAPI
from langserve import add_routes
from langchain_openai import ChatOpenAI

app = FastAPI(title="My LLM API")

# Add LangChain runnable as route
add_routes(
    app,
    ChatOpenAI(model="gpt-4o-mini"),
    path="/openai",
)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Run:
```bash
uvicorn main:app --reload
```

Access:
- `POST /openai/invoke` - Single invocation
- `POST /openai/batch` - Batch
- `POST /openai/stream` - Streaming
- `POST /openai/stream_log` - Stream với log
- `GET /openai/playground` - Built-in UI

### 2.3 Add Chain

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Translate to French: {text}")
chain = prompt | ChatOpenAI() | StrOutputParser()

add_routes(app, chain, path="/translate")
```

### 2.4 Multiple Routes

```python
add_routes(app, openai_chain, path="/openai")
add_routes(app, claude_chain, path="/claude")
add_routes(app, rag_chain, path="/rag")
```

### 2.5 Input/Output Schema

```python
from pydantic import BaseModel

class Question(BaseModel):
    text: str
    language: str = "en"

class Answer(BaseModel):
    response: str
    confidence: float

typed_chain = chain.with_types(input_type=Question, output_type=Answer)
add_routes(app, typed_chain, path="/qa")
```

### 2.6 Playground UI

Mở `http://localhost:8000/openai/playground` → UI để test.

---

## 3. Client SDK

```python
from langserve import RemoteRunnable

# Connect to LangServe API
remote_chain = RemoteRunnable("http://localhost:8000/translate/")

# Use như local
result = remote_chain.invoke({"text": "Hello"})

# Stream
for chunk in remote_chain.stream({"text": "Hi"}):
    print(chunk)

# Async
result = await remote_chain.ainvoke({"text": "Hello"})
```

---

## 4. FastAPI - Custom API

### 4.1 Basic Setup

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from langchain_openai import ChatOpenAI

app = FastAPI()
llm = ChatOpenAI(model="gpt-4o-mini")

class ChatRequest(BaseModel):
    message: str
    temperature: float = 0.7

class ChatResponse(BaseModel):
    response: str
    tokens_used: int

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    try:
        response = await llm.ainvoke(request.message)
        return ChatResponse(
            response=response.content,
            tokens_used=response.usage_metadata["total_tokens"],
        )
    except Exception as e:
        raise HTTPException(500, str(e))
```

### 4.2 Background Tasks

```python
from fastapi import BackgroundTasks

@app.post("/chat-bg")
async def chat_background(request: ChatRequest, background_tasks: BackgroundTasks):
    response = await llm.ainvoke(request.message)
    
    # Log async
    background_tasks.add_task(log_to_db, request, response)
    
    return {"response": response.content}

def log_to_db(request, response):
    # Long-running log
    db.insert({"request": request.dict(), "response": response.content})
```

### 4.3 Middleware

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourapp.com"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Custom middleware
@app.middleware("http")
async def add_timing(request, call_next):
    import time
    start = time.time()
    response = await call_next(request)
    elapsed = time.time() - start
    response.headers["X-Process-Time"] = str(elapsed)
    return response
```

---

## 5. Authentication

### 5.1 API Key

```python
from fastapi import Header, HTTPException

async def verify_api_key(x_api_key: str = Header()):
    if x_api_key != "secret-key":
        raise HTTPException(401, "Invalid API key")

@app.post("/chat", dependencies=[Depends(verify_api_key)])
async def chat(request: ChatRequest):
    ...
```

### 5.2 OAuth/JWT

```python
from fastapi.security import OAuth2PasswordBearer
import jwt

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, "secret", algorithms=["HS256"])
        return payload["sub"]
    except:
        raise HTTPException(401, "Invalid token")

@app.post("/chat")
async def chat(request: ChatRequest, user_id: str = Depends(get_user)):
    # Use user_id
    ...
```

---

## 6. Rate Limiting

```python
# pip install slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/chat")
@limiter.limit("10/minute")  # 10 requests per minute per IP
async def chat(request: Request, body: ChatRequest):
    ...
```

---

## 7. Validation & Error Handling

```python
from pydantic import BaseModel, Field, validator
from fastapi import HTTPException

class ChatRequest(BaseModel):
    message: str = Field(min_length=1, max_length=2000)
    temperature: float = Field(0.7, ge=0, le=2)
    
    @validator("message")
    def no_dangerous_content(cls, v):
        if any(bad in v.lower() for bad in ["delete from", "drop table"]):
            raise ValueError("Suspicious content")
        return v

@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    logger.error(f"Unhandled: {exc}")
    return JSONResponse(
        status_code=500,
        content={"error": "Internal server error"},
    )
```

---

## 8. Async + Concurrent Requests

```python
import asyncio
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

@app.post("/batch")
async def batch_chat(messages: list[str]):
    # Process in parallel
    results = await asyncio.gather(*[
        llm.ainvoke(m) for m in messages
    ])
    return [r.content for r in results]
```

---

## 9. Streaming Endpoint

```python
from fastapi.responses import StreamingResponse

@app.post("/chat-stream")
async def chat_stream(request: ChatRequest):
    async def generate():
        async for chunk in llm.astream(request.message):
            yield chunk.content
    
    return StreamingResponse(generate(), media_type="text/plain")
```

---

## 10. Health Check & Metrics

```python
@app.get("/health")
async def health():
    return {"status": "ok"}

@app.get("/health/llm")
async def health_llm():
    try:
        response = await llm.ainvoke("ping")
        return {"status": "ok", "model": llm.model_name}
    except Exception as e:
        raise HTTPException(503, f"LLM unavailable: {e}")

# Prometheus metrics
from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app)
# Metrics at /metrics
```

---

## 11. Deployment Stack

```
Client
   ↓
[Load Balancer (nginx, ALB)]
   ↓
[FastAPI workers (uvicorn × N)]
   ↓
[Redis (cache)]
[Vector DB (Chroma/Pinecone)]
[LLM API (OpenAI/Anthropic)]
   ↓
[Postgres (data)]
[LangSmith (monitoring)]
```

### Production Server

```bash
# Gunicorn + uvicorn workers
pip install gunicorn

gunicorn main:app \
    -w 4 \
    -k uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --timeout 120
```

---

## 12. Docker

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["gunicorn", "main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

### docker-compose.yml

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
      - postgres
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

## 13. Demo: Production-Ready RAG API

```python
from fastapi import FastAPI, HTTPException, Depends, Header
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
import json
import logging
import os

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="RAG API")

# Setup
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(embedding_function=embeddings, persist_directory="./db")
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

prompt = ChatPromptTemplate.from_template("""
Trả lời dựa trên context.
Context: {context}
Q: {question}
A:""")

def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

# Models
class QueryRequest(BaseModel):
    question: str = Field(min_length=1, max_length=2000)
    stream: bool = False

class QueryResponse(BaseModel):
    answer: str
    sources: list[dict]
    tokens_used: int

# Auth
async def verify_token(authorization: str = Header()):
    if not authorization.startswith("Bearer "):
        raise HTTPException(401, "Invalid auth")
    token = authorization[7:]
    if token != os.environ.get("API_TOKEN"):
        raise HTTPException(401, "Invalid token")

# Routes
@app.post("/query", response_model=QueryResponse)
async def query(req: QueryRequest, auth=Depends(verify_token)):
    try:
        docs = await retriever.ainvoke(req.question)
        context = format_docs(docs)
        
        from langchain.callbacks import get_openai_callback
        with get_openai_callback() as cb:
            answer = await (prompt | llm | StrOutputParser()).ainvoke({
                "context": context,
                "question": req.question,
            })
        
        return QueryResponse(
            answer=answer,
            sources=[{"content": d.page_content[:200], "metadata": d.metadata} for d in docs],
            tokens_used=cb.total_tokens,
        )
    except Exception as e:
        logger.error(f"Query error: {e}")
        raise HTTPException(500, "Internal error")

@app.post("/query-stream")
async def query_stream(req: QueryRequest, auth=Depends(verify_token)):
    async def stream():
        try:
            docs = await retriever.ainvoke(req.question)
            yield f"data: {json.dumps({'sources': [d.metadata for d in docs]})}\n\n"
            
            chain = prompt | llm | StrOutputParser()
            async for chunk in chain.astream({"context": format_docs(docs), "question": req.question}):
                yield f"data: {json.dumps({'chunk': chunk})}\n\n"
            
            yield "data: [DONE]\n\n"
        except Exception as e:
            yield f"data: {json.dumps({'error': str(e)})}\n\n"
    
    return StreamingResponse(stream(), media_type="text/event-stream")

@app.get("/health")
async def health():
    return {"status": "ok"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 14. Best Practices

✅ **Async everywhere**: dùng `ainvoke`, `astream`

✅ **Validation**: Pydantic schemas

✅ **Auth**: API key/JWT

✅ **Rate limit**: per user

✅ **Health checks**: monitor

✅ **Streaming**: cho chat UX

✅ **Background tasks**: cho logging

✅ **Multi-worker**: gunicorn -w 4

---

## 15. Checklist

- [ ] LangServe setup
- [ ] FastAPI custom endpoint
- [ ] Authentication
- [ ] Rate limiting
- [ ] Streaming SSE
- [ ] Health checks
- [ ] Docker deployment
- [ ] Production server (gunicorn)

➡️ **Tiếp theo**: [Logging & Monitoring](./05-Logging-Monitoring.md)
