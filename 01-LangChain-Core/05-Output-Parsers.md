# Output Parsers - Parse Output Của LLM

## 1. Tại Sao Cần Output Parser?

LLM trả về **text**, nhưng app cần **dữ liệu có cấu trúc** (dict, list, object).

**Vấn đề**:
```python
response = llm.invoke("Cho tôi 3 món ăn Việt Nam")
print(response.content)
# "1. Phở - món truyền thống...
#  2. Bún chả - đặc sản Hà Nội...
#  3. Bánh mì - đường phố..."

# Làm sao parse thành list?
```

**Giải pháp**: Output Parser
```python
parser = StructuredOutputParser(...)
result = parser.parse(response.content)
# [{"name": "Phở", ...}, {"name": "Bún chả", ...}, ...]
```

---

## 2. StrOutputParser - Cơ Bản

Lấy `.content` của AIMessage thành string:

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template("Dịch '{text}' sang tiếng Anh")
chain = prompt | llm | StrOutputParser()

result = chain.invoke({"text": "Xin chào"})
print(type(result))  # <class 'str'>
print(result)        # "Hello"
```

Không có StrOutputParser:
```python
chain = prompt | llm
result = chain.invoke({"text": "Xin chào"})
print(type(result))  # AIMessage
print(result.content)  # "Hello"
```

---

## 3. PydanticOutputParser - Structured Output

### 3.1 Cách Cơ Bản

```python
from pydantic import BaseModel, Field
from typing import List
from langchain_core.output_parsers import PydanticOutputParser

class Recipe(BaseModel):
    name: str = Field(description="Tên món ăn")
    ingredients: List[str] = Field(description="Danh sách nguyên liệu")
    cooking_time_minutes: int = Field(description="Thời gian nấu (phút)")

parser = PydanticOutputParser(pydantic_object=Recipe)

# Lấy format instructions
print(parser.get_format_instructions())
```

Output:
```
The output should be formatted as a JSON instance that conforms to the JSON schema below.

{"properties": {"name": {"description": "Tên món ăn", ...}, ...}}
```

### 3.2 Tích Hợp Vào Chain

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "{format_instructions}"),
    ("user", "Cho tôi công thức nấu {dish}")
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser

result = chain.invoke({"dish": "phở bò"})
print(type(result))      # <class '__main__.Recipe'>
print(result.name)       # "Phở bò"
print(result.ingredients)  # ["bánh phở", "thịt bò", ...]
print(result.cooking_time_minutes)  # 60
```

### 3.3 Khác Biệt với `with_structured_output`

```python
# Cách 1: PydanticOutputParser
chain = prompt | llm | parser
# → Dùng prompt engineering để buộc LLM trả JSON

# Cách 2: with_structured_output (KHUYÊN DÙNG)
structured_llm = llm.with_structured_output(Recipe)
result = structured_llm.invoke("Công thức phở bò")
# → Dùng tool calling của OpenAI/Anthropic - chính xác hơn
```

➡️ **Kết luận**: Ưu tiên `with_structured_output`. Chỉ dùng PydanticOutputParser khi model không hỗ trợ tool calling.

---

## 4. JsonOutputParser

Parse JSON đơn giản (không cần schema):

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", """Trả lời ở dạng JSON với format:
{{
    "answer": "câu trả lời",
    "confidence": 0.0-1.0
}}"""),
    ("user", "{question}")
])

chain = prompt | llm | parser

result = chain.invoke({"question": "Việt Nam có bao nhiêu tỉnh?"})
print(result)
# {"answer": "63 tỉnh thành", "confidence": 1.0}
print(type(result))  # dict
```

**Với Pydantic schema**:
```python
parser = JsonOutputParser(pydantic_object=Recipe)
# Tương tự PydanticOutputParser nhưng trả về dict, không phải Recipe object
```

---

## 5. ListOutputParser

### 5.1 CommaSeparatedListOutputParser

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser

parser = CommaSeparatedListOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "{format_instructions}"),
    ("user", "Liệt kê 5 ngôn ngữ lập trình phổ biến")
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser

result = chain.invoke({})
print(result)
# ['Python', 'JavaScript', 'Java', 'C++', 'Go']
```

### 5.2 NumberedListOutputParser

```python
from langchain.output_parsers import NumberedListOutputParser

parser = NumberedListOutputParser()
# Parse:
# "1. Item 1\n2. Item 2\n3. Item 3"
# → ["Item 1", "Item 2", "Item 3"]
```

---

## 6. StructuredOutputParser (Schema Đơn Giản)

Khi không muốn dùng Pydantic:

```python
from langchain.output_parsers import StructuredOutputParser, ResponseSchema

response_schemas = [
    ResponseSchema(name="title", description="Tiêu đề bài viết"),
    ResponseSchema(name="summary", description="Tóm tắt 50 từ"),
    ResponseSchema(name="tags", description="Tag, phân cách bằng dấu phẩy"),
]

parser = StructuredOutputParser.from_response_schemas(response_schemas)
format_instructions = parser.get_format_instructions()

prompt = ChatPromptTemplate.from_messages([
    ("system", "{format_instructions}"),
    ("user", "Phân tích bài viết: {article}")
]).partial(format_instructions=format_instructions)

chain = prompt | llm | parser

result = chain.invoke({"article": "..."})
print(result)
# {"title": "...", "summary": "...", "tags": "..."}
```

---

## 7. XMLOutputParser

Một số model trả về XML tốt hơn JSON (đặc biệt Claude):

```python
from langchain_core.output_parsers import XMLOutputParser

parser = XMLOutputParser(tags=["article", "title", "content"])

prompt = ChatPromptTemplate.from_template("""
Output ở format XML với tags: <article>, <title>, <content>.

Topic: {topic}
""")

chain = prompt | llm | parser

result = chain.invoke({"topic": "AI"})
# {"article": [{"title": "..."}, {"content": "..."}]}
```

---

## 8. DatetimeOutputParser

```python
from langchain.output_parsers import DatetimeOutputParser

parser = DatetimeOutputParser()
# format_instructions sẽ hướng dẫn LLM trả về datetime

prompt = ChatPromptTemplate.from_messages([
    ("system", "{format_instructions}"),
    ("user", "Khi nào Việt Nam độc lập?")
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser
result = chain.invoke({})
print(type(result))  # datetime
```

---

## 9. EnumOutputParser

Hạn chế output trong tập giá trị:

```python
from enum import Enum
from langchain.output_parsers import EnumOutputParser

class Sentiment(Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

parser = EnumOutputParser(enum=Sentiment)

prompt = ChatPromptTemplate.from_messages([
    ("system", "{format_instructions}"),
    ("user", "Phân tích sentiment: {text}")
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser

result = chain.invoke({"text": "Tôi rất vui!"})
print(result)  # Sentiment.POSITIVE
```

---

## 10. OutputFixingParser - Tự Sửa Lỗi

Khi LLM trả về JSON sai format, dùng LLM khác để fix:

```python
from langchain.output_parsers import OutputFixingParser

# Original parser
parser = PydanticOutputParser(pydantic_object=Recipe)

# Wrap với OutputFixingParser
fixing_parser = OutputFixingParser.from_llm(parser=parser, llm=llm)

# Khi parse fail, sẽ tự gọi LLM để fix
try:
    result = fixing_parser.parse(bad_json_text)
except Exception as e:
    print(f"Cannot fix: {e}")
```

---

## 11. RetryOutputParser

Khi parse fail, retry với prompt mới:

```python
from langchain.output_parsers import RetryOutputParser

parser = PydanticOutputParser(pydantic_object=Recipe)
retry_parser = RetryOutputParser.from_llm(parser=parser, llm=llm)

# Khi parse fail, sẽ:
# 1. Lấy original prompt + bad output
# 2. Gọi LLM với prompt "Hãy fix output này"
# 3. Parse lại
```

---

## 12. Streaming Với Output Parser

### 12.1 JsonOutputParser Streaming

JsonOutputParser support streaming partial JSON:

```python
parser = JsonOutputParser()
chain = prompt | llm | parser

# Stream: in dần các key/value khi LLM tạo ra
async for partial in chain.astream({"input": "..."}):
    print(partial)
# {"title": "He"}
# {"title": "Hello"}
# {"title": "Hello", "body": ""}
# {"title": "Hello", "body": "World"}
```

### 12.2 PydanticOutputParser Streaming

PydanticOutputParser KHÔNG stream được (cần full text). Dùng JsonOutputParser thay thế.

---

## 13. Custom Output Parser

```python
from langchain_core.output_parsers import BaseOutputParser
from typing import List

class CommaListParser(BaseOutputParser[List[str]]):
    """Parse text comma-separated thành list."""
    
    def parse(self, text: str) -> List[str]:
        return [item.strip() for item in text.split(",")]
    
    def get_format_instructions(self) -> str:
        return "Trả về danh sách, phân cách bằng dấu phẩy."

parser = CommaListParser()
chain = prompt | llm | parser
```

---

## 14. Demo Tổng Hợp: Email Analyzer

```python
from pydantic import BaseModel, Field
from typing import List, Literal
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class EmailAnalysis(BaseModel):
    """Phân tích nội dung email"""
    
    sentiment: Literal["positive", "negative", "neutral"] = Field(
        description="Cảm xúc tổng thể của email"
    )
    urgency: Literal["low", "medium", "high"] = Field(
        description="Mức độ khẩn cấp"
    )
    main_topics: List[str] = Field(
        description="Các chủ đề chính, tối đa 3"
    )
    action_required: bool = Field(
        description="Có cần action gì không"
    )
    suggested_response: str = Field(
        description="Đề xuất câu trả lời 1 câu"
    )
    estimated_response_time_minutes: int = Field(
        description="Ước tính thời gian cần để reply"
    )

# Dùng with_structured_output (preferred)
structured_llm = llm.with_structured_output(EmailAnalysis)

prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là expert phân tích email. 
Phân tích email sau và trả về thông tin có cấu trúc."""),
    ("user", "{email}")
])

chain = prompt | structured_llm

email_text = """
Hi team,

Khách hàng VIP báo lỗi không đăng nhập được vào hệ thống TỪ SÁNG NAY.
Đã có 5 user gọi điện complain. Cần fix gấp trong 1 giờ tới.

Best,
Manager
"""

result = chain.invoke({"email": email_text})
print(result.model_dump_json(indent=2))
```

Output:
```json
{
    "sentiment": "negative",
    "urgency": "high",
    "main_topics": ["login issue", "VIP customer", "urgent fix"],
    "action_required": true,
    "suggested_response": "Đã nhận thông tin, team đang xử lý ngay.",
    "estimated_response_time_minutes": 5
}
```

---

## 15. So Sánh Các Parser

| Parser | Use Case | Pros | Cons |
|--------|----------|------|------|
| `StrOutputParser` | Lấy plain text | Đơn giản nhất | Không cấu trúc |
| `JsonOutputParser` | Generic JSON | Stream được | Không type-safe |
| `PydanticOutputParser` | Schema strict | Type-safe | Không stream |
| `with_structured_output` | Production | Tin cậy nhất | Cần model support tool calling |
| `XMLOutputParser` | Claude models | Claude thích XML | Less common |
| `OutputFixingParser` | Wrap parser khác | Tự fix lỗi | Tốn extra LLM call |

---

## 16. Best Practices

✅ **Ưu tiên `with_structured_output`** cho production

✅ **Dùng `JsonOutputParser` khi cần streaming**

✅ **Test parser với edge cases**: empty input, format sai, content vô nghĩa

✅ **Validate sau khi parse**: Pydantic validate tự động, nhưng có thể thêm custom validator

✅ **Cache structured output** vì gọi LLM tốn tiền

---

## 17. Bài Tập

### Bài 1: Resume Parser
Tạo parser cho CV/resume → output Pydantic object:
- name, email, phone, skills, experience (list of jobs với title/company/duration)

### Bài 2: Streaming JSON Dashboard
Tạo app stream output JSON từ LLM và hiển thị từng phần khi xuất hiện.

### Bài 3: Self-Healing Parser
Dùng OutputFixingParser + custom validator để đảm bảo output luôn đúng format kể cả khi LLM lỗi.

---

## 18. Checklist

- [ ] Hiểu StrOutputParser, JsonOutputParser
- [ ] Dùng được PydanticOutputParser
- [ ] Biết khi nào dùng `with_structured_output`
- [ ] List, Enum, Datetime parser
- [ ] OutputFixingParser cho self-healing
- [ ] Streaming với JSON parser
- [ ] Tạo custom parser

➡️ **Phase 1 hoàn thành! Tiếp theo**: [Phase 2 - RAG](../02-RAG/01-Tong-Quan-RAG.md)
