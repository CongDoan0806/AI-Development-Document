# Scaling AI Applications

## 1. Vấn Đề Scaling

```
1 user, 1 request: chạy được trên laptop
100 users, 10 req/s: cần tune
1000+ users, 100+ req/s: cần infrastructure
```

**Bottlenecks**:
- LLM API rate limits
- Vector DB query latency
- Memory cho document storage
- Network I/O

---

## 2. Vertical vs Horizontal Scaling

### Vertical
Tăng CPU/RAM của 1 server.
- ✅ Đơn giản
- ❌ Giới hạn, expensive

### Horizontal
Thêm nhiều servers, load balance.
- ✅ Scale rộng
- ❌ Phức tạp hơn (state sync, cache)

→ Web tier: horizontal. Vector DB: tùy.

---

## 3. Concurrency & Async

### 3.1 Async I/O Everywhere

```python
# Đồng bộ - chỉ xử lý 1 lúc 1 request
def query(question):
    return chain.invoke(question)  # Block ~3s

# Async - xử lý 100 requests song song
async def query(question):
    return await chain.ainvoke(question)
```

### 3.2 Connection Pooling

```python
# Postgres
from psycopg_pool import ConnectionPool

pool = ConnectionPool(
    DB_URI,
    min_size=10,
    max_size=50,
)

async def query_db(sql):
    async with pool.connection() as conn:
        return await conn.execute(sql)

# Redis
from redis.asyncio import Redis
redis = Redis(host="redis", port=6379, max_connections=50)

# HTTP
import aiohttp
session = aiohttp.ClientSession(connector=aiohttp.TCPConnector(limit=100))
```

### 3.3 Semaphore Control

Giới hạn concurrent calls:

```python
semaphore = asyncio.Semaphore(50)  # Max 50 concurrent LLM calls

async def safe_llm_call(prompt):
    async with semaphore:
        return await llm.ainvoke(prompt)
```

---

## 4. Worker Processes

### 4.1 Gunicorn + Uvicorn

```bash
# 4 workers * 4 threads = 16 concurrent
gunicorn main:app \
    -w 4 \
    -k uvicorn.workers.UvicornWorker \
    --threads 4 \
    --bind 0.0.0.0:8000
```

**Số workers**: `2 * CPU_count + 1`

### 4.2 Multi-Process Considerations

Mỗi worker là 1 process độc lập:
- ❌ Memory không share
- ❌ Cache trong RAM không hiệu quả
- ✅ Crash isolation

→ Dùng Redis/external cache.

---

## 5. Load Balancing

```
Internet
   ↓
[Load Balancer] (nginx, AWS ALB)
   ↓
┌──────┬──────┬──────┐
│ API1 │ API2 │ API3 │
└──────┴──────┴──────┘
   ↓        ↓       ↓
[Shared Redis, DB, Vector Store]
```

### Nginx Config

```nginx
upstream api_backend {
    least_conn;
    server api1:8000;
    server api2:8000;
    server api3:8000;
}

server {
    listen 80;
    location / {
        proxy_pass http://api_backend;
        proxy_buffering off;  # For streaming
        proxy_read_timeout 300s;
    }
}
```

---

## 6. Rate Limiting

### 6.1 LLM Provider Limits

OpenAI Tier 4 example:
- 10K RPM, 30M TPM

```python
import asyncio
from datetime import datetime, timedelta

class RateLimiter:
    def __init__(self, max_per_minute):
        self.max = max_per_minute
        self.calls = []
        self.lock = asyncio.Lock()
    
    async def acquire(self):
        async with self.lock:
            now = datetime.now()
            self.calls = [t for t in self.calls if t > now - timedelta(minutes=1)]
            
            if len(self.calls) >= self.max:
                sleep_time = (self.calls[0] - (now - timedelta(minutes=1))).total_seconds()
                await asyncio.sleep(sleep_time)
            
            self.calls.append(now)

limiter = RateLimiter(max_per_minute=500)

async def llm_call(prompt):
    await limiter.acquire()
    return await llm.ainvoke(prompt)
```

### 6.2 Per-User Rate Limit

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

def get_user_id(request):
    return request.state.user_id

limiter = Limiter(key_func=get_user_id)

@app.post("/chat")
@limiter.limit("100/hour")
async def chat(...):
    ...
```

---

## 7. Vector DB Scaling

### 7.1 Chroma → Pinecone Migration

Chroma local OK cho dev, production cần:
- Pinecone (managed)
- Qdrant Cluster
- Milvus distributed

```python
# Migrate
from langchain_pinecone import PineconeVectorStore

# Read from Chroma
chroma_docs = chroma_store.get()

# Write to Pinecone
pinecone_store = PineconeVectorStore.from_documents(
    documents=[Document(page_content=d, metadata=m) for d, m in zip(chroma_docs["documents"], chroma_docs["metadatas"])],
    embedding=embeddings,
    index_name="prod-index",
)
```

### 7.2 Sharding

Chia vector DB theo:
- Tenant (per-customer)
- Topic
- Date

```python
def get_index_name(user_id):
    return f"prod-{user_id[:2]}"  # 256 shards

vectorstore = Chroma(
    collection_name=get_index_name(user_id),
    embedding_function=embeddings,
)
```

---

## 8. Caching For Scale

### 8.1 Multi-Layer

```
L1: In-process (LRU, very fast, small)
L2: Redis (shared across instances)
L3: Database (persistent)
```

### 8.2 Cache Stampede Protection

Nhiều requests cùng query miss cache → tất cả gọi LLM cùng lúc.

```python
import asyncio

class StampedeProtectedCache:
    def __init__(self, cache):
        self.cache = cache
        self.in_progress = {}  # key -> Future
    
    async def get_or_compute(self, key, compute_fn):
        # Check cache
        cached = await self.cache.get(key)
        if cached:
            return cached
        
        # Check in-progress
        if key in self.in_progress:
            return await self.in_progress[key]
        
        # Compute (only 1 caller does this)
        future = asyncio.ensure_future(compute_fn())
        self.in_progress[key] = future
        
        try:
            result = await future
            await self.cache.set(key, result)
            return result
        finally:
            del self.in_progress[key]
```

---

## 9. Queue-Based Architecture

Cho long-running tasks (bulk processing):

```
Client → API → Queue (Celery/RQ) → Workers → Result Storage
```

### Celery Example

```python
from celery import Celery

app = Celery("tasks", broker="redis://localhost:6379")

@app.task
def process_long_query(question):
    result = chain.invoke(question)
    save_result(question, result)
    return result

# API
@fastapi_app.post("/submit")
async def submit(req):
    task = process_long_query.delay(req.question)
    return {"task_id": task.id}

@fastapi_app.get("/status/{task_id}")
async def status(task_id):
    task = process_long_query.AsyncResult(task_id)
    return {"status": task.status, "result": task.result if task.ready() else None}
```

---

## 10. Database Scaling

### 10.1 Read Replicas

```python
# Primary cho writes
write_db = create_engine("postgresql://primary:5432/db")

# Read replicas
read_db = create_engine("postgresql://replica:5432/db")

# Use replica cho read queries
def get_user(user_id):
    return read_db.execute("SELECT * FROM users WHERE id = ?", user_id)

def update_user(user_id, data):
    write_db.execute("UPDATE users SET ... WHERE id = ?", user_id)
```

### 10.2 Sharding User Data

```python
def get_db_for_user(user_id):
    shard = hash(user_id) % NUM_SHARDS
    return databases[shard]
```

---

## 11. CDN For Static Assets

```python
# Static UI, vector DB read-only data
# Sử dụng CloudFront, Cloudflare
```

---

## 12. Edge Computing

LangChain qua Cloudflare Workers, Vercel Edge:

```javascript
// Vercel Edge
export default async function handler(req) {
    const response = await fetch("https://api.openai.com/v1/chat/completions", {
        ...
    });
    return new Response(response.body, { headers: response.headers });
}
```

- ✅ Low latency (gần user)
- ❌ Limited compute, no Python LangChain

---

## 13. Auto-Scaling

### Kubernetes HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### AWS ECS Auto Scaling

Tương tự, scale based on CPU/memory/request rate.

---

## 14. Load Testing

```python
# pip install locust
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(1, 5)
    
    @task
    def query(self):
        self.client.post("/query", json={
            "question": "Test question",
        }, headers={"Authorization": "Bearer ..."})

# Run
# locust -f locustfile.py --host=http://api.example.com
```

UI tại `http://localhost:8089`:
- Simulate 100 users, 10 spawn/sec
- See p50/p95/p99
- Find breaking point

---

## 15. Cost At Scale

```
1000 users × 100 queries/day × $0.01/query = $1000/day = $30K/month

Tối ưu:
- Cache 30% hit rate: -30% = $21K
- Cheaper model 50% queries: -25% = $15K  
- Open source for 30%: -15% = $12K
```

---

## 16. Demo: Scalable Architecture

```python
# main.py - Production setup
from fastapi import FastAPI
from langchain_openai import ChatOpenAI
from redis.asyncio import Redis
import asyncio

app = FastAPI()

# Connection pools
redis = Redis(host="redis", max_connections=50)

# Semaphores
llm_semaphore = asyncio.Semaphore(100)  # Max concurrent LLM
db_semaphore = asyncio.Semaphore(50)

llm = ChatOpenAI(model="gpt-4o-mini", max_retries=3, timeout=30)

async def cached_query(question: str):
    # Check cache
    cached = await redis.get(f"q:{question}")
    if cached:
        return cached.decode()
    
    # Compute
    async with llm_semaphore:
        result = await llm.ainvoke(question)
    
    # Cache
    await redis.setex(f"q:{question}", 3600, result.content)
    return result.content

@app.post("/query")
async def query(question: str):
    return {"answer": await cached_query(question)}

@app.get("/health")
async def health():
    # Check dependencies
    try:
        await redis.ping()
    except:
        return {"status": "degraded"}
    return {"status": "ok"}
```

Deploy:
```bash
# Multi-worker
gunicorn main:app -w 8 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000

# Docker
docker-compose up --scale api=4
```

---

## 17. Checklist

- [ ] Async I/O everywhere
- [ ] Connection pooling
- [ ] Semaphore concurrency limits
- [ ] Multi-worker (gunicorn)
- [ ] Load balancer (nginx)
- [ ] Rate limiting
- [ ] Multi-layer cache
- [ ] Queue for long tasks
- [ ] Load testing với Locust
- [ ] Auto-scaling

➡️ **Tiếp theo**: [Rate Limiting](./07-Rate-Limiting.md)
