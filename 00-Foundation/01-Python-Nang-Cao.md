# Python Nâng Cao Cho AI

## 1. Tại Sao Cần Python Nâng Cao?

Khi xây dựng ứng dụng AI với LangChain, LangGraph, MCP, bạn sẽ gặp rất nhiều khái niệm Python nâng cao như:
- **async/await** - vì LLM API call rất chậm (vài giây), cần async để tối ưu
- **Type hints** - LangChain/Pydantic dùng type hints để validate
- **OOP** - các framework đều theo paradigm OOP
- **Decorator** - `@tool`, `@chain` là decorator
- **Generator** - dùng cho streaming response

---

## 2. Async/Await - Lập Trình Bất Đồng Bộ

### 2.1 Vấn Đề Của Code Đồng Bộ

```python
import time
import requests

def call_llm(prompt):
    # Giả sử mỗi call mất 3 giây
    time.sleep(3)
    return f"Response for: {prompt}"

# Code đồng bộ - mất 9 giây
start = time.time()
result1 = call_llm("Hello 1")
result2 = call_llm("Hello 2")
result3 = call_llm("Hello 3")
print(f"Total: {time.time() - start}s")  # ~9 giây
```

### 2.2 Giải Pháp Async

```python
import asyncio

async def call_llm_async(prompt):
    await asyncio.sleep(3)  # Mô phỏng LLM call
    return f"Response for: {prompt}"

async def main():
    start = time.time()
    # Chạy 3 call đồng thời
    results = await asyncio.gather(
        call_llm_async("Hello 1"),
        call_llm_async("Hello 2"),
        call_llm_async("Hello 3"),
    )
    print(f"Total: {time.time() - start}s")  # ~3 giây
    print(results)

asyncio.run(main())
```

### 2.3 Async Trong LangChain

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# Sync - gọi tuần tự
response = llm.invoke("Xin chào")

# Async - gọi bất đồng bộ
async def main():
    response = await llm.ainvoke("Xin chào")
    # Hoặc gọi nhiều prompt cùng lúc
    responses = await llm.abatch(["Câu 1", "Câu 2", "Câu 3"])
    return responses

# Streaming async
async def stream_demo():
    async for chunk in llm.astream("Kể chuyện cười"):
        print(chunk.content, end="", flush=True)
```

**Quy tắc nhớ**:
- Method bình thường: `invoke()`, `batch()`, `stream()`
- Async version: `ainvoke()`, `abatch()`, `astream()` (thêm chữ `a` đầu)

---

## 3. Type Hints - Gõ Kiểu

### 3.1 Type Hints Cơ Bản

```python
# Không có type hints (cách cũ)
def greet(name, age):
    return f"Hi {name}, you are {age}"

# Có type hints (khuyến nghị)
def greet(name: str, age: int) -> str:
    return f"Hi {name}, you are {age}"
```

### 3.2 Collection Types

```python
from typing import List, Dict, Tuple, Optional, Union

# List of strings
def get_names() -> List[str]:
    return ["An", "Binh", "Chi"]

# Dictionary
def get_user() -> Dict[str, any]:
    return {"name": "An", "age": 25}

# Optional = có thể là None
def find_user(id: int) -> Optional[Dict]:
    if id == 1:
        return {"name": "An"}
    return None

# Union = có thể là nhiều kiểu
def process(value: Union[str, int]) -> str:
    return str(value)

# Python 3.10+ syntax mới
def process(value: str | int) -> str:
    return str(value)
```

### 3.3 Pydantic Models (Dùng Nhiều Trong LangChain)

```python
from pydantic import BaseModel, Field

class UserQuery(BaseModel):
    """Schema cho user query"""
    question: str = Field(..., description="Câu hỏi của user")
    user_id: int = Field(..., description="ID của user")
    max_tokens: int = Field(default=500, description="Số token tối đa")

# Sử dụng
query = UserQuery(question="Hello", user_id=1)
print(query.question)  # "Hello"
print(query.model_dump())  # {"question": "Hello", "user_id": 1, "max_tokens": 500}

# Validate tự động
try:
    bad_query = UserQuery(question="Hi", user_id="not a number")
except Exception as e:
    print(e)  # ValidationError
```

---

## 4. OOP - Lập Trình Hướng Đối Tượng

### 4.1 Class Cơ Bản

```python
class ChatBot:
    def __init__(self, name: str, model: str = "gpt-4o-mini"):
        self.name = name
        self.model = model
        self.history = []
    
    def chat(self, message: str) -> str:
        self.history.append({"role": "user", "content": message})
        response = f"[{self.model}] Reply to: {message}"
        self.history.append({"role": "assistant", "content": response})
        return response
    
    def reset(self):
        self.history = []

bot = ChatBot("Assistant")
print(bot.chat("Hello"))
```

### 4.2 Kế Thừa

```python
class BaseAgent:
    def __init__(self, name: str):
        self.name = name
    
    def think(self, input: str) -> str:
        raise NotImplementedError("Subclass phải implement")

class SearchAgent(BaseAgent):
    def think(self, input: str) -> str:
        return f"Searching for: {input}"

class CodingAgent(BaseAgent):
    def think(self, input: str) -> str:
        return f"Writing code for: {input}"

agents = [SearchAgent("Searcher"), CodingAgent("Coder")]
for agent in agents:
    print(agent.think("Python tutorial"))
```

### 4.3 Abstract Base Class

```python
from abc import ABC, abstractmethod

class Tool(ABC):
    """Base class cho mọi tool"""
    
    @abstractmethod
    def name(self) -> str:
        pass
    
    @abstractmethod
    def execute(self, input: str) -> str:
        pass

class CalculatorTool(Tool):
    def name(self) -> str:
        return "calculator"
    
    def execute(self, input: str) -> str:
        return str(eval(input))

# Không thể tạo instance của abstract class
# tool = Tool()  # ERROR
calc = CalculatorTool()
print(calc.execute("2 + 3"))  # "5"
```

---

## 5. Decorator

### 5.1 Decorator Cơ Bản

```python
def log_call(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Done {func.__name__}")
        return result
    return wrapper

@log_call
def add(a, b):
    return a + b

add(1, 2)
# Output:
# Calling add
# Done add
```

### 5.2 Decorator Trong LangChain

```python
from langchain_core.tools import tool

@tool
def search_web(query: str) -> str:
    """Tìm kiếm web với query cho trước"""
    return f"Search results for: {query}"

@tool
def calculate(expression: str) -> str:
    """Tính toán biểu thức toán học"""
    return str(eval(expression))

# LangChain tự động biến function thành tool object
print(search_web.name)  # "search_web"
print(search_web.description)  # "Tìm kiếm web với query cho trước"
print(search_web.args_schema)  # Schema tự sinh từ type hints
```

---

## 6. Generator & Streaming

### 6.1 Generator Cơ Bản

```python
def count_down(n):
    while n > 0:
        yield n
        n -= 1

for i in count_down(3):
    print(i)
# 3, 2, 1
```

### 6.2 Streaming LLM Response

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# Streaming - in từng chunk khi LLM tạo ra
for chunk in llm.stream("Viết một bài thơ ngắn"):
    print(chunk.content, end="", flush=True)
```

---

## 7. Context Manager (with statement)

```python
# Tự động đóng file
with open("data.txt", "r") as f:
    data = f.read()
# File đã được đóng tự động

# LangChain callback context
from langchain.callbacks import get_openai_callback

with get_openai_callback() as cb:
    result = llm.invoke("Hello")
    print(f"Tokens used: {cb.total_tokens}")
    print(f"Cost: ${cb.total_cost}")
```

---

## 8. Bài Tập Thực Hành

### Bài 1: Async LLM Calls
Viết function `process_questions_async(questions: List[str])` gọi LLM cho mỗi câu hỏi và trả về list responses, sử dụng `asyncio.gather`.

### Bài 2: Pydantic Model
Tạo Pydantic model `Article` với các fields: `title`, `content`, `tags` (list), `published_at` (datetime), `views` (int, default 0).

### Bài 3: Custom Tool
Tạo custom tool bằng decorator `@tool`:
- `get_weather(city: str)` - trả về thời tiết giả lập
- `get_news(topic: str, limit: int = 5)` - trả về news giả lập

---

## 9. Checklist Kiến Thức

- [ ] Hiểu async/await và biết khi nào dùng
- [ ] Sử dụng được type hints với typing module
- [ ] Tạo được Pydantic model với validation
- [ ] OOP: class, kế thừa, abstract class
- [ ] Viết được decorator đơn giản
- [ ] Generator và streaming
- [ ] Context manager

➡️ **Tiếp theo**: [LLM Concepts](./02-LLM-Concepts.md)
