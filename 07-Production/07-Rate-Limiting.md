# Rate Limiting - Bảo Vệ Service

## 1. Tại Sao Cần Rate Limit?

- 💰 **Cost control**: ngăn user "lạm dụng" API
- 🛡️ **DDoS protection**: tránh abuse
- ⚖️ **Fair usage**: ai cũng được phục vụ
- 🔌 **Upstream limits**: respect LLM provider limits

---

## 2. Loại Rate Limit

### 2.1 Token Bucket
- N "tokens" trong bucket, refill rate
- Mỗi request "tiêu" 1 token
- Hết tokens → reject

### 2.2 Fixed Window
- Counter reset mỗi window (vd: 100/minute)
- Đơn giản nhưng spike-prone

### 2.3 Sliding Window
- Count requests trong N giây gần nhất
- Smooth hơn

### 2.4 Leaky Bucket
- Queue requests, process at fixed rate
- Buffer spikes

---

## 3. slowapi - FastAPI Rate Limit

```bash
pip install slowapi
```

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import FastAPI, Request

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/chat")
@limiter.limit("10/minute")
async def chat(request: Request, ...):
    ...
```

### Per-User Limit

```python
def get_user_id(request: Request):
    return request.state.user.id  # Set by auth middleware

limiter = Limiter(key_func=get_user_id)

@app.post("/chat")
@limiter.limit("100/hour")
async def chat(request: Request):
    ...
```

### Tier-Based

```python
def get_user_tier(request):
    user = request.state.user
    return f"{user.id}:{user.tier}"

@app.post("/chat")
@limiter.limit(lambda req: {
    "free": "10/day",
    "pro": "100/day",
    "enterprise": "10000/day",
}[req.state.user.tier])
async def chat(...):
    ...
```

---

## 4. Custom Rate Limiter (Redis)

```python
import time
from redis.asyncio import Redis

class TokenBucketLimiter:
    def __init__(self, redis: Redis, capacity: int, refill_rate: float):
        self.redis = redis
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
    
    async def consume(self, key: str, tokens: int = 1) -> bool:
        """Returns True if allowed, False if rate limited"""
        now = time.time()
        
        # Lua script for atomic operation
        lua = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local tokens_requested = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])
        
        local data = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(data[1] or capacity)
        local last_refill = tonumber(data[2] or now)
        
        -- Refill
        local elapsed = now - last_refill
        tokens = math.min(capacity, tokens + elapsed * refill_rate)
        
        if tokens >= tokens_requested then
            tokens = tokens - tokens_requested
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)
            return 1
        else
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            return 0
        end
        """
        
        result = await self.redis.eval(
            lua, 1, key, self.capacity, self.refill_rate, tokens, now
        )
        return bool(result)

# Use
limiter = TokenBucketLimiter(redis, capacity=100, refill_rate=1/60)  # 100 tokens, refill 1/min

async def query(user_id, question):
    allowed = await limiter.consume(f"user:{user_id}")
    if not allowed:
        raise HTTPException(429, "Rate limit exceeded")
    
    return await chain.ainvoke(question)
```

---

## 5. Token-Based Limit (Cho LLM)

Không chỉ count requests, count tokens:

```python
class TokenLimiter:
    def __init__(self, redis, tokens_per_day):
        self.redis = redis
        self.limit = tokens_per_day
    
    async def check_and_consume(self, user_id: str, tokens: int):
        key = f"tokens:{user_id}:{datetime.now().date()}"
        
        current = int(await self.redis.get(key) or 0)
        if current + tokens > self.limit:
            return False
        
        await self.redis.incrby(key, tokens)
        await self.redis.expire(key, 86400 * 2)  # Keep 2 days
        return True

# Use
tlimiter = TokenLimiter(redis, tokens_per_day=100_000)

async def query(user_id, question):
    # Estimate tokens first
    estimated = count_tokens(question) + 500  # Reserve for output
    
    if not await tlimiter.check_and_consume(user_id, estimated):
        raise HTTPException(429, "Daily token limit reached")
    
    return await chain.ainvoke(question)
```

---

## 6. LLM API Rate Limiting

Respect provider limits:

```python
from langchain.rate_limiters import InMemoryRateLimiter

# Limit OpenAI calls
rate_limiter = InMemoryRateLimiter(
    requests_per_second=10,
    check_every_n_seconds=0.1,
    max_bucket_size=100,
)

llm = ChatOpenAI(
    model="gpt-4o-mini",
    rate_limiter=rate_limiter,
)

# LLM tự wait nếu vượt rate
```

---

## 7. Retry Với Backoff

Khi gặp 429 (rate limit):

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
)

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=4, max=60),
    retry=retry_if_exception_type(RateLimitError),
)
async def safe_llm_call(prompt):
    return await llm.ainvoke(prompt)
```

LangChain built-in:
```python
llm_with_retry = llm.with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True,
)
```

---

## 8. Queue-Based Throttling

Khi quá tải, đẩy vào queue:

```python
import asyncio

class QueueThrottler:
    def __init__(self, max_concurrent=10):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.queue = asyncio.Queue()
    
    async def submit(self, coro):
        async with self.semaphore:
            return await coro

throttler = QueueThrottler(max_concurrent=50)

@app.post("/query")
async def query(req):
    return await throttler.submit(chain.ainvoke(req.question))
```

---

## 9. Response Headers

Tell client về rate limit status:

```python
@app.post("/chat")
async def chat(request: Request, response: Response):
    user_id = request.state.user.id
    
    # Get current usage
    used = await get_user_usage(user_id)
    limit = get_user_limit(user_id)
    remaining = max(0, limit - used)
    
    # Set headers
    response.headers["X-RateLimit-Limit"] = str(limit)
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    response.headers["X-RateLimit-Reset"] = str(get_reset_time())
    
    if remaining == 0:
        raise HTTPException(429, "Rate limit exceeded")
    
    # Process
    return await process()
```

---

## 10. Distributed Rate Limit

Multi-instance API → cần shared counter:

```python
# Sai: in-memory không share
in_memory_count = {}

# Đúng: Redis shared
from redis.asyncio import Redis

async def check_limit(user_id, limit, window_sec=60):
    redis = Redis(host="redis")
    key = f"rl:{user_id}:{int(time.time() // window_sec)}"
    
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, window_sec)
    
    return count <= limit
```

---

## 11. Adaptive Rate Limit

Tăng limit cho good users, giảm cho abusers:

```python
class AdaptiveLimiter:
    def __init__(self, redis):
        self.redis = redis
    
    async def get_user_limit(self, user_id):
        # Check user behavior
        violations = await self.redis.get(f"violations:{user_id}") or 0
        
        if int(violations) > 10:
            return 5  # Reduce
        elif int(violations) == 0:
            return 100  # Generous
        return 50  # Default
    
    async def record_violation(self, user_id):
        await self.redis.incr(f"violations:{user_id}")
        await self.redis.expire(f"violations:{user_id}", 86400 * 30)
```

---

## 12. Demo: Production Rate Limit

```python
from fastapi import FastAPI, HTTPException, Request, Depends
from redis.asyncio import Redis
import time

app = FastAPI()
redis = Redis(host="redis", port=6379)

class MultiTierLimiter:
    """Per-user RPM and TPM limits"""
    
    LIMITS = {
        "free": {"rpm": 10, "tpm": 5000},
        "pro": {"rpm": 100, "tpm": 100000},
        "enterprise": {"rpm": 10000, "tpm": 10000000},
    }
    
    def __init__(self, redis):
        self.redis = redis
    
    async def check(self, user_id, tier, estimated_tokens):
        limits = self.LIMITS[tier]
        minute_key = int(time.time() // 60)
        
        # RPM check
        rpm_key = f"rpm:{user_id}:{minute_key}"
        rpm_count = await self.redis.incr(rpm_key)
        if rpm_count == 1:
            await self.redis.expire(rpm_key, 120)
        if rpm_count > limits["rpm"]:
            return False, f"RPM exceeded ({rpm_count}/{limits['rpm']})"
        
        # TPM check
        tpm_key = f"tpm:{user_id}:{minute_key}"
        tpm_count = await self.redis.incrby(tpm_key, estimated_tokens)
        if tpm_count == estimated_tokens:
            await self.redis.expire(tpm_key, 120)
        if tpm_count > limits["tpm"]:
            return False, f"TPM exceeded ({tpm_count}/{limits['tpm']})"
        
        return True, "OK"

limiter = MultiTierLimiter(redis)

@app.post("/query")
async def query(req, user=Depends(get_user)):
    estimated = count_tokens(req.question) + 500
    
    ok, message = await limiter.check(user.id, user.tier, estimated)
    if not ok:
        raise HTTPException(429, message)
    
    return await chain.ainvoke(req.question)
```

---

## 13. Best Practices

✅ **Different limits per tier**

✅ **Both RPM and TPM**: requests và tokens

✅ **Return clear errors**: với retry-after header

✅ **Test load**: với Locust để find limits

✅ **Monitor 429s**: ai bị limit nhiều?

✅ **Graceful degradation**: cheaper model khi gần limit

---

## 14. Checklist

- [ ] slowapi cho FastAPI
- [ ] Token bucket implementation
- [ ] Token-based limit (cho LLM)
- [ ] Backoff & retry
- [ ] Redis-based distributed limit
- [ ] Response headers
- [ ] Tier-based limits

➡️ **Tiếp theo**: [Security](./08-Security.md)
