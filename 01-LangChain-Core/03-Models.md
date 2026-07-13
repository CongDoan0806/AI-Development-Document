# Models - Làm Việc Với LLM Trong LangChain

## 1. Các Loại Model Trong LangChain

LangChain hỗ trợ 3 loại model chính:

| Loại | Interface | Use case |
|------|-----------|----------|
| **Chat Model** | `BaseChatModel` | Hiện đại nhất, dùng message format |
| **LLM (Completion)** | `BaseLLM` | Cũ, dùng text completion |
| **Embedding Model** | `Embeddings` | Tạo vector cho text |

> 💡 **Nên dùng Chat Model** cho mọi use case mới.

---

## 2. Chat Model

### 2.1 OpenAI

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-4o-mini",       # hoặc gpt-4o, gpt-4-turbo
    temperature=0.7,
    max_tokens=500,
    api_key="...",             # hoặc dùng env OPENAI_API_KEY
)

response = llm.invoke("Hello")
print(response.content)
```

### 2.2 Anthropic Claude

```python
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(
    model="claude-sonnet-4-6",  # claude-opus-4-7, claude-haiku-4-5
    temperature=0.7,
    max_tokens=1000,
)

response = llm.invoke("Hello")
print(response.content)
```

### 2.3 Google Gemini

```python
from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(
    model="gemini-2.0-flash",
    temperature=0.7,
    google_api_key="...",
)
```

### 2.4 Local Models (Ollama)

```python
# Cài: pip install langchain-ollama
# Cài Ollama: https://ollama.ai
# Kéo model: ollama pull llama3

from langchain_ollama import ChatOllama

llm = ChatOllama(
    model="llama3",
    temperature=0.7,
)

response = llm.invoke("Hello")
```

### 2.5 Azure OpenAI

```python
from langchain_openai import AzureChatOpenAI

llm = AzureChatOpenAI(
    azure_endpoint="https://YOUR_RESOURCE.openai.azure.com",
    deployment_name="gpt-4o",
    api_version="2024-08-01-preview",
)
```

### 2.6 AWS Bedrock

```python
from langchain_aws import ChatBedrock

llm = ChatBedrock(
    model_id="anthropic.claude-3-sonnet-20240229-v1:0",
    region_name="us-east-1",
)
```

### 2.7 Cohere

Cohere là provider mạnh cho enterprise, có model `command` family và rerank model rất tốt.

```bash
pip install langchain-cohere cohere
```

Lấy API key tại: https://cohere.com/

```python
import os
os.environ["COHERE_API_KEY"] = "..."

# Completion LLM
from langchain_community.llms import Cohere

llm = Cohere(
    model="command",
    max_tokens=256,
    temperature=0.75,
)
response = llm.invoke("Xin chào")
print(response)  # Trả về string

# Chat model
from langchain_community.chat_models import ChatCohere

chat = ChatCohere(
    model="command",        # command, command-r, command-r-plus
    max_tokens=256,
    temperature=0.1,
)
response = chat.invoke("Xin chào")
print(response.content)  # AIMessage.content
```

**Models của Cohere**:
| Model | Đặc điểm |
|-------|----------|
| `command` | General purpose, cân bằng |
| `command-r` | Tốt cho RAG, tool use |
| `command-r-plus` | Bản nâng cao của command-r |
| `command-light` | Nhỏ, nhanh, rẻ |

**Lưu ý** cho Cohere:
- `Cohere` là legacy completion LLM (trả `str`)
- `ChatCohere` là chat model (trả `AIMessage`)
- Khi dùng với `initialize_agent` (legacy API) → nên dùng `Cohere`
- Khi dùng với LCEL hoặc LangGraph → ưu tiên `ChatCohere`

---

## 3. Cấu Trúc Input/Output Của Chat Model

### 3.1 Input: Messages

```python
from langchain_core.messages import (
    SystemMessage,
    HumanMessage,
    AIMessage,
    ToolMessage,
)

messages = [
    SystemMessage(content="Bạn là trợ lý AI."),
    HumanMessage(content="Hello!"),
    AIMessage(content="Chào bạn!"),
    HumanMessage(content="Bạn tên gì?"),
]

response = llm.invoke(messages)
```

**Tuple syntax (gọn hơn)**:
```python
response = llm.invoke([
    ("system", "Bạn là trợ lý AI."),
    ("user", "Hello!"),
    ("assistant", "Chào bạn!"),
    ("user", "Bạn tên gì?"),
])
```

### 3.2 Output: AIMessage

```python
response = llm.invoke("Hello")

print(type(response))           # AIMessage
print(response.content)         # Text response
print(response.usage_metadata)  # {input_tokens, output_tokens, total_tokens}
print(response.response_metadata)  # Model info, finish_reason
print(response.id)              # Unique ID
```

---

## 4. Parameter Quan Trọng

### 4.1 Temperature

```python
# Strict, deterministic
llm = ChatOpenAI(temperature=0)

# Balanced
llm = ChatOpenAI(temperature=0.7)

# Creative
llm = ChatOpenAI(temperature=1.5)
```

### 4.2 max_tokens

Giới hạn output:
```python
llm = ChatOpenAI(max_tokens=200)
```

### 4.3 top_p

Nucleus sampling (thay thế temperature):
```python
llm = ChatOpenAI(top_p=0.9)
```

### 4.4 frequency_penalty & presence_penalty

```python
llm = ChatOpenAI(
    frequency_penalty=0.5,  # Giảm lặp từ
    presence_penalty=0.3,   # Khuyến khích chủ đề mới
)
```

### 4.5 timeout & max_retries

```python
llm = ChatOpenAI(
    timeout=30,        # Timeout sau 30s
    max_retries=2,     # Retry 2 lần khi fail
)
```

---

## 5. Streaming

### 5.1 Stream Sync

```python
for chunk in llm.stream("Viết bài thơ về mùa thu"):
    print(chunk.content, end="", flush=True)
```

### 5.2 Stream Async

```python
import asyncio

async def stream_demo():
    async for chunk in llm.astream("Hello"):
        print(chunk.content, end="", flush=True)

asyncio.run(stream_demo())
```

### 5.3 Stream Events (Chi tiết hơn)

```python
async def stream_events():
    async for event in llm.astream_events("Hello", version="v2"):
        kind = event["event"]
        if kind == "on_chat_model_stream":
            print(event["data"]["chunk"].content, end="")
        elif kind == "on_chat_model_end":
            print(f"\n\nDone. Usage: {event['data']['output'].usage_metadata}")
```

---

## 6. Structured Output

Lấy output theo schema có sẵn (Pydantic).

### 6.1 Cách 1: with_structured_output

```python
from pydantic import BaseModel, Field
from typing import List

class Person(BaseModel):
    """Thông tin một người"""
    name: str = Field(description="Tên đầy đủ")
    age: int = Field(description="Tuổi")
    skills: List[str] = Field(description="Danh sách kỹ năng")

structured_llm = llm.with_structured_output(Person)

result = structured_llm.invoke("""
    Nguyễn Văn A, 25 tuổi, biết Python, JavaScript, và Docker.
""")
print(result)
# Person(name='Nguyễn Văn A', age=25, skills=['Python', 'JavaScript', 'Docker'])
print(type(result))  # <class '__main__.Person'>
print(result.skills)  # ['Python', 'JavaScript', 'Docker']
```

### 6.2 Cách 2: Function Calling (chi tiết)

```python
from typing import Literal

class JoinUsRequest(BaseModel):
    """Schema cho yêu cầu tham gia team"""
    
    name: str = Field(description="Tên người nộp")
    email: str = Field(description="Email liên hệ")
    role: Literal["developer", "designer", "manager"] = Field(
        description="Vai trò mong muốn"
    )
    experience_years: int = Field(description="Số năm kinh nghiệm")
    skills: List[str] = Field(description="Kỹ năng chính")

structured_llm = llm.with_structured_output(JoinUsRequest)

email_text = """
Xin chào, tôi là Trần Thị B, email b@example.com.
Tôi có 5 năm kinh nghiệm và muốn ứng tuyển vị trí developer.
Kỹ năng: Python, React, AWS.
"""

result = structured_llm.invoke(email_text)
print(result.model_dump_json(indent=2))
```

### 6.3 List Output

```python
class Article(BaseModel):
    title: str
    summary: str

class ArticleList(BaseModel):
    """Danh sách bài viết"""
    articles: List[Article]

structured_llm = llm.with_structured_output(ArticleList)
result = structured_llm.invoke("Liệt kê 3 bài viết về AI")
for article in result.articles:
    print(f"- {article.title}: {article.summary}")
```

### 6.4 Optional Fields

```python
from typing import Optional

class ContactInfo(BaseModel):
    name: str
    email: Optional[str] = None  # Có thể không có
    phone: Optional[str] = None

structured_llm = llm.with_structured_output(ContactInfo)
result = structured_llm.invoke("Tên: A. Phone: 0901234567")
print(result)  # ContactInfo(name='A', email=None, phone='0901234567')
```

---

## 7. Tool Calling (Function Calling)

LLM tự quyết định khi nào gọi function nào.

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Lấy thông tin thời tiết của một thành phố"""
    weather_data = {
        "Hà Nội": "30°C, nắng",
        "TPHCM": "33°C, mưa rào",
    }
    return weather_data.get(city, "Không có dữ liệu")

@tool
def get_population(city: str) -> int:
    """Lấy dân số của một thành phố"""
    return {"Hà Nội": 8_000_000, "TPHCM": 9_000_000}.get(city, 0)

# Bind tools vào LLM
llm_with_tools = llm.bind_tools([get_weather, get_population])

# LLM trả về tool call (chưa thực thi)
response = llm_with_tools.invoke("Hà Nội đang thế nào về thời tiết và dân số?")

print(response.tool_calls)
# [
#   {"name": "get_weather", "args": {"city": "Hà Nội"}, "id": "..."},
#   {"name": "get_population", "args": {"city": "Hà Nội"}, "id": "..."}
# ]

# Bạn cần execute tool và gửi lại kết quả (sẽ học sâu ở phase Agents)
```

---

## 8. Multi-Modal (Vision)

GPT-4o, Claude, Gemini hỗ trợ image input.

```python
import base64

def encode_image(image_path):
    with open(image_path, "rb") as f:
        return base64.b64encode(f.read()).decode()

image_base64 = encode_image("photo.jpg")

response = llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Mô tả hình ảnh này"},
        {
            "type": "image_url",
            "image_url": {"url": f"data:image/jpeg;base64,{image_base64}"}
        },
    ])
])
print(response.content)
```

**Với URL trực tiếp**:
```python
response = llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Mô tả hình này"},
        {"type": "image_url", "image_url": {"url": "https://example.com/img.jpg"}},
    ])
])
```

---

## 9. Caching - Giảm Chi Phí

### 9.1 In-Memory Cache

```python
from langchain.globals import set_llm_cache
from langchain.cache import InMemoryCache

set_llm_cache(InMemoryCache())

# Lần 1: gọi API
result1 = llm.invoke("Hello")  # Chậm

# Lần 2: lấy từ cache (cùng prompt)
result2 = llm.invoke("Hello")  # Tức thì
```

### 9.2 SQLite Cache (persistent)

```python
from langchain_community.cache import SQLiteCache

set_llm_cache(SQLiteCache(database_path=".langchain.db"))
```

### 9.3 Redis Cache

```python
from langchain_community.cache import RedisCache
from redis import Redis

set_llm_cache(RedisCache(redis_=Redis()))
```

### 9.4 Semantic Cache (cache theo nghĩa)

```python
from langchain.cache import RedisSemanticCache
from langchain_openai import OpenAIEmbeddings

set_llm_cache(RedisSemanticCache(
    redis_url="redis://localhost:6379",
    embedding=OpenAIEmbeddings(),
    score_threshold=0.95,  # Cache hit nếu similarity > 95%
))

llm.invoke("Hello, how are you?")    # Gọi API
llm.invoke("Hi, how do you do?")     # Cache hit (nghĩa tương tự)
```

---

## 10. Theo Dõi Chi Phí

### 10.1 callback get_openai_callback

```python
from langchain.callbacks import get_openai_callback

with get_openai_callback() as cb:
    response = llm.invoke("Hello")
    print(f"Total Tokens: {cb.total_tokens}")
    print(f"Prompt Tokens: {cb.prompt_tokens}")
    print(f"Completion Tokens: {cb.completion_tokens}")
    print(f"Total Cost (USD): ${cb.total_cost:.6f}")
```

### 10.2 Trong response.usage_metadata

```python
response = llm.invoke("Hello")
print(response.usage_metadata)
# {'input_tokens': 8, 'output_tokens': 9, 'total_tokens': 17}
```

---

## 11. Model Switching - Dễ Đổi Provider

```python
from langchain.chat_models import init_chat_model

# Function này tự động chọn provider phù hợp
llm_openai = init_chat_model("gpt-4o-mini", model_provider="openai")
llm_claude = init_chat_model("claude-sonnet-4-6", model_provider="anthropic")
llm_gemini = init_chat_model("gemini-2.0-flash", model_provider="google_genai")

# Cùng interface, đổi model dễ dàng
for model in [llm_openai, llm_claude, llm_gemini]:
    print(model.invoke("Hello").content)
```

---

## 12. Best Practices

### 12.1 Chọn Model Đúng

| Use case | Model gợi ý |
|----------|-------------|
| Q&A đơn giản | gpt-4o-mini, claude-haiku-4-5 |
| Code generation | claude-sonnet-4-6, gpt-4o |
| Reasoning phức tạp | claude-opus-4-7, gpt-4o |
| Multimodal (image) | gpt-4o, claude-opus-4-7, gemini-2.0 |
| Long context (1M+) | claude-opus-4-7 (1M), gemini-2.0 |
| Cheap, fast | gpt-4o-mini, claude-haiku-4-5 |

### 12.2 Set Timeout

```python
llm = ChatOpenAI(timeout=30, max_retries=2)
```

### 12.3 Dùng Async Khi Có Nhiều Call

```python
import asyncio

async def process_many(prompts):
    return await llm.abatch(prompts)

results = asyncio.run(process_many(["Q1", "Q2", "Q3"]))
```

### 12.4 Monitor Cost
```python
# Set LANGCHAIN_TRACING_V2=true để dùng LangSmith dashboard
# Hoặc dùng get_openai_callback() trong code
```

---

## 13. Bài Tập

### Bài 1: Multi-Model Comparison
Tạo function so sánh response của GPT-4o-mini, Claude Sonnet, và Gemini Flash với cùng 1 câu hỏi.

### Bài 2: Structured Output - Email Parser
Tạo Pydantic schema cho email và dùng `with_structured_output` để parse email thành cấu trúc:
- subject, from, to, body_summary, action_required (bool), urgency (low/medium/high)

### Bài 3: Vision App
Tạo app input ảnh hóa đơn → output JSON gồm: vendor, date, items (list), total.

---

## 14. Checklist

- [ ] Khởi tạo được Chat Model từ các provider
- [ ] Hiểu cấu trúc messages
- [ ] Biết tune temperature, max_tokens
- [ ] Streaming sync và async
- [ ] Structured output với Pydantic
- [ ] Tool calling cơ bản
- [ ] Multi-modal (vision)
- [ ] Setup caching
- [ ] Track cost

➡️ **Tiếp theo**: [Prompts](./04-Prompts.md)
