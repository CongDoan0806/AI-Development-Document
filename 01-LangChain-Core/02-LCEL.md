# LCEL - LangChain Expression Language

## 1. LCEL Là Gì?

**LCEL (LangChain Expression Language)** là cách viết chain trong LangChain bằng cú pháp `|` (pipe), tương tự Unix pipe.

```python
chain = prompt | llm | output_parser
result = chain.invoke({"question": "Hi"})
```

LCEL là **cách viết chuẩn** trong LangChain hiện đại (thay thế cho `LLMChain`, `SequentialChain` cũ).

---

## 2. Ưu Điểm Của LCEL

✅ **Streaming sẵn**: tất cả LCEL chain đều stream được
✅ **Async sẵn**: gọi `.ainvoke()` mà không cần code thêm
✅ **Batch sẵn**: gọi `.batch()` để xử lý nhiều input song song
✅ **Parallel**: dễ dàng chạy nhiều bước song song
✅ **Tracing**: tự động log lên LangSmith
✅ **Type safety**: dễ debug

---

## 3. Cú Pháp Cơ Bản

### 3.1 Pipe Operator

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Dịch '{text}' sang tiếng {language}")
llm = ChatOpenAI(model="gpt-4o-mini")
parser = StrOutputParser()

# Chain bằng pipe
chain = prompt | llm | parser

# Tương đương với:
# 1. prompt.invoke(input) → ChatPromptValue
# 2. llm.invoke(ChatPromptValue) → AIMessage
# 3. parser.invoke(AIMessage) → str

result = chain.invoke({"text": "Hello", "language": "Việt"})
print(result)  # "Xin chào"
```

### 3.2 So Sánh Với Cách Cũ

**Cách cũ (deprecated)**:
```python
from langchain.chains import LLMChain

chain = LLMChain(llm=llm, prompt=prompt)
result = chain.run({"text": "Hello", "language": "Việt"})
```

**Cách mới (LCEL)**:
```python
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"text": "Hello", "language": "Việt"})
```

---

## 4. Các Method Của Runnable

Mọi LCEL chain đều có các method này:

```python
chain = prompt | llm | parser

# Sync
chain.invoke(input)                  # 1 input
chain.batch([input1, input2, input3]) # Nhiều input
for chunk in chain.stream(input):    # Streaming
    print(chunk, end="")

# Async
await chain.ainvoke(input)
await chain.abatch([input1, input2])
async for chunk in chain.astream(input):
    print(chunk, end="")
```

### Demo Batch
```python
chain = prompt | llm | StrOutputParser()

inputs = [
    {"text": "Hello", "language": "Việt"},
    {"text": "Goodbye", "language": "Nhật"},
    {"text": "Thank you", "language": "Pháp"},
]

# Chạy 3 prompt song song
results = chain.batch(inputs)
for r in results:
    print(r)
```

---

## 5. RunnablePassthrough - Truyền Input Trực Tiếp

`RunnablePassthrough()` truyền input vào nguyên si, không thay đổi.

```python
from langchain_core.runnables import RunnablePassthrough

chain = RunnablePassthrough() | (lambda x: x.upper())
print(chain.invoke("hello"))  # "HELLO"

# Hữu ích khi cần thêm field vào dict
chain = RunnablePassthrough.assign(
    upper=lambda x: x["text"].upper()
)
print(chain.invoke({"text": "hello"}))
# {"text": "hello", "upper": "HELLO"}
```

---

## 6. RunnableLambda - Wrap Function

Wrap function Python thành Runnable:

```python
from langchain_core.runnables import RunnableLambda

def add_one(x: int) -> int:
    return x + 1

def double(x: int) -> int:
    return x * 2

# Tự động wrap khi dùng trong chain
chain = RunnableLambda(add_one) | RunnableLambda(double)
print(chain.invoke(5))  # (5+1)*2 = 12

# Hoặc dùng trực tiếp lambda
chain = (lambda x: x + 1) | RunnableLambda(double)
# LƯU Ý: bước đầu phải là RunnableLambda hoặc Runnable
```

---

## 7. RunnableParallel - Chạy Song Song

Chạy nhiều branch song song với cùng input:

```python
from langchain_core.runnables import RunnableParallel

# Tạo 2 chain riêng
joke_chain = (
    ChatPromptTemplate.from_template("Kể chuyện cười về {topic}")
    | llm | StrOutputParser()
)

poem_chain = (
    ChatPromptTemplate.from_template("Viết bài thơ về {topic}")
    | llm | StrOutputParser()
)

# Chạy song song
parallel = RunnableParallel(joke=joke_chain, poem=poem_chain)

# Hoặc syntax dict (tương đương)
parallel = {"joke": joke_chain, "poem": poem_chain}

result = parallel.invoke({"topic": "lập trình"})
print(result["joke"])
print(result["poem"])
```

**Lợi ích**: 2 LLM call chạy song song → tiết kiệm thời gian.

---

## 8. Composition Pattern Phổ Biến

### 8.1 RAG Chain

```python
from langchain_core.runnables import RunnablePassthrough

retriever = vectorstore.as_retriever()

rag_chain = (
    {
        "context": retriever,
        "question": RunnablePassthrough(),
    }
    | prompt
    | llm
    | StrOutputParser()
)

# input "AI là gì?" → 
# {"context": <retrieved docs>, "question": "AI là gì?"} → 
# prompt → llm → string
result = rag_chain.invoke("AI là gì?")
```

### 8.2 Multi-Step Processing

```python
# Bước 1: Trích xuất keyword
extract_prompt = ChatPromptTemplate.from_template(
    "Liệt kê 3 keyword từ câu: {text}"
)
extract_chain = extract_prompt | llm | StrOutputParser()

# Bước 2: Tìm kiếm với keyword
def search(keywords: str) -> list:
    # Giả sử search trên DB
    return [f"Result for: {keywords}"]

# Bước 3: Tổng hợp
summary_prompt = ChatPromptTemplate.from_template(
    "Tổng hợp kết quả: {results}"
)
summary_chain = summary_prompt | llm | StrOutputParser()

# Combine
full_chain = (
    extract_chain
    | RunnableLambda(search)
    | RunnableLambda(lambda r: {"results": "\n".join(r)})
    | summary_chain
)

result = full_chain.invoke({"text": "Tôi muốn học AI"})
```

---

## 9. Conditional Branching

### 9.1 RunnableBranch

```python
from langchain_core.runnables import RunnableBranch

def is_question(input: dict) -> bool:
    return "?" in input["text"]

def is_command(input: dict) -> bool:
    return input["text"].startswith("!")

question_chain = ChatPromptTemplate.from_template("Trả lời: {text}") | llm | StrOutputParser()
command_chain = ChatPromptTemplate.from_template("Thực hiện: {text}") | llm | StrOutputParser()
default_chain = ChatPromptTemplate.from_template("Phản hồi: {text}") | llm | StrOutputParser()

branch = RunnableBranch(
    (is_question, question_chain),
    (is_command, command_chain),
    default_chain,  # default
)

print(branch.invoke({"text": "Python là gì?"}))    # question_chain
print(branch.invoke({"text": "!shutdown"}))         # command_chain
print(branch.invoke({"text": "Tôi đói"}))           # default_chain
```

### 9.2 Routing Bằng Lambda

```python
def route(input):
    if "code" in input["question"].lower():
        return code_chain
    else:
        return general_chain

chain = (
    {"question": RunnablePassthrough()}
    | RunnableLambda(route)
)
```

---

## 10. Configurable Chains

Cho phép user chỉnh config tại runtime:

```python
from langchain_core.runnables import ConfigurableField

llm = ChatOpenAI(model="gpt-4o-mini").configurable_fields(
    temperature=ConfigurableField(
        id="temp",
        name="Temperature",
        description="LLM temperature",
    )
)

chain = prompt | llm | StrOutputParser()

# Default
result1 = chain.invoke({"topic": "AI"})

# Chỉnh temperature lúc invoke
result2 = chain.invoke(
    {"topic": "AI"},
    config={"configurable": {"temp": 0.9}}
)
```

---

## 11. Retry & Fallback

### 11.1 with_retry

```python
chain_with_retry = (prompt | llm | parser).with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True,
)
```

### 11.2 with_fallbacks

```python
primary = ChatOpenAI(model="gpt-4o", temperature=0)
fallback = ChatOpenAI(model="gpt-4o-mini", temperature=0)

llm_with_fallback = primary.with_fallbacks([fallback])

chain = prompt | llm_with_fallback | parser
# Nếu gpt-4o fail → tự động dùng gpt-4o-mini
```

---

## 12. Demo Tổng Hợp: Chain Phức Tạp

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableParallel

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# 1. Sinh ra outline cho bài viết
outline_prompt = ChatPromptTemplate.from_template(
    "Tạo outline 5 mục cho bài viết về chủ đề: {topic}"
)
outline_chain = outline_prompt | llm | StrOutputParser()

# 2. Sinh tiêu đề
title_prompt = ChatPromptTemplate.from_template(
    "Tạo tiêu đề hấp dẫn cho chủ đề: {topic}"
)
title_chain = title_prompt | llm | StrOutputParser()

# 3. Sinh introduction
intro_prompt = ChatPromptTemplate.from_template(
    "Viết đoạn mở bài 50 từ cho chủ đề: {topic}"
)
intro_chain = intro_prompt | llm | StrOutputParser()

# 4. Parallel: chạy 3 chain cùng lúc
parallel_chain = RunnableParallel(
    title=title_chain,
    outline=outline_chain,
    intro=intro_chain,
)

# 5. Format kết quả
def format_article(data: dict) -> str:
    return f"""
# {data['title']}

{data['intro']}

## Outline
{data['outline']}
"""

final_chain = parallel_chain | RunnableLambda(format_article)

# Chạy
result = final_chain.invoke({"topic": "Học AI hiệu quả"})
print(result)
```

---

## 13. Debug LCEL Chain

### 13.1 Inspect Schema

```python
print(chain.input_schema.schema())
print(chain.output_schema.schema())
```

### 13.2 Get Graph (Visualization)

```python
chain.get_graph().print_ascii()
```

### 13.3 Verbose Logging

```python
from langchain.globals import set_verbose, set_debug

set_verbose(True)  # In ra mỗi bước
set_debug(True)    # Verbose hơn nữa

result = chain.invoke(...)
```

### 13.4 LangSmith Tracing

Set env `LANGCHAIN_TRACING_V2=true` → mọi call tự log lên LangSmith. Click vào trace để xem từng bước.

---

## 14. Bài Tập

### Bài 1: Translation Chain
Tạo chain dịch text qua 3 ngôn ngữ (Anh → Việt → Nhật) và in từng bước.

### Bài 2: Question Classifier + Router
Tạo chain phân loại câu hỏi (code, food, weather) và route đến chain phù hợp.

### Bài 3: Parallel Processing
Cho 1 đoạn text, dùng RunnableParallel để cùng lúc:
- Tóm tắt
- Trích keyword
- Phân tích sentiment
- Dịch sang tiếng Anh

---

## 15. Checklist

- [ ] Hiểu pipe operator `|`
- [ ] Dùng được `invoke`, `batch`, `stream`, async versions
- [ ] Hiểu RunnablePassthrough, RunnableLambda
- [ ] Tạo được RunnableParallel
- [ ] Conditional branching với RunnableBranch
- [ ] Setup retry và fallback
- [ ] Debug được LCEL chain

➡️ **Tiếp theo**: [Models](./03-Models.md)
