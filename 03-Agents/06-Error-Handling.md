# Error Handling - Xử Lý Lỗi Trong Agent

## 1. Các Loại Lỗi Thường Gặp

| Loại lỗi | Nguyên nhân | Giải pháp |
|----------|-------------|-----------|
| **API timeout** | LLM provider chậm | Retry + timeout |
| **Rate limit** | Quá nhiều request | Backoff, retry |
| **Tool error** | Tool fail (network, bug) | Tool-level retry |
| **Parse error** | LLM trả output sai format | OutputFixingParser |
| **Hallucinated tool** | LLM gọi tool không tồn tại | Validate trước khi exec |
| **Infinite loop** | Agent loop mãi không kết thúc | Max iterations |
| **Bad arguments** | LLM pass args sai schema | Pydantic validation |
| **Authorization** | Tool cần permission | Catch + ask user |

---

## 2. Retry Với with_retry

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# Auto retry với exponential backoff
llm_with_retry = llm.with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True,
    retry_if_exception_type=(Exception,),
)

# Tự retry nếu fail
response = llm_with_retry.invoke("Hello")
```

### Custom Retry Logic

```python
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(min=1, max=60),
)
def call_llm_with_retry(prompt):
    return llm.invoke(prompt)
```

---

## 3. Fallback Models

Nếu primary model fail → dùng backup:

```python
primary = ChatOpenAI(model="gpt-4o")
fallback = ChatOpenAI(model="gpt-4o-mini")
fallback2 = ChatAnthropic(model="claude-haiku-4-5")

llm_with_fallbacks = primary.with_fallbacks([fallback, fallback2])

# Nếu gpt-4o fail → thử gpt-4o-mini → thử Claude Haiku
response = llm_with_fallbacks.invoke("Hello")
```

### Fallback Cho Chain

```python
chain_a = prompt_a | llm_a
chain_b = prompt_b | llm_b  # Simpler fallback

chain_with_fallback = chain_a.with_fallbacks([chain_b])
```

---

## 4. Tool Error Handling

### 4.1 Raise ToolException

```python
from langchain_core.tools import tool, ToolException

@tool
def divide(a: float, b: float) -> float:
    """Chia 2 số"""
    if b == 0:
        raise ToolException("Cannot divide by zero. Use different value.")
    return a / b
```

LLM thấy error → tự thử lại với input khác.

### 4.2 Return Error As String

```python
@tool
def safe_divide(a: float, b: float) -> str:
    """Chia 2 số"""
    if b == 0:
        return "Error: Cannot divide by zero"
    return str(a / b)
```

### 4.3 Try-Except Trong Tool

```python
@tool
def fetch_data(url: str) -> str:
    """Fetch data từ URL"""
    try:
        import requests
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.text[:500]
    except requests.Timeout:
        return "Error: Request timed out. Try a different URL."
    except requests.HTTPError as e:
        return f"HTTP Error {e.response.status_code}: {e.response.reason}"
    except Exception as e:
        return f"Unknown error: {str(e)}"
```

---

## 5. Output Parser Errors

### 5.1 OutputFixingParser

```python
from langchain.output_parsers import OutputFixingParser, PydanticOutputParser
from pydantic import BaseModel

class Response(BaseModel):
    answer: str
    confidence: float

base_parser = PydanticOutputParser(pydantic_object=Response)
fixing_parser = OutputFixingParser.from_llm(parser=base_parser, llm=llm)

# Khi parse fail, fixing_parser tự gọi LLM để fix
try:
    result = fixing_parser.parse(bad_output)
except Exception as e:
    print(f"Cannot fix: {e}")
```

### 5.2 RetryOutputParser

```python
from langchain.output_parsers import RetryOutputParser

retry_parser = RetryOutputParser.from_llm(parser=base_parser, llm=llm)

# Khi fail, retry với original prompt + bad output
```

### 5.3 Try Structured Output First

```python
# Dùng with_structured_output (native) thay vì parser
structured_llm = llm.with_structured_output(Response)
# Ít fail hơn vì dùng tool calling
```

---

## 6. Validation Tool Arguments

```python
from pydantic import BaseModel, Field, validator

class TransferInput(BaseModel):
    amount: float = Field(gt=0, description="Amount > 0")
    to_account: str = Field(min_length=10, max_length=20)
    
    @validator("amount")
    def amount_limit(cls, v):
        if v > 1_000_000:
            raise ValueError("Amount > 1M needs approval")
        return v

@tool(args_schema=TransferInput)
def transfer_money(amount: float, to_account: str) -> str:
    """Transfer money"""
    return f"Transferred {amount} to {to_account}"

# Nếu LLM pass amount=-100 hoặc amount=2000000:
# → Pydantic raise → LLM thấy error → tự sửa
```

---

## 7. Max Iterations - Tránh Infinite Loop

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(llm, tools)

# Config recursion_limit
config = {"recursion_limit": 25}  # Max 25 LLM calls

try:
    result = agent.invoke(
        {"messages": [("user", "complex task")]},
        config=config,
    )
except GraphRecursionError:
    print("Agent reached max iterations")
```

---

## 8. Timeout

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-4o-mini",
    timeout=30,  # 30 giây timeout
    max_retries=2,
)
```

### Asyncio Timeout

```python
import asyncio

async def run_with_timeout(coro, seconds=60):
    try:
        return await asyncio.wait_for(coro, timeout=seconds)
    except asyncio.TimeoutError:
        return "Operation timed out"

result = await run_with_timeout(agent.ainvoke({...}), seconds=120)
```

---

## 9. Rate Limit Handling

```python
from langchain_community.callbacks.openai_info import OpenAICallbackHandler
import time

class RateLimiter:
    def __init__(self, max_per_minute=60):
        self.max = max_per_minute
        self.calls = []
    
    def wait_if_needed(self):
        now = time.time()
        self.calls = [t for t in self.calls if now - t < 60]
        if len(self.calls) >= self.max:
            sleep_time = 60 - (now - self.calls[0])
            time.sleep(sleep_time)
        self.calls.append(now)

limiter = RateLimiter(max_per_minute=50)

def call_llm(prompt):
    limiter.wait_if_needed()
    return llm.invoke(prompt)
```

---

## 10. Circuit Breaker Pattern

Nếu tool fail liên tục → tạm "ngắt mạch" (không gọi nữa).

```python
class CircuitBreaker:
    def __init__(self, threshold=5, recovery_time=60):
        self.failures = 0
        self.threshold = threshold
        self.recovery_time = recovery_time
        self.last_fail = None
        self.state = "closed"  # closed, open, half-open
    
    def call(self, func, *args, **kwargs):
        if self.state == "open":
            if time.time() - self.last_fail > self.recovery_time:
                self.state = "half-open"
            else:
                raise Exception("Circuit breaker is open")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_fail = time.time()
            if self.failures >= self.threshold:
                self.state = "open"
            raise

breaker = CircuitBreaker(threshold=3)

@tool
def flaky_tool(input: str) -> str:
    """Tool có thể fail"""
    try:
        return breaker.call(actual_function, input)
    except Exception as e:
        return f"Service unavailable: {e}"
```

---

## 11. Graceful Degradation

Khi tool fail → fallback đến simpler answer:

```python
@tool
def get_realtime_price(symbol: str) -> str:
    """Get realtime stock price"""
    try:
        return realtime_api.get(symbol)
    except:
        # Fallback to cached/static
        return cached_db.get(symbol, "Unable to fetch price")
```

---

## 12. Validation Layer Cho Agent

Wrap agent với validation:

```python
def safe_agent_invoke(agent, question):
    # Pre-checks
    if len(question) > 5000:
        return "Question too long"
    
    if any(bad in question.lower() for bad in ["hack", "exploit"]):
        return "Cannot help with that"
    
    # Run agent với timeout & max iterations
    try:
        result = agent.invoke(
            {"messages": [("user", question)]},
            config={"recursion_limit": 15, "timeout": 60},
        )
    except Exception as e:
        return f"Agent error: {str(e)[:200]}"
    
    # Post-validation
    answer = result["messages"][-1].content
    if not answer:
        return "No answer generated"
    
    return answer
```

---

## 13. Comprehensive Error Handler

```python
import logging
from typing import Any, Dict
from langchain_core.callbacks import BaseCallbackHandler

class ErrorTracker(BaseCallbackHandler):
    """Track errors trong agent execution"""
    
    def __init__(self):
        self.errors = []
    
    def on_tool_error(self, error: BaseException, **kwargs) -> None:
        self.errors.append({
            "type": "tool_error",
            "tool": kwargs.get("name"),
            "error": str(error),
        })
        logging.error(f"Tool error: {error}")
    
    def on_llm_error(self, error: BaseException, **kwargs) -> None:
        self.errors.append({
            "type": "llm_error",
            "error": str(error),
        })
        logging.error(f"LLM error: {error}")
    
    def on_chain_error(self, error: BaseException, **kwargs) -> None:
        self.errors.append({
            "type": "chain_error",
            "error": str(error),
        })

tracker = ErrorTracker()
result = agent.invoke(
    {"messages": [...]},
    config={"callbacks": [tracker]},
)

if tracker.errors:
    print(f"Errors during execution: {tracker.errors}")
```

---

## 14. Demo: Robust Agent

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool, ToolException
from langgraph.prebuilt import create_react_agent
from pydantic import BaseModel, Field, validator

# === LLMs with fallback ===
primary_llm = ChatOpenAI(model="gpt-4o", temperature=0, timeout=30)
fallback_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
llm = primary_llm.with_fallbacks([fallback_llm])

# === Tools with validation ===
class TransferArgs(BaseModel):
    amount: float = Field(gt=0, le=1_000_000)
    account: str = Field(min_length=10)

@tool(args_schema=TransferArgs)
def transfer(amount: float, account: str) -> str:
    """Transfer money. Max 1M."""
    try:
        # Mock transfer
        if account.startswith("INVALID"):
            raise ToolException(f"Invalid account: {account}")
        return f"Transferred {amount} to {account}"
    except Exception as e:
        return f"Transfer failed: {str(e)}"

@tool
def check_balance(account: str) -> str:
    """Check account balance"""
    try:
        # Mock
        return f"Balance: 5000000 VND"
    except Exception as e:
        return f"Cannot check balance: {str(e)}"

# === Agent với checkpointer ===
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
agent = create_react_agent(
    llm,
    [transfer, check_balance],
    checkpointer=memory,
)

# === Robust invocation ===
def robust_invoke(question: str, max_attempts=3):
    config = {
        "configurable": {"thread_id": "session_1"},
        "recursion_limit": 10,
    }
    
    for attempt in range(max_attempts):
        try:
            result = agent.invoke(
                {"messages": [("user", question)]},
                config=config,
            )
            return result["messages"][-1].content
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_attempts - 1:
                return f"Failed after {max_attempts} attempts: {e}"

# Test
print(robust_invoke("Chuyển 1000 đến account 1234567890"))
print(robust_invoke("Chuyển 999999999 (vượt limit)"))  # Validation error
print(robust_invoke("Chuyển 100 đến INVALID_ACC"))     # Tool error
```

---

## 15. Best Practices

✅ **Defense in depth**: nhiều layer protection
- LLM retry/fallback
- Tool error handling
- Output validation
- Agent max iterations
- Outer try-catch

✅ **Fail safe**: agent fail → return helpful message

✅ **Log mọi error**: dễ debug sau

✅ **Idempotency**: tools nên safe to retry

✅ **User confirmation**: cho destructive actions (delete, send money)

✅ **Test edge cases**: empty input, malformed, malicious

---

## 16. Bài Tập

### Bài 1: Robust Math Agent
Math agent với:
- Validate division by zero
- Max iterations
- Fallback LLM
- Output parser với retry

Test với input lạ.

### Bài 2: API Agent Với Circuit Breaker
Agent gọi external API. Implement circuit breaker khi API fail 3 lần.

### Bài 3: Banking Agent
Agent xử lý transactions:
- Validate amounts
- Confirm destructive actions
- Rate limit per user
- Audit log mọi action

---

## 17. Checklist

- [ ] Retry với with_retry
- [ ] Fallback models
- [ ] Tool error handling (ToolException, try-except)
- [ ] OutputFixingParser
- [ ] Pydantic validation cho tool args
- [ ] Max iterations
- [ ] Timeout
- [ ] Rate limiting
- [ ] Circuit breaker
- [ ] Error tracking callbacks

➡️ **Tiếp theo**: [Debug & Tracing](./07-Debug-Tracing.md)
