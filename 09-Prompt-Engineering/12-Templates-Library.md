# Templates Library

Thư viện prompt templates sẵn dùng cho các use case phổ biến.

## 1. Sentiment Analysis

### Template 1.1: Basic
```
Phân loại sentiment thành: positive, negative, neutral.

Quy tắc:
- Mixed feelings → neutral
- Sarcasm → đánh giá ý thật
- Empty/ambiguous → neutral

Examples:
"Yêu sản phẩm!" → positive
"Tệ" → negative
"Tốt nhưng đắt" → neutral

Text: {text}
Output (1 từ):
```

### Template 1.2: With Confidence + Aspects
```
Phân tích sentiment chi tiết.

Text: {text}

Output JSON:
{
  "overall_sentiment": "positive | negative | neutral",
  "confidence": 0.0-1.0,
  "aspects": {
    "product_quality": "positive | negative | neutral | not_mentioned",
    "price": "positive | negative | neutral | not_mentioned",
    "delivery": "positive | negative | neutral | not_mentioned",
    "service": "positive | negative | neutral | not_mentioned"
  },
  "key_phrases": ["..."]
}
```

---

## 2. Summarization

### Template 2.1: Concise Summary
```
Tóm tắt văn bản trong {N} câu, giữ ý chính.

Văn bản:
{text}

Tóm tắt:
```

### Template 2.2: Structured Summary
```
Tóm tắt theo cấu trúc:

## Topic
[1 sentence]

## Key Points (3-5 bullets)
- ...

## Conclusion
[1 sentence]

## Action Items (nếu có)
- ...

Văn bản:
{text}
```

### Template 2.3: Executive Summary
```
Tạo executive summary cho tài liệu này.

Format:
- 1 paragraph (3-5 sentences)
- Lead với bottom line / TLDR
- Include: key finding, business impact, recommended action
- Avoid jargon
- Quantify when possible

Document: {document}

Executive Summary:
```

---

## 3. Information Extraction

### Template 3.1: Contact Info
```
Trích xuất thông tin liên hệ từ text.

Output JSON:
{
  "name": "string | null",
  "email": "string | null",
  "phone": "string | null",
  "address": "string | null",
  "company": "string | null",
  "title": "string | null"
}

Text: {text}
```

### Template 3.2: Resume Parser
```
Parse resume thành cấu trúc JSON:

{
  "personal": {
    "name": "...",
    "email": "...",
    "phone": "..."
  },
  "summary": "1-2 sentence summary",
  "experience": [
    {
      "company": "...",
      "title": "...",
      "duration": "...",
      "responsibilities": ["..."]
    }
  ],
  "education": [
    {"school": "...", "degree": "...", "year": "..."}
  ],
  "skills": ["..."],
  "years_of_experience": 0
}

Resume: {text}
```

### Template 3.3: Invoice OCR Parser
```
Parse hóa đơn từ OCR text.

Output JSON:
{
  "vendor": "string",
  "invoice_number": "string",
  "date": "YYYY-MM-DD",
  "due_date": "YYYY-MM-DD | null",
  "items": [
    {
      "description": "string",
      "quantity": number,
      "unit_price": number,
      "total": number
    }
  ],
  "subtotal": number,
  "tax": number,
  "total": number,
  "currency": "VND | USD | ..."
}

OCR text:
{ocr}
```

---

## 4. Translation

### Template 4.1: Standard
```
Dịch sang tiếng {language}. Giữ tone của bản gốc.

Original: {text}
Translation:
```

### Template 4.2: Domain-Specific
```
Dịch document {domain} sang tiếng {language}.

Quy tắc:
- Technical terms: giữ tiếng Anh, thêm Vietnamese trong ngoặc
- Acronyms: explain lần đầu xuất hiện
- Numbers, dates, units: giữ format gốc
- Style: formal/academic

Text: {text}

Translation:
```

---

## 5. Classification

### Template 5.1: Email Classification
```
Phân loại email vào 1 category:
- complaint (khiếu nại)
- inquiry (hỏi thông tin)
- feedback (góp ý)
- support (hỗ trợ kỹ thuật)
- sales (mua bán)
- spam

Email: {email}

Output (1 category, không giải thích):
```

### Template 5.2: Multi-Label
```
Tag bài viết với nhiều topics phù hợp.

Topics available: tech, business, education, health, sport, entertainment, politics, lifestyle

Article: {text}

Output JSON: ["topic1", "topic2", ...]
```

### Template 5.3: Urgency Detection
```
Đánh giá mức độ urgent của message:

Levels:
- critical: cần phản hồi trong 1h, có thể impact business
- high: phản hồi trong 4h
- medium: trong 1 ngày
- low: > 1 ngày OK

Indicators:
- Words: "urgent", "ASAP", "immediately", "critical"
- Tone: angry, escalating
- Topic: payment, security, outage

Message: {text}

Output: {"urgency": "...", "reasoning": "..."}
```

---

## 6. Code Generation

### Template 6.1: Function Writer
```
Viết Python function:

Specification:
- Name: {name}
- Description: {description}
- Inputs: {inputs}
- Output: {output}
- Edge cases: {edge_cases}

Requirements:
- Type hints
- Google-style docstring với Example section
- Handle edge cases
- No external dependencies

Output ONLY code, no commentary:
```

### Template 6.2: Test Generator
```
Generate pytest tests for this function:

{function_code}

Coverage:
- 3 happy path tests
- 5 edge case tests (empty, None, boundary, invalid type, very large)
- 2 error handling tests

Use parameterize where appropriate.

Output:
```python
import pytest
from {module} import {function_name}

# tests here
```
```

### Template 6.3: Code Reviewer
```
Review code. Output structured feedback.

Code:
```{language}
{code}
```

Format:
## Bugs (critical to fix)
- Line X: [issue]
  - Why: [explanation]
  - Fix: [suggestion]

## Issues (should fix)
- ...

## Suggestions (improvements)
- ...

## Verdict: APPROVE | REQUEST_CHANGES | NEEDS_DISCUSSION
```

### Template 6.4: Bug Fix
```
Debug this code.

Code:
```
{code}
```

Symptom: {bug_description}

Output:
## Root Cause
[explanation]

## Fix
```python
{fixed_code}
```

## Test to Prevent Regression
```python
{test_case}
```
```

---

## 7. RAG / Q&A

### Template 7.1: Standard RAG
```
Bạn là trợ lý AI. Trả lời câu hỏi DỰA TRÊN context.

Quy tắc:
- CHỈ dùng info trong context
- Cite [Source N] sau mỗi claim
- Nếu không có info → "Tôi không có thông tin về điều này"
- Trả lời ngắn gọn, có cấu trúc

Context:
[Source 1] {doc1}
[Source 2] {doc2}
[Source 3] {doc3}

Question: {question}

Answer:
```

### Template 7.2: Conversational RAG
```
Bạn là AI assistant. Trả lời câu hỏi follow-up dựa trên history + context.

Chat History:
{history}

Context:
{context}

Question: {question}

Answer (consider history for pronouns/references):
```

### Template 7.3: Multi-Step QA
```
Trả lời câu hỏi phức tạp bằng cách chia thành sub-questions.

Question: {complex_question}

Step 1: Sub-questions cần trả lời:
1. ...
2. ...

Step 2: Trả lời từng sub-question (dùng context):
1. Answer 1: ...
2. Answer 2: ...

Step 3: Synthesize final answer:
[final answer combining sub-answers]
```

---

## 8. Customer Service

### Template 8.1: Reply Generator
```
Bạn là customer service rep. Generate reply email.

Customer email: {email}
Customer context: {history}
Issue category: {category}

Reply phải:
1. Greet warmly với tên (nếu biết)
2. Acknowledge cảm xúc/issue
3. Provide solution hoặc next steps
4. Set expectation (timing)
5. Friendly sign-off

Tone: empathetic, professional, solution-focused
Length: 100-200 từ
```

### Template 8.2: Escalation Decision
```
Quyết định case có cần escalate không.

Case: {case_details}

Criteria to escalate:
- Customer is VIP
- Loss > 100K VND
- Legal/compliance involved
- Customer threatens to leave/sue
- Issue affects multiple customers
- Repeated complaint same issue

Output JSON:
{
  "escalate": bool,
  "reason": "...",
  "suggested_handler": "manager | legal | tech_lead | retention_team",
  "priority": "low | medium | high | critical"
}
```

---

## 9. Marketing / Content

### Template 9.1: Product Description
```
Generate product description.

Product: {product_name}
Features: {features}
Target audience: {audience}
Tone: {tone}

Format:
- Headline (catchy, < 60 chars)
- Subheadline (value prop, < 100 chars)
- Body (3 short paragraphs)
- Bullet features (5 items)
- CTA (1 sentence)

Constraints:
- Avoid superlatives without proof
- Customer-focused (you/your, not we/our)
- SEO keywords: {keywords}
```

### Template 9.2: Social Media Post
```
Create {platform} post.

Topic: {topic}
Goal: {goal} (awareness/engagement/conversion)
Audience: {audience}

Constraints:
- Length: {limit} (e.g., 280 for Twitter)
- Tone: {tone}
- Include: 2-3 hashtags
- CTA: clear action

Output: text + hashtags
```

### Template 9.3: Email Subject Lines
```
Generate 10 email subject line variations.

Email purpose: {purpose}
Target audience: {audience}

Constraints:
- Length: 40-60 chars
- Avoid: "Free", "Buy now", "!!!" (spam triggers)
- Personalization: use {name} variable
- A/B test ready

Variations:
1. [Curiosity-driven]: ...
2. [Benefit-focused]: ...
3. [Question]: ...
4. [Number/list]: ...
5. [Urgency]: ...
6. [Personalized]: ...
...
```

---

## 10. Data Analysis

### Template 10.1: SQL Query Generation
```
Generate SQL query (PostgreSQL).

Schema:
{schema}

Question: {natural_language_question}

Rules:
- Use joins properly
- Add LIMIT 100 nếu không specify
- Comment explaining logic
- Handle NULL appropriately

Output:
-- {brief explanation}
SELECT ...
```

### Template 10.2: Data Insights
```
Analyze data và extract insights.

Data:
{data}

Output:
## Summary
[1 paragraph overview]

## Key Insights (3-5)
1. [Insight + supporting numbers]

## Anomalies / Outliers
[Anything unusual]

## Recommendations
- [Action items]

## Limitations
[What data doesn't tell us]
```

---

## 11. Education / Tutoring

### Template 11.1: Concept Explainer
```
Giải thích {concept} cho {audience_level}.

Format:
1. **Simple definition** (1 sentence, no jargon)
2. **Why it matters** (1 sentence, relate to daily life)
3. **Analogy** (compare to familiar thing)
4. **Detailed explanation** (3-4 sentences)
5. **Example** (concrete case)
6. **Common misconception** (clarify confusion)
7. **Next steps** (what to learn next)
```

### Template 11.2: Practice Problem Generator
```
Generate practice problems for {topic}, difficulty {level}.

Number: 5 problems

Per problem:
- Question
- Hint (no spoiler)
- Solution với explanation step-by-step

Format:
## Problem 1
[Question]

<details>
<summary>Hint</summary>
[hint]
</details>

<details>
<summary>Solution</summary>
[step-by-step]
</details>
```

### Template 11.3: Tutor Persona
```
Bạn là tutor patient, dạy {subject} cho học sinh {level}.

Approach:
1. KHÔNG cho đáp án ngay
2. Hỏi để hiểu họ stuck ở đâu
3. Give hints, không spoiler
4. Yêu cầu họ thử trước
5. Verify hiểu, không chỉ memorize
6. Connect to known concepts

Student question: {question}

Response (as tutor):
```

---

## 12. Brainstorming / Creativity

### Template 12.1: Idea Generator
```
Generate {N} ideas cho {topic}.

Constraints:
- Mỗi idea unique (no overlap)
- Mix conventional + creative
- Include 2-3 "out of the box" ideas
- Brief description (1-2 sentences each)

Categories to cover:
- [Cat 1]
- [Cat 2]
- [Cat 3]
```

### Template 12.2: SCAMPER (Innovation)
```
Apply SCAMPER framework to {topic/product}:

S - Substitute: gì có thể thay thế?
C - Combine: kết hợp gì với gì?
A - Adapt: từ ngành/contexts khác có gì?
M - Modify/Magnify: thay đổi/phóng đại gì?
P - Put to another use: dùng vào việc khác?
E - Eliminate: bỏ phần nào?
R - Reverse/Rearrange: đảo ngược/sắp xếp lại?

For each, generate 2 specific ideas.
```

---

## 13. Productivity

### Template 13.1: Meeting Summarizer
```
Tóm tắt meeting transcript.

Transcript: {transcript}

Output:
## Attendees
[list]

## Key Decisions
- [decision] (made by [whom])

## Action Items
- [ ] [task] - @{owner} - by [date]

## Open Questions
- [question]

## Follow-up Required
- [topic]
```

### Template 13.2: Email Triage
```
Triage email cho user.

Email: {email}

Output JSON:
{
  "priority": "high | medium | low | spam",
  "category": "...",
  "response_needed": bool,
  "estimated_response_time_min": int,
  "key_action": "string (1 sentence what user needs to do)",
  "suggested_response": "string (if can auto-draft)"
}
```

---

## 14. Research / Analysis

### Template 14.1: Compare & Contrast
```
So sánh {A} vs {B} về {dimensions}.

For each dimension:
- A: [position] - [reasoning]
- B: [position] - [reasoning]
- Winner: A | B | TIE
- Significance: high | medium | low

Final verdict:
- Overall winner: ...
- A is better when: [scenarios]
- B is better when: [scenarios]
- Avoid both when: [scenarios]
```

### Template 14.2: SWOT Analysis
```
Analyze {topic} using SWOT framework.

## Strengths (internal positive)
- ...

## Weaknesses (internal negative)
- ...

## Opportunities (external positive)
- ...

## Threats (external negative)
- ...

## Strategic Recommendation
[1 paragraph based on SWOT]
```

---

## 15. Persona Templates

### Template 15.1: Senior Engineer
```
Bạn là Senior Software Engineer với 15+ năm kinh nghiệm.

Background:
- Worked at FAANG companies
- Expert in [specific domain]
- Strong opinions backed by experience

Style:
- Direct, no sugarcoating
- Practical > theoretical
- Always consider trade-offs
- Reference patterns, real-world examples

Avoid:
- Vague advice
- Jargon without explanation
- "It depends" without context
```

### Template 15.2: Friendly Assistant
```
Bạn là trợ lý AI thân thiện như 1 người bạn.

Personality:
- Warm, supportive
- Use casual language
- Encouraging
- Empathetic with frustrations

Style:
- Conversational tone
- Short sentences
- Use emojis SẢN sàng moderately (😊 🎉) - không quá nhiều
- Vietnamese natural như nói chuyện

Avoid:
- Overly formal
- Lecture-y
- Cold/robotic
```

---

## 16. Specialized

### Template 16.1: JSON Validator + Fixer
```
Validate JSON. Nếu invalid, fix.

JSON: {json_string}

Output:
{
  "valid": bool,
  "errors": ["..."],
  "fixed_json": "..." or null
}
```

### Template 16.2: Regex Generator
```
Generate regex pattern.

Need to match: {description}

Must match these:
{positive_examples}

Must NOT match:
{negative_examples}

Output:
{
  "pattern": "regex string",
  "flags": "i|g|m|s as needed",
  "explanation": "what each part does",
  "test_against_examples": "verify each example"
}
```

---

## 17. Use Cases Specific to Vietnam

### Template 17.1: Vietnamese Address Parser
```
Parse Vietnamese address.

Address: {address}

Output JSON:
{
  "so_nha": "string | null",
  "ten_duong": "string | null",
  "phuong_xa": "string | null",
  "quan_huyen": "string | null",
  "tinh_thanh_pho": "string | null",
  "full_address_normalized": "string"
}

Note: Cấu trúc địa chỉ VN thường: [số nhà] [đường], [phường/xã], [quận/huyện], [tỉnh/TP]
```

### Template 17.2: Vietnamese Name Extractor
```
Extract Vietnamese names from text.

Text: {text}

Quy tắc:
- Tên Việt thường 2-4 từ (Họ Đệm Tên)
- Họ: Nguyễn, Trần, Lê, Phạm, Hoàng, ...
- Capitalize properly
- Có thể có dấu

Output JSON:
{
  "names": [
    {
      "full_name": "Nguyễn Văn An",
      "ho": "Nguyễn",
      "dem": "Văn",
      "ten": "An"
    }
  ]
}
```

### Template 17.3: Phone Number VN Parser
```
Parse Vietnamese phone numbers.

Text: {text}

Quy tắc:
- VN phone: +84xxx hoặc 0xxx (10-11 số)
- Mobile prefix: 03, 05, 07, 08, 09
- Landline: 02xxxxx

Output JSON:
{
  "phones": [
    {
      "original": "0901234567",
      "normalized": "+84901234567",
      "type": "mobile | landline | invalid"
    }
  ]
}
```

---

## 18. How To Use Templates

### Step 1: Pick Template

Find closest match to your use case.

### Step 2: Customize

- Replace `{placeholders}` với specific values
- Add domain-specific examples
- Adjust constraints

### Step 3: Test

- Run với 10+ test cases
- Measure accuracy, format compliance

### Step 4: Iterate

- Identify failures
- Refine prompt
- Re-test

### Step 5: Save

```python
# Save winning version
PROMPTS = {
    "sentiment_v3": """...""",
    "email_classify_v2": """...""",
}

# Or use LangSmith Hub
from langchain import hub
hub.push("my-org/sentiment-classifier", prompt_template)
```

---

## 19. Resources

### Public Prompt Libraries
- **LangSmith Hub**: https://smith.langchain.com/hub
- **PromptHero**: https://prompthero.com
- **Awesome ChatGPT Prompts**: https://github.com/f/awesome-chatgpt-prompts
- **Anthropic Prompt Library**: https://docs.anthropic.com/en/prompt-library

### Tools
- **PromptPerfect**: optimize prompts
- **Helicone**: prompt analytics
- **PromptLayer**: version control

---

## 20. Final Tips

✅ **Đừng reinvent the wheel**: search prompt hub trước khi viết từ đầu

✅ **Adapt templates**: không copy-paste, customize cho context

✅ **Version control**: track changes

✅ **Test on real data**: production input có thể different

✅ **Share with team**: build internal library

---

## 🎉 Hoàn Thành Phase 9 - Prompt Engineering!

Bạn đã học:
- ✅ Anatomy của 1 prompt tốt
- ✅ Zero-shot, few-shot, CoT techniques
- ✅ Advanced reasoning (ToT, ReAct, Reflexion)
- ✅ Role personas & multi-persona
- ✅ Output formatting (JSON, XML, structured)
- ✅ Task-specific templates
- ✅ Anti-hallucination strategies
- ✅ Prompt security (injection, jailbreak)
- ✅ A/B testing & optimization
- ✅ Templates library

### Next Steps
- Apply trong Real Projects ([08-Real-Projects](../08-Real-Projects/))
- Combine với RAG ([02-RAG](../02-RAG/))
- Build agents ([03-Agents](../03-Agents/))
- Evaluate prompts ([06-Evaluation/02-Prompt-Evaluation](../06-Evaluation/02-Prompt-Evaluation.md))
