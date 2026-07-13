# Caching - Tăng Tốc Và Giảm Chi Phí

## 1. Tại Sao Cần Cache?

Cùng 1 câu hỏi → cùng 1 answer (thường). Cache để:
- ⚡ **Faster**: < 10ms vs 1-5s LLM call
- 💰 **Cheaper**: free vs $0.01/call
- 🛡️ **Reliability**: không phụ thuộc API uptime

---

## 2. Loại Cache

| Type | What | When |
|------|------|------|
| **Exact match** | Cache prompt giống hệt | FAQ, repeated queries |
| **Semantic** | Cache theo nghĩa | Paraphrased queries |
| **Prompt caching** | Cache prefix prompt | Long system prompt |
| **Embedding** | Cache embedding vectors | Re-indexing |
| **Retriever** | Cache retrieval results | Hot queries |

---

## 3. In-Memory Cache (Quick Start)

```python
from langchain.globals import set_llm_cache
from langchain.cache import InMemoryCache

set_llm_cache(InMemoryCache())

# Lần 1: gọi API
result1 = llm.invoke("Hello")

# Lần 2: cache hit
result2 = llm.invoke("Hello")  # Tức thì
```

---

## 4. SQLite Cache (Persistent)

```python
from langchain_community.cache import SQLiteCache

set_llm_cache(SQLiteCache(database_path=".langchain.db"))
```

→ Cache persist qua restart.

---

## 5. Redis Cache

```bash
# Run Redis
docker run -p 6379:6379 redis
```

```python
from langchain_community.cache import RedisCache
from redis import Redis

set_llm_cache(RedisCache(redis_=Redis(host="localhost", port=6379)))
```

### With TTL

```python
from langchain_community.cache import RedisCache

set_llm_cache(RedisCache(redis_=Redis(), ttl=3600))  # 1 hour
```

---

## 6. Semantic Cache

Match by **nghĩa** thay vì exact string:

```python
from langchain.cache import RedisSemanticCache
from langchain_openai import OpenAIEmbeddings

set_llm_cache(RedisSemanticCache(
    redis_url="redis://localhost:6379",
    embedding=OpenAIEmbeddings(),
    score_threshold=0.95,  # Cache hit nếu similarity > 0.95
))

# Cache hit
llm.invoke("Hello, how are you?")
llm.invoke("Hi, how do you do?")  # Hit nhờ semantic
```

⚠️ **Lưu ý**: vẫn tốn embedding cost cho lookup.

---

## 7. Prompt Caching (Anthropic/OpenAI Native)

Cache **prefix** của prompt (system message, examples, ...). 

### Anthropic

```python
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(
    model="claude-sonnet-4-6",
    extra_headers={"anthropic-beta": "prompt-caching-2024-07-31"},
)

system_message = """[LONG INSTRUCTION ~5000 tokens]"""

messages = [
    {
        "role": "system",
        "content": [
            {
                "type": "text",
                "text": system_message,
                "cache_control": {"type": "ephemeral"}  # Cache đây
            }
        ]
    },
    {"role": "user", "content": "Question 1"},
]

# Lần đầu: full cost
# Lần sau (cùng system): chỉ trả ~10% cho cached part
```

### OpenAI

OpenAI cache tự động cho prompts > 1024 tokens, không cần config.

**Tiết kiệm**: 50% cost cho cached tokens, lower latency.

---

## 8. Cache At Chain Level

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_chain_invoke(question: str) -> str:
    return chain.invoke({"question": question})

result = cached_chain_invoke("What is AI?")  # Cache
```

---

## 9. Cache Retriever Results

```python
import hashlib
import json
from langchain_core.runnables import RunnableLambda

cache = {}

def cached_retriever(query: str):
    key = hashlib.md5(query.encode()).hexdigest()
    if key in cache:
        return cache[key]
    
    docs = retriever.invoke(query)
    cache[key] = docs
    return docs

cached = RunnableLambda(cached_retriever)
```

---

## 10. Cache Embeddings

Tránh re-embed text cũ:

```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore

store = LocalFileStore("./embedding_cache/")
underlying = OpenAIEmbeddings(model="text-embedding-3-small")

cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=underlying,
    document_embedding_cache=store,
    namespace="oai-small-v1",
)

# Lần đầu: gọi API
v1 = cached_embeddings.embed_query("Hello")

# Lần sau: cache (cùng text)
v2 = cached_embeddings.embed_query("Hello")
```

---

## 11. Cache Invalidation

### 11.1 TTL

```python
RedisCache(redis_=Redis(), ttl=3600)  # 1h
```

### 11.2 Manual Clear

```python
from langchain.globals import set_llm_cache

# Clear all
set_llm_cache(None)  # Disable

# Or với Redis
redis.flushdb()
```

### 11.3 Versioned Cache

```python
class VersionedCache:
    def __init__(self, version):
        self.version = version
        self.cache = {}
    
    def get(self, key):
        return self.cache.get(f"{self.version}:{key}")
    
    def set(self, key, value):
        self.cache[f"{self.version}:{key}"] = value

# Bump version để invalidate
cache_v2 = VersionedCache("v2")
```

---

## 12. Hybrid Cache Strategy

```python
class HybridCache:
    """L1 (memory) + L2 (Redis) cache"""
    
    def __init__(self, max_memory=1000, redis_client=None):
        from collections import OrderedDict
        self.memory = OrderedDict()
        self.max_memory = max_memory
        self.redis = redis_client
    
    def get(self, key):
        # L1
        if key in self.memory:
            self.memory.move_to_end(key)
            return self.memory[key]
        
        # L2
        if self.redis:
            value = self.redis.get(key)
            if value:
                # Promote to L1
                self._set_memory(key, value)
                return value
        
        return None
    
    def set(self, key, value):
        self._set_memory(key, value)
        if self.redis:
            self.redis.setex(key, 3600, value)
    
    def _set_memory(self, key, value):
        if key in self.memory:
            self.memory.move_to_end(key)
        else:
            if len(self.memory) >= self.max_memory:
                self.memory.popitem(last=False)
            self.memory[key] = value
```

---

## 13. Monitoring Cache

```python
class TrackedCache:
    def __init__(self, cache):
        self.cache = cache
        self.hits = 0
        self.misses = 0
    
    def get(self, key):
        value = self.cache.get(key)
        if value:
            self.hits += 1
        else:
            self.misses += 1
        return value
    
    def stats(self):
        total = self.hits + self.misses
        return {
            "hits": self.hits,
            "misses": self.misses,
            "hit_rate": self.hits / total if total else 0,
        }
```

---

## 14. Cache Patterns

### 14.1 Cache-Aside

```python
def get_answer(question):
    cached = cache.get(question)
    if cached:
        return cached
    
    answer = chain.invoke(question)
    cache.set(question, answer)
    return answer
```

### 14.2 Write-Through

```python
def get_or_create(question):
    answer = chain.invoke(question)  # Always call
    cache.set(question, answer)      # Always write
    return answer
```

### 14.3 Stale-While-Revalidate

```python
def get_with_swr(question):
    cached = cache.get(question)
    if cached:
        # Return cached, refresh async
        background_refresh(question)
        return cached
    
    return refresh(question)
```

---

## 15. Best Practices

✅ **Start simple**: InMemory or SQLite

✅ **Production**: Redis với TTL

✅ **Semantic cache cho FAQ**

✅ **Native prompt caching**: bật cho Claude/OpenAI

✅ **Don't cache personalized**: per-user data sai cache key

✅ **Cache invalidation**: TTL, versioning, manual

✅ **Monitor hit rate**: target > 30%

---

## 16. Bài Tập

### Bài 1: FAQ Bot Với Cache
Build FAQ bot, đo:
- No cache: latency, cost
- Exact cache: same
- Semantic cache: same

### Bài 2: Hybrid Cache
Implement L1 (memory) + L2 (Redis) cache với metrics.

### Bài 3: Prompt Cache Anthropic
Test prompt caching với Claude. Đo cost reduction.

---

## 17. Checklist

- [ ] InMemory, SQLite, Redis cache
- [ ] Semantic cache
- [ ] Prompt caching (Anthropic/OpenAI)
- [ ] Embedding cache
- [ ] TTL & invalidation
- [ ] Hit rate monitoring

➡️ **Tiếp theo**: [Cost Optimization](./02-Cost-Optimization.md)
