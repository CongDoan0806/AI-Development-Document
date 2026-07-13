# Task-Specific Prompts

Templates và best practices cho các task phổ biến.

## 1. Summarization

### 1.1 Basic Summarization

```
Tóm tắt văn bản dưới đây thành {N} câu, tập trung vào điểm chính:

{text}

Tóm tắt:
```

### 1.2 Structured Summary

```
Tóm tắt bài viết theo cấu trúc:

## Main Topic
[1 sentence]

## Key Points
- Point 1
- Point 2
- Point 3

## Conclusion
[1 sentence]

Article:
{text}
```

### 1.3 Persona-Based Summary

```
Tóm tắt cho người không biết domain. Yêu cầu:
- Dùng simple language (Flesch grade 8)
- Không jargon, hoặc giải thích nếu phải dùng
- Có analogy với cuộc sống thường ngày
- Tối đa 100 từ

Text: {text}
```

### 1.4 Multi-Document Summary

```
Bạn có {N} documents cùng chủ đề. Hãy:
1. Tóm tắt mỗi doc 1 câu
2. Tìm common themes
3. Tìm contradictions/disagreements
4. Đưa ra unified summary

Documents:
[Doc 1] {doc1}
[Doc 2] {doc2}
...
```

### 1.5 Map-Reduce For Long Docs

```python
# Map: tóm tắt từng chunk
chunk_summaries = []
for chunk in chunks:
    s = llm.invoke(f"Tóm tắt 100 từ: {chunk}").content
    chunk_summaries.append(s)

# Reduce: combine
final = llm.invoke(f"""
Combine các summaries này thành 1 summary thống nhất:

{chunk_summaries}

Final summary:
""").content
```

---

## 2. Classification

### 2.1 Single-Label

```
Phân loại text vào 1 trong các category:
- complaint
- inquiry  
- feedback
- spam

Text: {text}

Output (CHỈ category name, không giải thích): 
```

### 2.2 Multi-Label

```
Text có thể thuộc nhiều category. Liệt kê TẤT CẢ phù hợp:

Categories: tech, business, education, health, sport

Text: {text}

Output (JSON list): ["tech", "business"]
```

### 2.3 Hierarchical

```
Phân loại text theo hierarchy:

Level 1: tech / business / sport
Level 2 (under tech): software / hardware / AI
Level 3 (under AI): NLP / vision / RL

Text: {text}

Output:
{
  "level_1": "...",
  "level_2": "...",
  "level_3": "..."
}
```

### 2.4 With Confidence

```
Classify + confidence:

Categories: positive | negative | neutral

Text: {text}

Output JSON:
{
  "label": "...",
  "confidence": 0.0-1.0,
  "reasoning": "1 sentence"
}
```

---

## 3. Information Extraction

### 3.1 Named Entities

```
Trích xuất entities:

Types: PERSON, ORGANIZATION, LOCATION, DATE, MONEY

Text: "An làm tại Google ở Mountain View từ 2020 với lương $150K"

Output JSON:
{
  "entities": [
    {"text": "An", "type": "PERSON"},
    {"text": "Google", "type": "ORGANIZATION"},
    {"text": "Mountain View", "type": "LOCATION"},
    {"text": "2020", "type": "DATE"},
    {"text": "$150K", "type": "MONEY"}
  ]
}
```

### 3.2 Relations

```
Extract relations giữa entities:

Text: {text}

Output:
{
  "relations": [
    {"subject": "An", "predicate": "works_at", "object": "Google"},
    {"subject": "Google", "predicate": "located_in", "object": "Mountain View"}
  ]
}
```

### 3.3 Form Extraction

```
Trích xuất từ document hóa đơn:

Output JSON:
{
  "vendor": "string",
  "date": "YYYY-MM-DD",
  "total_amount": number,
  "currency": "string",
  "items": [
    {"name": "string", "quantity": number, "unit_price": number}
  ],
  "tax": number,
  "subtotal": number
}

Document:
{ocr_text}
```

---

## 4. Translation

### 4.1 Basic

```
Dịch sang {target_language}:

{text}

Bản dịch:
```

### 4.2 Preserve Style

```
Dịch sang {target_language}. Yêu cầu:
- Giữ nguyên tone (formal/casual)
- Giữ cultural references (giải thích trong ngoặc nếu cần)
- Code/technical terms giữ tiếng Anh
- Idioms: chuyển sang idiom tương đương của target language

Text: {text}
```

### 4.3 Domain-Specific

```
Dịch document y khoa từ English sang Vietnamese.

Quy tắc:
- Giữ medical terminology tiếng Anh trong ngoặc: "tăng huyết áp (hypertension)"
- Tên thuốc giữ generic name English
- Dose, units giữ nguyên (mg, ml, IU)
- Style: formal, khoa học

Text: {text}
```

### 4.4 Translation With Explanation

```
Dịch + giải thích choice:

Text: "He kicked the bucket"

Output JSON:
{
  "literal_translation": "Anh ấy đá cái xô",
  "actual_meaning": "Anh ấy chết (idiom)",
  "recommended_translation": "Anh ấy qua đời",
  "context_notes": "..."
}
```

---

## 5. Code Generation

### 5.1 Function Generation

```
Viết Python function:

Name: {function_name}
Description: {description}
Inputs: {inputs}
Output: {output}

Requirements:
- Type hints
- Google-style docstring với Example
- Handle edge cases: empty input, None, invalid types
- No external dependencies (or list them)

Output ONLY code, no explanation:
```

### 5.2 Code With Tests

```
Viết Python function + unit tests:

Task: {task}

Output:
```python
# 1. Implementation
def solution(...):
    """..."""
    ...

# 2. Tests (pytest)
def test_basic():
    assert solution(...) == ...

def test_edge_case_empty():
    ...

def test_edge_case_invalid():
    ...
```
```

### 5.3 Refactor

```
Refactor code này:

Original:
{code}

Goals:
1. Improve readability
2. Reduce complexity
3. Add type hints
4. Extract magic numbers to constants

Output refactored code with brief comment about each change.
```

### 5.4 Code Review

```
Review code with focus on:

1. Correctness (bugs, edge cases)
2. Performance (algorithmic complexity)
3. Security (input validation, injection)
4. Maintainability (naming, structure)
5. Best practices (PEP 8, idioms)

Code:
{code}

Output:
## Critical Issues
- ...

## Major Issues
- ...

## Suggestions
- ...

## Verdict
APPROVE | NEEDS_CHANGES | REJECT
```

### 5.5 Bug Fix

```
Debug code này:

Code:
{code}

Error/Bug description:
{error}

Steps:
1. Identify root cause
2. Propose fix
3. Show fixed code
4. Suggest test to prevent regression

Output:
## Root Cause
[explanation]

## Fix
```python
[fixed code]
```

## Test
```python
[test case]
```
```

---

## 6. Q&A

### 6.1 Factual Q&A With Context

```
Trả lời câu hỏi DỰA TRÊN context. Nếu không có info, nói "Không có thông tin trong context".

Context:
{context}

Question: {question}

Answer:
```

### 6.2 Open-Ended Q&A

```
Trả lời câu hỏi sau. Cung cấp:
1. Direct answer (1-2 sentences)
2. Explanation (3-5 sentences)
3. Examples (if applicable)
4. Caveats / limitations

Question: {question}
```

### 6.3 Conversational Q&A

```
Trả lời câu hỏi follow-up dựa trên conversation:

Lịch sử:
{history}

Câu hỏi mới: {question}

Trả lời:
```

---

## 7. RAG (Retrieval-Augmented Generation)

### 7.1 Standard RAG

```
Bạn là trợ lý AI dựa trên context cho trước.

Quy tắc:
1. CHỈ dùng info trong context
2. Cite [Source N] sau mỗi claim
3. Nếu không có info → "Tôi không có thông tin"
4. KHÔNG suy đoán

Context:
[Source 1] {doc1}
[Source 2] {doc2}
...

Question: {question}

Answer (với citations):
```

### 7.2 RAG With Confidence

```
Trả lời với confidence score:

Context:
{context}

Question: {question}

Output JSON:
{
  "answer": "...",
  "confidence": 0.0-1.0,
  "supporting_quotes": ["...", "..."],
  "limitations": "what info is missing"
}
```

### 7.3 RAG With Hypothetical

```
Sử dụng technique HyDE (Hypothetical Document Embeddings):

Step 1: Generate hypothetical answer dựa trên general knowledge
Step 2: Match với retrieved docs
Step 3: Refine answer based on actual docs

Question: {question}
Retrieved docs: {docs}
```

---

## 8. Sentiment Analysis

### 8.1 Basic

```
Phân tích sentiment:

Text: {text}

Output JSON:
{
  "sentiment": "positive | negative | neutral",
  "score": -1.0 to 1.0,
  "confidence": 0.0 to 1.0
}
```

### 8.2 Aspect-Based

```
Phân tích sentiment theo từng aspect:

Text: "Sản phẩm tốt, giá rẻ, nhưng giao hàng chậm và đóng gói kém"

Aspects to check: product, price, delivery, packaging

Output:
{
  "product": {"sentiment": "positive", "evidence": "..."},
  "price": {"sentiment": "positive", "evidence": "..."},
  "delivery": {"sentiment": "negative", "evidence": "..."},
  "packaging": {"sentiment": "negative", "evidence": "..."}
}
```

### 8.3 Emotion Detection

```
Phân tích emotions có trong text:

Emotions: joy, sadness, anger, fear, surprise, disgust

Text: {text}

Output: top 3 emotions với intensity 0-1
```

---

## 9. Creative Writing

### 9.1 Story Generation

```
Viết short story:

Theme: {theme}
Genre: {genre}
Length: {n_words} words
Style: {style_reference}

Requirements:
- Vivid imagery
- Character development
- Clear beginning-middle-end
- Twist or revelation

Story:
```

### 9.2 Article Writing

```
Viết bài blog:

Topic: {topic}
Target audience: {audience}
Tone: {tone}
Word count: {n} words

Structure:
1. Hook (1 paragraph)
2. Thesis (1 sentence)
3. Body (3-5 sections with H2)
4. Conclusion (with call-to-action)

SEO:
- Include keywords: {keywords}
- Meta description: 150 chars

Output as Markdown.
```

### 9.3 Poetry

```
Write a poem:

Topic: {topic}
Form: {form} (sonnet/haiku/free verse/limerick)
Tone: {tone}
Required imagery: {imagery_hints}

Constraints:
- Adhere to form rules (rhyme, meter)
- Avoid clichés
- Original metaphors
```

---

## 10. Data Generation

### 10.1 Synthetic Data

```
Generate {N} synthetic data records.

Schema:
{
  "id": "integer (incrementing)",
  "name": "Vietnamese full name",
  "email": "realistic email",
  "age": "integer 18-70 (skewed normal around 35)",
  "income_vnd": "integer 5M-200M (log-normal)",
  "city": "Vietnamese city",
  "purchase_count": "integer (Poisson lambda=5)"
}

Constraints:
- Diverse names (cover regions)
- Realistic distributions
- No duplicates
- 70% Hanoi/TPHCM, 30% other cities

Output as JSON array.
```

### 10.2 Test Cases Generation

```
Generate test cases for function:

Function: {function_signature + description}

Coverage:
- Happy path (3 cases)
- Edge cases (5 cases)
- Error cases (3 cases)

Format: pytest

Output:
```python
import pytest

def test_happy_path_1():
    ...
```
```

---

## 11. Comparison & Analysis

### 11.1 Compare Items

```
So sánh {A} vs {B}:

Tiêu chí:
1. {criterion_1} (weight: 30%)
2. {criterion_2} (weight: 30%)
3. {criterion_3} (weight: 40%)

Output:
| Criterion | A | B | Winner |
|-----------|---|---|--------|
| ...       |...|...| ...   |

Overall winner: {A/B/TIE}
Reasoning: ...

Recommendation: dùng {A/B} nếu {condition}, dùng {other} nếu {condition}
```

### 11.2 Pros & Cons

```
Liệt kê pros/cons của {topic}:

Output:
## Pros
1. [Pro] - [Why]
2. ...

## Cons
1. [Con] - [Why]

## Verdict
- For: [target audience/scenario]
- Against: [target audience/scenario]
```

---

## 12. Validation & Verification

### 12.1 Fact Checking

```
Verify claims in text. Use ONLY general knowledge (no current internet):

Text: {text}

Output JSON:
{
  "claims": [
    {
      "claim": "...",
      "verdict": "TRUE | FALSE | UNCERTAIN | OUTDATED",
      "explanation": "..."
    }
  ]
}
```

### 12.2 Logical Consistency

```
Check logical consistency:

Statements:
1. {s1}
2. {s2}
3. {s3}

Identify:
- Contradictions
- Implicit assumptions
- Logical fallacies
- Missing premises
```

---

## 13. Conversation Generation

### 13.1 Roleplay Dialog

```
Generate dialog between:

Character A: {persona_a}
Character B: {persona_b}

Setting: {setting}
Goal: {dialog_goal}
Length: {n_turns} turns

Format:
A: ...
B: ...

End with: clear resolution or cliffhanger.
```

### 13.2 Mock Interview

```
Simulate job interview:

Role: {role}
Level: {seniority}
Type: {behavioral / technical / system_design}

Interview format:
1. You ask question
2. Wait for my answer
3. Give feedback after my answer
4. Ask follow-up
5. Repeat for {n} questions

Start with: brief intro then first question.
```

---

## 14. Specialized Tasks

### 14.1 SQL Query Generation

```
Generate SQL query (PostgreSQL):

Schema:
{schema}

Question (natural language):
{question}

Rules:
- Use proper joins
- Add LIMIT 100 nếu không specify
- Use parameterized values (placeholders $1, $2)
- Avoid SELECT *
- Explain query intent in comment

Output:
-- {intent}
SELECT ...
```

### 14.2 Regex Generation

```
Generate regex pattern:

Need to match: {description}

Examples to match:
- ...
- ...

Examples to NOT match:
- ...
- ...

Output:
- Pattern: /.../flags
- Explanation
- Test the pattern against examples mentally
```

### 14.3 Email Drafting

```
Draft email:

To: {recipient}
Tone: {formal / casual / friendly}
Purpose: {purpose}
Key points to cover:
- ...
- ...

Length: {short / medium / long}

Format:
Subject: ...

[Body]

[Sign-off]
```

---

## 15. Demo: Complete Email Analysis

```python
from pydantic import BaseModel, Field
from typing import Literal, List

class EmailAnalysis(BaseModel):
    """Comprehensive email analysis"""
    
    # Summary
    summary: str = Field(description="1-2 sentence summary")
    
    # Classification
    category: Literal["complaint", "inquiry", "feedback", "sales", "spam"]
    urgency: Literal["low", "medium", "high", "critical"]
    
    # Extraction
    sender_intent: str = Field(description="What sender wants")
    action_items: List[str] = Field(description="Actions needed")
    deadline: str | None = Field(description="If mentioned")
    
    # Sentiment
    sentiment: Literal["positive", "neutral", "negative", "angry"]
    
    # Response suggestion
    suggested_reply: str = Field(description="Draft response 100 words")
    estimated_response_time_minutes: int

llm = ChatOpenAI(model="gpt-4o-mini")
analyzer = llm.with_structured_output(EmailAnalysis)

email = """
Subject: URGENT - Đơn hàng #12345 chưa nhận!

Đã 10 ngày kể từ ngày đặt mà tôi chưa nhận được hàng.
Đã gọi support 3 lần, mỗi lần được hứa hôm sau giao.
Đây là quà sinh nhật vợ tôi VÀO NGÀY MAI.
Nếu ngày mai không nhận được, tôi sẽ:
- Yêu cầu refund toàn bộ
- Review 1 sao trên các platform
- Báo cáo Cục QLCT

An - khách VIP từ 2020
"""

result = analyzer.invoke(email)
print(result.model_dump_json(indent=2))
```

---

## 16. Checklist

- [ ] Summarization (basic, structured, multi-doc)
- [ ] Classification (single, multi, hierarchical)
- [ ] Information extraction
- [ ] Translation (basic + style preservation)
- [ ] Code generation, review, refactor
- [ ] RAG prompts
- [ ] Sentiment analysis (basic + aspect-based)
- [ ] Creative writing templates
- [ ] Synthetic data generation
- [ ] Comparison & analysis
- [ ] SQL/regex generation
- [ ] Conversational templates

➡️ **Tiếp theo**: [Anti-Hallucination](./09-Anti-Hallucination.md)
