# Output Formatting

## 1. Tại Sao Format Quan Trọng?

Output không cấu trúc → khó parse, không reliable:
```
"The user An is 25 years old and lives in Hanoi"
   → Khó extract programmatically
```

Output có cấu trúc → dùng được ngay:
```json
{"name": "An", "age": 25, "city": "Hanoi"}
```

---

## 2. Common Formats

| Format | Use case | Pros | Cons |
|--------|----------|------|------|
| **JSON** | API integration | Parse dễ | Đôi khi LLM sai escape |
| **XML** | Long structured | Claude thích | Verbose |
| **Markdown** | Display to user | Readable | Khó parse |
| **CSV** | Tabular | Compact | Khó với nested |
| **YAML** | Config | Human-friendly | Indentation tricky |
| **Plain text** | Free-form | Flexible | Không structure |

---

## 3. JSON Output

### 3.1 Basic JSON Prompt

```
Extract info from text. Output JSON ONLY (no other text, no markdown):

{
  "name": "string",
  "age": "integer or null",
  "email": "string or null"
}

Text: "Hi I'm An, age 25"
```

→ Output:
```json
{"name": "An", "age": 25, "email": null}
```

### 3.2 Strict JSON Schema

```
Output a JSON matching this exact schema:

{
  "type": "object",
  "required": ["name", "age"],
  "properties": {
    "name": {"type": "string", "minLength": 1},
    "age": {"type": "integer", "minimum": 0, "maximum": 150},
    "email": {"type": "string", "format": "email"}
  }
}

Rules:
- Output ONLY valid JSON
- No backticks, no markdown, no commentary
- If field unknown, use null (not omit)
- All strings in double quotes
```

### 3.3 LangChain Structured Output (Recommended)

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

class Person(BaseModel):
    name: str
    age: int = Field(ge=0, le=150)
    email: str | None = None

llm = ChatOpenAI(model="gpt-4o-mini")
structured = llm.with_structured_output(Person)

result = structured.invoke("Hi I'm An, 25 tuổi")
# result: Person(name='An', age=25, email=None) - typed!
```

### 3.4 JSON Mode (Native OpenAI)

```python
llm = ChatOpenAI(
    model="gpt-4o-mini",
    response_format={"type": "json_object"},
)

# Must mention "JSON" in prompt
response = llm.invoke(
    "Output JSON with keys: name, age. Text: Hi I'm An, 25."
)
# Always valid JSON guaranteed
```

---

## 4. XML Output (Claude Friendly)

Claude responds excellently to XML structure.

### 4.1 Pattern

```
Extract info. Output in XML:

<extraction>
  <name>...</name>
  <age>...</age>
  <email>...</email>
</extraction>

Text: {input}
```

### 4.2 Parse XML

```python
import re

def extract_xml(text, tag):
    pattern = f"<{tag}>(.*?)</{tag}>"
    match = re.search(pattern, text, re.DOTALL)
    return match.group(1).strip() if match else None

# Use
output = llm.invoke(prompt).content
name = extract_xml(output, "name")
age = int(extract_xml(output, "age"))
```

### 4.3 Or với BeautifulSoup

```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(output, "xml")
name = soup.find("name").text
```

---

## 5. Markdown Output

Cho output displayed to user.

```
Format output as Markdown:

# Title

## Section 1
Content...

## Section 2
- Bullet 1
- Bullet 2

## Code
```python
code here
```

## Table
| Col1 | Col2 |
|------|------|
| Val1 | Val2 |
```

---

## 6. CSV/TSV Output

```
Output as CSV with header:

Name,Age,City,Email
An,25,Hanoi,a@example.com
Binh,30,TPHCM,b@example.com
```

### Tips
- Use TSV (tabs) nếu values có commas
- Quote strings: `"value, with comma"`
- Escape: `""` cho quote trong value

---

## 7. YAML Output

```
Output as YAML:

name: An
age: 25
addresses:
  - city: Hanoi
    type: home
  - city: TPHCM
    type: office
preferences:
  - coffee
  - jazz
```

---

## 8. Mixed / Custom Formats

### 8.1 Numbered List

```
List 5 features. Use this exact format:

1. [Feature name]: [1-sentence description]
2. ...
```

### 8.2 Q&A Pairs

```
Generate FAQs. Format:

Q1: [Question]
A1: [Answer]

Q2: [Question]
A2: [Answer]
```

### 8.3 Comparison Table

```
Compare X, Y, Z across 5 criteria. Format:

| Criteria | X | Y | Z |
|----------|---|---|---|
| Speed    |...|...|...|
| Cost     |...|...|...|
| Quality  |...|...|...|

Winner: [X/Y/Z because...]
```

---

## 9. Format Compliance Tricks

### 9.1 Show Example Output

```
Generate response in this EXACT format:

EXAMPLE OUTPUT:
{
  "sentiment": "positive",
  "confidence": 0.95,
  "keywords": ["great", "love"]
}

Now process: "Sản phẩm tuyệt vời!"
Output:
```

### 9.2 "Only Output" Reminder

```
... your instructions ...

CRITICAL: Output ONLY the JSON. No explanation, no markdown, 
no "Here is the JSON:", no backticks. JUST raw JSON.
```

### 9.3 Open Prompt With Format

```
Task: ...

I'll start the output, you complete it:

{
  "result":
```

→ LLM completes JSON.

### 9.4 Pre-Fill Assistant

Anthropic API supports prefill:

```python
client.messages.create(
    model="claude-sonnet-4-6",
    messages=[
        {"role": "user", "content": "Output JSON..."},
        {"role": "assistant", "content": "{"},  # Pre-fill
    ]
)
```

LLM continues từ "{".

---

## 10. Handling Format Errors

### 10.1 Validation Layer

```python
import json

def parse_json_safely(text):
    # Strip markdown code blocks
    text = text.replace("```json", "").replace("```", "").strip()
    
    try:
        return json.loads(text)
    except json.JSONDecodeError as e:
        return {"error": str(e), "raw": text}
```

### 10.2 Retry Với Feedback

```python
def generate_json(prompt, schema, max_retries=3):
    for attempt in range(max_retries):
        output = llm.invoke(prompt).content
        try:
            data = json.loads(output)
            # Validate against schema
            jsonschema.validate(data, schema)
            return data
        except json.JSONDecodeError as e:
            prompt += f"\n\nPrevious output had JSON error: {e}. Fix and retry."
        except jsonschema.ValidationError as e:
            prompt += f"\n\nSchema validation failed: {e}. Fix and retry."
    
    raise Exception("Failed after retries")
```

### 10.3 OutputFixingParser

```python
from langchain.output_parsers import PydanticOutputParser, OutputFixingParser

base = PydanticOutputParser(pydantic_object=Person)
fixer = OutputFixingParser.from_llm(parser=base, llm=llm)

# Tự gọi LLM fix nếu parse fail
result = fixer.parse(bad_output)
```

---

## 11. Nested Structures

### Deeply Nested JSON

```
Extract resume info. Schema:

{
  "personal": {
    "name": "string",
    "contact": {
      "email": "string",
      "phone": "string"
    }
  },
  "experience": [
    {
      "company": "string",
      "title": "string",
      "duration_months": "integer",
      "responsibilities": ["string"]
    }
  ],
  "skills": {
    "technical": ["string"],
    "soft": ["string"]
  }
}
```

### Tip: Use Pydantic
```python
class Contact(BaseModel):
    email: str
    phone: str

class Personal(BaseModel):
    name: str
    contact: Contact

class Job(BaseModel):
    company: str
    title: str
    duration_months: int
    responsibilities: List[str]

class Skills(BaseModel):
    technical: List[str]
    soft: List[str]

class Resume(BaseModel):
    personal: Personal
    experience: List[Job]
    skills: Skills

result = llm.with_structured_output(Resume).invoke(text)
# Fully typed, nested object
```

---

## 12. Streaming Structured Output

JSON có thể stream partial:

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()
chain = prompt | llm | parser

async for partial in chain.astream(input):
    print(partial)
# {"name": "A"}
# {"name": "An"}
# {"name": "An", "age": 2}
# {"name": "An", "age": 25}
```

→ Tốt cho UI realtime.

---

## 13. Multi-Output In Single Prompt

### Pattern: Multiple Sections

```
Analyze this email. Output:

===SUMMARY===
[1-2 sentences]

===CATEGORY===
[one of: complaint, inquiry, feedback]

===URGENCY===
[low | medium | high]

===KEYWORDS===
[comma-separated]

===SUGGESTED_REPLY===
[3-5 sentences]
```

### Parse

```python
def parse_sections(text):
    import re
    sections = {}
    pattern = r"===(\w+)===\n(.*?)(?===|$)"
    for match in re.finditer(pattern, text, re.DOTALL):
        sections[match.group(1).lower()] = match.group(2).strip()
    return sections
```

---

## 14. Format Per Model

### OpenAI GPT-4o
- ✅ JSON mode native
- ✅ Function calling
- ✅ Markdown
- Format usually correct

### Anthropic Claude
- ✅ XML excellent
- ✅ Long structured outputs
- ✅ Pre-fill assistant
- ⚠️ JSON đôi khi extra commentary

### Gemini
- ✅ JSON mode native
- ✅ Strict schemas
- ✅ Multimodal output

### Llama 3
- ⚠️ Less reliable formatting
- ✅ Tốt với simple JSON
- ❌ Khó với deeply nested
- Tip: dùng few-shot examples nhiều hơn

---

## 15. Special Outputs

### 15.1 Code Output

```
Output ONLY code, no explanation:

```python
def hello():
    print("Hi")
```

Strip ``` markers khi parse:
```python
def extract_code(text, lang="python"):
    pattern = f"```{lang}?\\n(.*?)```"
    match = re.search(pattern, text, re.DOTALL)
    return match.group(1) if match else text
```

### 15.2 SQL Output

```
Generate SQL query (PostgreSQL syntax).
Output ONLY the SQL, no markdown, no explanation.
End with semicolon.

Schema: {schema}
Question: {question}

SELECT
```

### 15.3 Image Description Format

```
Describe image in JSON:

{
  "main_subject": "string",
  "scene": "string",
  "objects": ["string"],
  "colors": ["string"],
  "mood": "string",
  "estimated_time_period": "string",
  "estimated_location": "string"
}
```

---

## 16. Demo: Resume Parser

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import date
from langchain_openai import ChatOpenAI

class Contact(BaseModel):
    email: Optional[str] = None
    phone: Optional[str] = None
    linkedin: Optional[str] = None

class Experience(BaseModel):
    company: str
    title: str
    start_date: str
    end_date: Optional[str] = Field(None, description="None nếu đang làm")
    description: str
    
class Education(BaseModel):
    school: str
    degree: str
    field: str
    graduation_year: Optional[int] = None

class Resume(BaseModel):
    name: str
    contact: Contact
    summary: str = Field(description="2-3 sentence summary")
    experience: List[Experience]
    education: List[Education]
    skills: List[str]
    years_of_experience: int

llm = ChatOpenAI(model="gpt-4o-mini")
parser = llm.with_structured_output(Resume)

resume_text = """
Nguyễn Văn A
Email: an@example.com | Phone: 0901234567

Summary: 5 năm kinh nghiệm software engineering, chuyên Python và AI.

Experience:
- Senior Engineer at FPT (2022-present)
  Led AI team, built RAG systems
- Engineer at Tiki (2020-2022)
  Backend Python services

Education:
- BS Computer Science, HUST, 2020

Skills: Python, LangChain, Docker, AWS
"""

result = parser.invoke(resume_text)
print(result.model_dump_json(indent=2))
```

---

## 17. Best Practices

✅ **Use Pydantic + `with_structured_output`** cho production

✅ **JSON mode** khi available (OpenAI, Gemini)

✅ **Show example output** trong prompt

✅ **Validate sau khi parse** (schema, types)

✅ **Retry với feedback** khi fail

✅ **Streaming JsonOutputParser** cho UI realtime

✅ **XML cho Claude**, **JSON cho OpenAI/Gemini**

---

## 18. Bài Tập

### Bài 1: Format Comparison
Cùng task extraction, test 3 formats (JSON, XML, YAML). Đo:
- Compliance rate (% valid format)
- Parse success rate
- Token count

### Bài 2: Resume Parser
Build resume parser handle nhiều format CV khác nhau. Output Pydantic model.

### Bài 3: Self-Healing JSON
Implement retry với feedback. Test với 20 prompts, đo cải thiện rate.

---

## 19. Checklist

- [ ] JSON output (basic + schema)
- [ ] XML output cho Claude
- [ ] Markdown cho display
- [ ] Structured output với Pydantic
- [ ] Native JSON mode
- [ ] Streaming partial JSON
- [ ] Multi-section output
- [ ] Format error handling

➡️ **Tiếp theo**: [Task-Specific Prompts](./08-Task-Specific-Prompts.md)
