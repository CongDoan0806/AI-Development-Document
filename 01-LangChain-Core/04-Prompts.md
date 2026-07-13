# Prompts - Quản Lý Prompt Trong LangChain

## 1. Tại Sao Cần Prompt Template?

**Vấn đề khi hardcode prompt**:
```python
question = "Python là gì?"
prompt = "Bạn là chuyên gia. Trả lời: " + question  # Khó maintain
```

**Với PromptTemplate**:
```python
template = "Bạn là chuyên gia. Trả lời: {question}"
prompt = template.format(question="Python là gì?")
```

**Lợi ích**:
- Tái sử dụng template
- Tách logic và content
- Validate input (Pydantic)
- Versioning prompt (LangSmith)
- Few-shot examples dễ dàng

---

## 2. PromptTemplate Cơ Bản

### 2.1 String Template

```python
from langchain_core.prompts import PromptTemplate

template = PromptTemplate.from_template(
    "Dịch câu sau sang tiếng {language}: {text}"
)

prompt = template.invoke({"language": "Việt", "text": "Hello world"})
print(prompt)  # "Dịch câu sau sang tiếng Việt: Hello world"
```

### 2.2 Chat Template (Khuyên dùng)

```python
from langchain_core.prompts import ChatPromptTemplate

template = ChatPromptTemplate.from_messages([
    ("system", "Bạn là chuyên gia {domain}. Trả lời ngắn gọn."),
    ("user", "{question}"),
])

result = template.invoke({
    "domain": "Python",
    "question": "Decorator là gì?"
})
print(result.messages)
# [SystemMessage('Bạn là chuyên gia Python...'), HumanMessage('Decorator là gì?')]
```

### 2.3 Single Message Template

```python
# Cách 1: from_template (chỉ user message)
template = ChatPromptTemplate.from_template("Hello {name}")

# Cách 2: from_messages với tuple
template = ChatPromptTemplate.from_messages([
    ("user", "Hello {name}")
])

# Cách 3: from_messages với object
from langchain_core.messages import HumanMessage
template = ChatPromptTemplate.from_messages([
    HumanMessage(content="Hello {name}")  # LƯU Ý: không format được!
])
```

---

## 3. Multi-Variable Template

```python
template = ChatPromptTemplate.from_messages([
    ("system", """Bạn là trợ lý AI chuyên về {topic}.
Style: {style}
Ngôn ngữ: {language}"""),
    ("user", "{question}"),
])

result = template.invoke({
    "topic": "lập trình",
    "style": "vui vẻ, dí dỏm",
    "language": "tiếng Việt",
    "question": "Why is Python slow?"
})
```

---

## 4. MessagesPlaceholder - Chèn History

```python
from langchain_core.prompts import MessagesPlaceholder

template = ChatPromptTemplate.from_messages([
    ("system", "Bạn là trợ lý AI."),
    MessagesPlaceholder("history"),  # Sẽ chèn các message vào đây
    ("user", "{question}"),
])

from langchain_core.messages import HumanMessage, AIMessage

history = [
    HumanMessage(content="Tên tôi là An"),
    AIMessage(content="Chào An!"),
]

result = template.invoke({
    "history": history,
    "question": "Bạn nhớ tên tôi không?"
})

# Output messages:
# 1. SystemMessage: Bạn là trợ lý AI.
# 2. HumanMessage: Tên tôi là An
# 3. AIMessage: Chào An!
# 4. HumanMessage: Bạn nhớ tên tôi không?
```

---

## 5. Few-Shot Prompting

Cho LLM xem vài ví dụ để học cách trả lời.

### 5.1 FewShotPromptTemplate

```python
from langchain_core.prompts import (
    FewShotPromptTemplate,
    PromptTemplate,
)

# Examples
examples = [
    {"input": "happy", "output": "vui vẻ"},
    {"input": "sad", "output": "buồn"},
    {"input": "angry", "output": "tức giận"},
]

# Format mỗi example
example_prompt = PromptTemplate(
    input_variables=["input", "output"],
    template="Input: {input}\nOutput: {output}"
)

# Combine
few_shot = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    prefix="Dịch từ tiếng Anh sang tiếng Việt:",
    suffix="Input: {word}\nOutput:",
    input_variables=["word"],
)

print(few_shot.format(word="excited"))
```

Output:
```
Dịch từ tiếng Anh sang tiếng Việt:

Input: happy
Output: vui vẻ

Input: sad
Output: buồn

Input: angry
Output: tức giận

Input: excited
Output:
```

### 5.2 Few-Shot Cho Chat Model

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate

examples = [
    {"input": "2 + 2", "output": "4"},
    {"input": "5 * 3", "output": "15"},
]

example_prompt = ChatPromptTemplate.from_messages([
    ("user", "{input}"),
    ("assistant", "{output}"),
])

few_shot = FewShotChatMessagePromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
)

final = ChatPromptTemplate.from_messages([
    ("system", "Bạn là máy tính. Chỉ trả số."),
    few_shot,
    ("user", "{question}"),
])

result = final.invoke({"question": "10 - 3"})
# Sẽ gồm: system → 2 example pairs → user "10 - 3"
```

### 5.3 Dynamic Example Selection

Chỉ chọn examples relevant với query:

```python
from langchain_community.example_selectors import SemanticSimilarityExampleSelector
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma

# Pool nhiều examples
examples = [
    {"input": "Việt Nam có bao nhiêu tỉnh thành?", "output": "63"},
    {"input": "Thủ đô Việt Nam?", "output": "Hà Nội"},
    {"input": "2 + 2", "output": "4"},
    {"input": "100 * 5", "output": "500"},
    {"input": "Python là ngôn ngữ gì?", "output": "Lập trình"},
]

selector = SemanticSimilarityExampleSelector.from_examples(
    examples,
    OpenAIEmbeddings(),
    Chroma,
    k=2,  # Chọn 2 examples gần nhất
)

# Test
result = selector.select_examples({"input": "TPHCM có gì nổi tiếng?"})
# Sẽ chọn 2 examples về Việt Nam thay vì toán
```

---

## 6. Partial Variables

Đặt sẵn 1 vài variable, để sau mới fill phần còn lại.

```python
template = PromptTemplate.from_template(
    "Tên: {name}\nNgày: {date}\nNội dung: {content}"
)

# Partial: cố định date
from datetime import datetime
partial = template.partial(date=datetime.now().strftime("%Y-%m-%d"))

# Giờ chỉ cần fill name + content
result = partial.invoke({"name": "An", "content": "Hi"})
```

**Partial bằng function (lazy)**:
```python
def get_today():
    return datetime.now().strftime("%Y-%m-%d")

template = PromptTemplate(
    input_variables=["content"],
    template="Date: {date}, Content: {content}",
    partial_variables={"date": get_today}  # Gọi lúc render
)

print(template.invoke({"content": "Hello"}))
```

---

## 7. Loading Prompt Từ File

### 7.1 YAML

```yaml
# prompt.yaml
_type: prompt
input_variables: ["topic", "question"]
template: |
  Bạn là chuyên gia về {topic}.
  Câu hỏi: {question}
```

```python
from langchain_core.prompts import load_prompt

prompt = load_prompt("prompt.yaml")
print(prompt.format(topic="AI", question="LLM là gì?"))
```

### 7.2 JSON

```json
{
    "_type": "prompt",
    "input_variables": ["topic"],
    "template": "Giải thích {topic}"
}
```

```python
prompt = load_prompt("prompt.json")
```

### 7.3 LangChain Hub

```python
from langchain import hub

# Pull prompt từ community
prompt = hub.pull("rlm/rag-prompt")
print(prompt)
```

---

## 8. Prompt Composition - Ghép Prompt

### 8.1 Pipe Prompt

```python
from langchain_core.prompts import PromptTemplate

intro = PromptTemplate.from_template("Bạn là {role}.")
question = PromptTemplate.from_template("Trả lời: {question}")

# Cách 1: format thủ công
combined = PromptTemplate.from_template(
    intro.template + "\n" + question.template
)

# Cách 2: pipeline với LCEL (cho ChatPromptTemplate)
template = ChatPromptTemplate.from_messages([
    ("system", "Bạn là {role}."),
    ("user", "Trả lời: {question}"),
])
```

### 8.2 PipelinePromptTemplate (Nâng cao)

```python
from langchain_core.prompts import PipelinePromptTemplate

intro_prompt = PromptTemplate.from_template("Bạn là {role}.")
example_prompt = PromptTemplate.from_template("Ví dụ: {example}")
question_prompt = PromptTemplate.from_template("Q: {question}\nA:")

full_template = """{intro}

{examples}

{question}"""

full_prompt = PromptTemplate.from_template(full_template)

pipeline = PipelinePromptTemplate(
    final_prompt=full_prompt,
    pipeline_prompts=[
        ("intro", intro_prompt),
        ("examples", example_prompt),
        ("question", question_prompt),
    ],
)

result = pipeline.format(
    role="chuyên gia Python",
    example="def hello(): print('Hi')",
    question="Đây là gì?"
)
print(result)
```

---

## 9. Prompting Techniques (Quan Trọng!)

### 9.1 Chain-of-Thought (CoT)

Yêu cầu LLM suy luận từng bước:

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", """Khi giải toán, hãy suy nghĩ từng bước (think step by step) 
trước khi đưa ra đáp án cuối cùng."""),
    ("user", """Q: An có 5 quả táo. Anh ấy cho Bình 2 quả, sau đó mua thêm 6 quả. 
Hỏi An có bao nhiêu quả?"""),
])

# LLM sẽ trả lời:
# "Bước 1: An có 5 quả
#  Bước 2: Cho Bình 2 → còn 3 quả
#  Bước 3: Mua thêm 6 → có 9 quả
#  Đáp án: 9 quả"
```

### 9.2 Zero-Shot CoT

Đơn giản thêm "Let's think step by step":

```python
prompt = "Q: {question}\n\nA: Let's think step by step."
```

### 9.3 Role Playing

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là Senior Python Engineer với 15 năm kinh nghiệm,
chuyên về performance optimization và clean code.
Khi review code, bạn:
- Chỉ ra điểm yếu cụ thể
- Đưa ra giải pháp với code example
- Giải thích tại sao
"""),
    ("user", "Review code này: {code}"),
])
```

### 9.4 Self-Consistency

Gọi LLM nhiều lần và lấy kết quả phổ biến nhất:

```python
results = llm.batch([prompt] * 5)  # Gọi 5 lần
# Sau đó majority vote
```

### 9.5 ReAct (Reasoning + Acting)

```python
react_prompt = """Trả lời câu hỏi sau bằng cách dùng các tool có sẵn.

Có format sau:
Thought: bạn nghĩ gì
Action: tool_name
Action Input: input cho tool
Observation: kết quả từ tool
... (lặp lại Thought/Action/Observation)
Final Answer: câu trả lời cuối

Question: {question}
{agent_scratchpad}"""
```

(Sẽ học sâu ở Phase 3 - Agents)

### 9.6 Constitutional AI / Output Constraints

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", """Khi trả lời, tuân thủ các quy tắc:
1. KHÔNG đưa ra lời khuyên y tế cụ thể
2. KHÔNG tiết lộ thông tin cá nhân
3. CHỈ trả lời bằng tiếng Việt
4. Nếu không biết, nói "Tôi không có thông tin về việc này"
5. Format: bullet point, mỗi point < 50 từ"""),
    ("user", "{question}"),
])
```

---

## 10. Prompt Engineering Best Practices

### 10.1 Cấu Trúc Prompt Tốt

```
[ROLE] - Bạn là gì
[CONTEXT] - Background information
[TASK] - Yêu cầu cụ thể
[CONSTRAINTS] - Giới hạn, quy tắc
[EXAMPLES] - Few-shot examples (optional)
[OUTPUT FORMAT] - Format mong muốn
[INPUT] - Input từ user
```

**Ví dụ**:
```python
prompt = """[ROLE]
Bạn là chuyên gia phân tích dữ liệu.

[CONTEXT]
Bạn đang phân tích dữ liệu bán hàng năm 2024 của công ty.

[TASK]
Phân tích dữ liệu sau và đưa ra 3 insight quan trọng.

[CONSTRAINTS]
- Tiếng Việt
- Mỗi insight < 100 từ
- Có số liệu cụ thể
- Đề xuất action

[OUTPUT FORMAT]
1. **Insight 1**: ...
   - Số liệu: ...
   - Action: ...

[INPUT]
{data}
"""
```

### 10.2 Tips Quan Trọng

✅ **Specific & explicit**: "Trả lời < 100 từ" thay vì "Trả lời ngắn"

✅ **Examples > Description**: cho 1 example tốt hơn 10 dòng mô tả

✅ **Output format rõ ràng**: nói chính xác format (JSON, list, table)

✅ **Negative constraints**: "KHÔNG dùng từ X", "TRÁNH ..."

✅ **Test edge cases**: thử với input lạ

❌ **Tránh ambiguous**: "Trả lời tốt" - tốt theo nghĩa nào?

❌ **Quá dài**: prompt > 5000 token có thể giảm chất lượng

---

## 11. Prompt Versioning Với LangSmith Hub

```python
from langchain import hub

# Push prompt lên hub
my_prompt = ChatPromptTemplate.from_template("Hello {name}")
hub.push("username/my-prompt", my_prompt)

# Pull prompt từ hub
prompt = hub.pull("username/my-prompt")
```

Lợi ích:
- Versioning (rollback dễ)
- Team share prompt
- A/B test prompt
- Track prompt changes

---

## 12. Bài Tập

### Bài 1: Customer Support Prompt
Tạo prompt template cho customer support bot:
- Variables: company_name, product, customer_question, conversation_history
- Constraints: lịch sự, không hứa hẹn, chỉ trả lời trong scope của product

### Bài 2: Code Review Prompt
Tạo prompt review code Python với:
- Role: Senior dev
- Output format: list các issue với severity + giải pháp
- Few-shot: 1 example về code có bug và review

### Bài 3: Multi-Language Translator
Tạo translator với dynamic example selection:
- Pool 20 cặp dịch Anh-Việt
- Chọn 3 examples gần nhất với input
- Output: JSON {translation, confidence, alternative}

---

## 13. Checklist

- [ ] Tạo được PromptTemplate và ChatPromptTemplate
- [ ] Sử dụng MessagesPlaceholder cho history
- [ ] Few-shot prompting cơ bản và dynamic
- [ ] Partial variables
- [ ] Loading prompt từ file/hub
- [ ] Hiểu các technique: CoT, Role-playing, ReAct
- [ ] Apply prompt engineering best practices

➡️ **Tiếp theo**: [Output Parsers](./05-Output-Parsers.md)
