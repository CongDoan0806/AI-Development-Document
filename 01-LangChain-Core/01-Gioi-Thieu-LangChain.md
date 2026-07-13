# Giới Thiệu LangChain

## 1. LangChain Là Gì?

**LangChain** là framework Python (và JS) giúp xây dựng ứng dụng dùng LLM một cách nhanh chóng và có cấu trúc.

**Vấn đề LangChain giải quyết**:
- Code LLM trực tiếp → lặp lại nhiều, khó maintain
- Mỗi LLM provider có API khác nhau → khó đổi
- Khó kết hợp nhiều bước: prompt → LLM → parse → tool call → ...
- Khó tích hợp với vector DB, document loader, memory

**LangChain cung cấp**:
- **Models**: interface chung cho mọi LLM
- **Prompts**: quản lý prompt template
- **Chains**: kết hợp các bước thành pipeline
- **Agents**: LLM tự quyết định tool nào gọi
- **Memory**: lưu lịch sử hội thoại
- **Retrievers**: kết nối vector DB
- **Tools**: function LLM có thể gọi

---

## 2. Cài Đặt

```bash
# Core
pip install langchain langchain-core

# Provider (chọn theo nhu cầu)
pip install langchain-openai      # OpenAI
pip install langchain-anthropic   # Claude
pip install langchain-google-genai # Gemini
pip install langchain-community   # Các tools cộng đồng

# Vector store
pip install langchain-chroma
pip install langchain-pinecone

# Document loaders
pip install pypdf python-docx unstructured
```

---

## 3. Kiến Trúc LangChain

```
┌─────────────────────────────────────────────┐
│           LangChain Application             │
├─────────────────────────────────────────────┤
│  Chains / Agents / LangGraph                │
├─────────────────────────────────────────────┤
│  Building Blocks:                           │
│  • Prompts        • Tools                   │
│  • Models         • Retrievers              │
│  • Output Parsers • Memory                  │
├─────────────────────────────────────────────┤
│  Integrations:                              │
│  • OpenAI/Anthropic/Google/...              │
│  • Chroma/Pinecone/FAISS/...                │
│  • PDF/Web/SQL/...                          │
└─────────────────────────────────────────────┘
```

**Các package chính (2026)**:
- `langchain-core`: abstractions cơ bản
- `langchain`: chains, agents, retrievers
- `langchain-community`: integrations cộng đồng
- `langchain-{provider}`: integrations chính thức (openai, anthropic, ...)
- `langgraph`: framework cho stateful agent
- `langsmith`: monitoring & evaluation platform

---

## 4. Hello World Đầu Tiên

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

load_dotenv()

# 1. Khởi tạo LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# 2. Gọi LLM
response = llm.invoke("Chào bạn! Bạn có khỏe không?")

# 3. Xem kết quả
print(response.content)
print(f"Tokens: {response.usage_metadata}")
```

---

## 5. Các Concept Chính

### 5.1 Runnable Interface

Tất cả thành phần LangChain đều implement interface `Runnable`:

```python
runnable.invoke(input)        # Sync, 1 input
runnable.batch([input1, input2]) # Sync, nhiều input
runnable.stream(input)        # Streaming
await runnable.ainvoke(input) # Async
await runnable.abatch([...])  # Async batch
async for chunk in runnable.astream(input): ... # Async streaming
```

→ Bạn có thể "nối" các Runnable lại với nhau bằng `|` (pipe).

### 5.2 LCEL (LangChain Expression Language)

```python
chain = prompt | llm | output_parser
result = chain.invoke({"topic": "AI"})
```

Đây là điểm mạnh lớn nhất của LangChain. Sẽ học sâu ở doc tiếp theo.

### 5.3 Document

Đại diện cho 1 đoạn text với metadata:

```python
from langchain_core.documents import Document

doc = Document(
    page_content="Nội dung document...",
    metadata={"source": "wiki", "page": 1}
)
```

### 5.4 Message Types

```python
from langchain_core.messages import (
    SystemMessage,   # role: "system"
    HumanMessage,    # role: "user"
    AIMessage,       # role: "assistant"
    ToolMessage,     # role: "tool"
)

messages = [
    SystemMessage("Bạn là trợ lý AI."),
    HumanMessage("Hello!"),
    AIMessage("Chào bạn!"),
    HumanMessage("Bạn tên gì?"),
]

response = llm.invoke(messages)
```

---

## 6. Use Cases Phổ Biến

### 6.1 Q&A Chatbot
LLM trả lời câu hỏi user (basic).

### 6.2 RAG (Retrieval-Augmented Generation)
LLM + private data: trả lời dựa trên document của bạn.

### 6.3 Summarization
Tóm tắt document/transcript.

### 6.4 Information Extraction
Trích xuất thông tin có cấu trúc từ text.

```python
# Ví dụ: trích xuất thông tin liên hệ từ email
from pydantic import BaseModel

class Contact(BaseModel):
    name: str
    email: str
    phone: str

llm_structured = llm.with_structured_output(Contact)
result = llm_structured.invoke("""
    Tôi là Nguyễn Văn A, email a@example.com, 
    số điện thoại 0901234567.
""")
print(result)  # Contact(name='Nguyễn Văn A', email='a@example.com', phone='0901234567')
```

### 6.5 Agent / Tool Calling
LLM tự quyết định tool nào để gọi.

### 6.6 Multi-Agent System
Nhiều agent phối hợp giải quyết task phức tạp.

---

## 7. So Sánh: LangChain vs Direct API

### Cách 1: Gọi OpenAI trực tiếp
```python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.choices[0].message.content)
```

### Cách 2: Dùng LangChain
```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
response = llm.invoke("Hello")
print(response.content)
```

**Ưu điểm LangChain**:
- API thống nhất cho mọi provider
- Dễ chain nhiều bước
- Có sẵn nhiều tools, retrievers, parsers
- Tích hợp tracing (LangSmith)

**Nhược điểm**:
- Abstraction nặng
- Update API thường xuyên (cần đọc docs)
- Có thể overkill cho task đơn giản

---

## 8. Setup Project Cấu Trúc Tốt

```
my-ai-app/
├── .env                    # API keys
├── .gitignore
├── pyproject.toml          # Dependencies
├── README.md
├── src/
│   ├── __init__.py
│   ├── config.py           # Load config
│   ├── llm.py              # LLM setup
│   ├── prompts/
│   │   └── system.py       # System prompts
│   ├── chains/
│   │   └── qa_chain.py
│   ├── agents/
│   │   └── tools.py
│   └── retrievers/
│       └── vector_store.py
├── tests/
└── data/                   # Documents để index
```

**Ví dụ `src/llm.py`**:
```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
import os

def get_llm(provider: str = "openai", **kwargs):
    if provider == "openai":
        return ChatOpenAI(
            model=kwargs.get("model", "gpt-4o-mini"),
            temperature=kwargs.get("temperature", 0.7),
        )
    elif provider == "anthropic":
        return ChatAnthropic(
            model=kwargs.get("model", "claude-sonnet-4-6"),
            temperature=kwargs.get("temperature", 0.7),
        )
    raise ValueError(f"Unknown provider: {provider}")
```

---

## 9. LangSmith - Tracing & Monitoring

LangSmith là platform monitor LangChain app. Setup:

```env
# .env
LANGCHAIN_API_KEY=lsv2_...
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=my-project
```

```python
# Code không cần thay đổi gì!
# Chỉ cần set env, tất cả call sẽ tự log lên LangSmith
llm = ChatOpenAI(model="gpt-4o-mini")
response = llm.invoke("Hello")

# Vào https://smith.langchain.com để xem trace
```

LangSmith cho phép:
- 👁️ Xem từng bước LLM call
- ⏱️ Đo latency
- 💰 Track cost
- 🐛 Debug khi có lỗi
- 📊 Tạo dataset để evaluation

---

## 10. Demo: Mini Q&A App

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

load_dotenv()

# 1. Tạo LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)

# 2. Tạo prompt template
prompt = ChatPromptTemplate.from_messages([
    ("system", "Bạn là chuyên gia về {topic}. Trả lời ngắn gọn, chính xác."),
    ("user", "{question}"),
])

# 3. Tạo chain bằng LCEL
chain = prompt | llm | StrOutputParser()

# 4. Chạy thử
result = chain.invoke({
    "topic": "Python",
    "question": "Decorator là gì?"
})
print(result)

# 5. Stream
for chunk in chain.stream({
    "topic": "AI",
    "question": "RAG là gì?"
}):
    print(chunk, end="", flush=True)
```

---

## 11. Roadmap Phase 1

Phase này (LangChain Core) bao gồm:
1. ✅ Giới thiệu LangChain (file này)
2. ⏭️ [LCEL - LangChain Expression Language](./02-LCEL.md)
3. ⏭️ [Models](./03-Models.md)
4. ⏭️ [Prompts](./04-Prompts.md)
5. ⏭️ [Output Parsers](./05-Output-Parsers.md)

---

## 12. Checklist

- [ ] Cài đặt được LangChain và package cần thiết
- [ ] Hiểu kiến trúc LangChain
- [ ] Gọi được LLM bằng LangChain
- [ ] Hiểu Runnable interface
- [ ] Hiểu các message types
- [ ] Setup LangSmith tracing
- [ ] Chạy thành công demo Q&A

➡️ **Tiếp theo**: [LCEL](./02-LCEL.md)
