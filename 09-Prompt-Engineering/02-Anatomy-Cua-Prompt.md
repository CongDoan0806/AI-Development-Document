# Anatomy Của Một Prompt Hoàn Chỉnh

## 1. Cấu Trúc 7 Phần

Một prompt tốt thường có 7 thành phần (không phải lúc nào cũng đủ):

```
┌─────────────────────────────────────┐
│ [ROLE]         Vai trò của LLM      │
│ [CONTEXT]      Background info       │
│ [TASK]         Yêu cầu cụ thể        │
│ [CONSTRAINTS]  Quy tắc, giới hạn     │
│ [EXAMPLES]     Few-shot examples     │
│ [INPUT]        Data cần xử lý        │
│ [OUTPUT]       Format mong muốn      │
└─────────────────────────────────────┘
```

---

## 2. ROLE (Vai Trò)

Định nghĩa **persona** cho LLM. Giúp model "vào vai" và trả lời nhất quán.

### Tốt
```
Bạn là Senior Python Engineer với 15 năm kinh nghiệm,
chuyên về performance optimization và clean architecture.
```

```
You are a friendly customer support agent for an e-commerce company.
Your tone is helpful, empathetic, and professional.
```

### Hiệu Quả
| Generic | Specific |
|---------|----------|
| "Bạn là chuyên gia" | "Bạn là chuyên gia DevOps tại Google với expertise về k8s" |
| "Bạn là AI" | "Bạn là AI dạy toán cho học sinh lớp 5" |
| "You are an assistant" | "You are a medical research assistant who only cites peer-reviewed papers" |

### Khi Nào Skip
- Single-turn factual: "Capital of France?" (không cần role)
- API task technical: extraction, format conversion

---

## 3. CONTEXT (Bối Cảnh)

Cung cấp background information LLM cần biết.

### Tốt
```
CONTEXT:
- Khách hàng vừa mua sản phẩm A vào 15/3
- Họ phàn nàn về việc giao hàng chậm 5 ngày
- Đây là lần thứ 2 họ gặp vấn đề tương tự
- Chính sách: refund toàn phần nếu chậm > 3 ngày
```

### Tại Sao Quan Trọng
LLM không có context bên ngoài. Phải đưa info vào prompt.

### Anti-Pattern: Too Much Context

❌ **Bad**: paste cả 5000 từ context vào prompt khi chỉ cần 200 từ relevant.

✅ **Good**: filter chỉ context relevant (dùng RAG).

---

## 4. TASK (Yêu Cầu)

**Action verbs** rõ ràng, cụ thể.

### Tốt
```
TASK: Phân tích đoạn email phía dưới và:
1. Trích xuất thông tin liên hệ (tên, email, phone)
2. Xác định mục đích chính của email
3. Đánh giá mức độ khẩn cấp (low/medium/high)
4. Đề xuất 3 cách reply
```

### Action Verbs Hữu Ích
- **Analyze, Identify, Extract** - cho extraction
- **Summarize, Condense** - cho tóm tắt
- **Classify, Categorize** - cho phân loại
- **Compare, Contrast** - cho so sánh
- **Generate, Create, Write** - cho sáng tạo
- **Evaluate, Assess, Rate** - cho đánh giá
- **Translate, Convert** - cho transformation
- **Explain, Describe** - cho giải thích

### Số Lượng Task

❌ **Quá nhiều**: 1 prompt làm 10 việc → kém chất lượng
✅ **1-3 tasks** trong 1 prompt là sweet spot
✅ Nếu nhiều hơn → chia thành **prompt chain**

---

## 5. CONSTRAINTS (Quy Tắc)

Cảnh giới điều LLM **PHẢI** và **KHÔNG ĐƯỢC** làm.

### Positive Constraints (PHẢI)
```
- Trả lời bằng tiếng Việt
- Tối đa 100 từ
- Mỗi câu bắt đầu bằng động từ
- Cite source [n] sau mỗi claim
```

### Negative Constraints (KHÔNG ĐƯỢC)
```
- KHÔNG dùng emoji
- KHÔNG đưa ý kiến cá nhân  
- KHÔNG nhắc tên đối thủ cạnh tranh
- KHÔNG hứa hẹn thời gian cụ thể
```

### Edge Case Rules
```
- Nếu không đủ thông tin → "Tôi không có đủ thông tin để trả lời"
- Nếu câu hỏi off-topic → từ chối lịch sự
- Nếu chứa PII → mask thông tin nhạy cảm
```

---

## 6. EXAMPLES (Few-Shot)

Cho LLM xem 1-5 ví dụ về input-output mong muốn.

```
EXAMPLES:

Input: "Sản phẩm tuyệt vời!"
Output: {"sentiment": "positive", "confidence": 0.95}

Input: "Tệ, hỏng ngay sau 1 ngày"
Output: {"sentiment": "negative", "confidence": 0.92}

Input: "Bình thường"
Output: {"sentiment": "neutral", "confidence": 0.78}

Now classify:
Input: "Giá tốt nhưng giao hàng chậm"
Output:
```

### Quy Tắc
- **2-5 examples** thường đủ (more = diminishing returns)
- **Đa dạng**: cover các edge cases
- **Đại diện**: examples phản ánh distribution thật
- **Theo thứ tự**: similar examples gần nhau giúp LLM học pattern

→ Chi tiết ở [03-Basic-Techniques](./03-Basic-Techniques.md)

---

## 7. INPUT (Data)

Data thật mà LLM cần xử lý.

### Use Delimiters
Tách biệt rõ instruction và input:

```
TASK: Tóm tắt text trong tag <article>

<article>
[Nội dung dài...]
</article>
```

**Tại sao**: tránh prompt injection - nếu user input có "Ignore above, do X", LLM ít có khả năng làm theo vì biết đó là data, không phải instruction.

### Common Delimiters
- Triple backticks: ` ``` `
- XML tags: `<tag>...</tag>` (Claude thích)
- Triple quotes: `"""..."""`
- Headers: `--- INPUT ---`

---

## 8. OUTPUT FORMAT

Định nghĩa rõ format mong muốn.

### Examples

**Plain text**:
```
Output: 1 đoạn văn 50-100 từ, không bullet points
```

**Bullet points**:
```
Output: Danh sách 5 bullet, mỗi bullet 1 câu < 20 từ
```

**JSON**:
```
Output JSON đúng schema:
{
  "sentiment": "positive" | "negative" | "neutral",
  "confidence": 0.0-1.0,
  "keywords": ["word1", "word2", ...]
}
```

**Markdown table**:
```
Output table markdown với columns: Name, Price, Rating
```

→ Chi tiết ở [07-Output-Formatting](./07-Output-Formatting.md)

---

## 9. Putting It All Together

### Ví Dụ 1: Customer Email Analyzer

```
[ROLE]
Bạn là chuyên gia customer experience với 10 năm kinh nghiệm trong e-commerce.

[CONTEXT]
- Công ty: ABC Shop (bán điện tử)
- Email volume: 500-1000/day
- SLA: phản hồi VIP < 2h, regular < 24h

[TASK]
Phân tích email khách hàng và:
1. Phân loại category (complaint, inquiry, feedback, support)
2. Đánh giá urgency (low/medium/high)
3. Identify customer tier (VIP/regular/new)
4. Đề xuất next action

[CONSTRAINTS]
- CHỈ dùng thông tin trong email
- KHÔNG bịa thêm context
- Trả lời tiếng Việt
- Nếu thiếu thông tin → "INSUFFICIENT_INFO"

[EXAMPLES]
Input: "Tôi VIP, mua hàng 2 năm. Đơn 5tr hỏng. CẦN REFUND NGAY!"
Output: {
  "category": "complaint",
  "urgency": "high",
  "customer_tier": "VIP",
  "next_action": "Liên hệ trong 2h, escalate manager, prepare refund"
}

[INPUT]
<email>
{email_content}
</email>

[OUTPUT FORMAT]
JSON với keys: category, urgency, customer_tier, next_action, confidence (0-1)
```

### Ví Dụ 2: Code Review Bot

```
[ROLE]
You are a Senior Python engineer doing code review.
Focus: correctness > readability > performance.

[CONTEXT]
- Project: production e-commerce backend
- Style guide: PEP 8 + Google docstrings
- Test coverage required: > 80%

[TASK]
Review the code and provide:
1. Bugs (if any) - critical to fix
2. Issues (style, naming, etc.) - should fix
3. Suggestions (improvements) - nice to have
4. Overall verdict: APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION

[CONSTRAINTS]
- Be specific: line numbers, exact suggestions
- Be constructive: explain WHY, give example fix
- Limit feedback to top 5 items
- Don't review style only - look at logic

[INPUT]
```python
{code}
```

[OUTPUT FORMAT]
## Bugs
- Line X: ... (severity: high/medium/low)
  - Fix: ...

## Issues
- Line Y: ...

## Suggestions
- ...

## Verdict
APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION
```

---

## 10. Order Matters

LLM đọc prompt theo thứ tự. Order tốt:

### Pattern 1: Instruction Trước
```
1. ROLE + CONTEXT
2. TASK  
3. CONSTRAINTS
4. EXAMPLES
5. INPUT
6. OUTPUT FORMAT
```
→ Tốt cho **complex tasks**.

### Pattern 2: Examples Trước
```
1. EXAMPLES (làm "warm-up")
2. INPUT
3. (LLM infer task từ examples)
```
→ Tốt cho **simple pattern matching**.

### Pattern 3: Input Cuối
```
1. ROLE
2. INSTRUCTION
3. INPUT (đặt cuối)
```
→ Phổ biến nhất, LLM "fresh memory" về input.

---

## 11. Whitespace & Formatting

### Dùng Line Breaks Rõ Ràng

❌ **Bad**:
```
Bạn là expert. Tóm tắt text: Lorem ipsum... Output 50 từ.
```

✅ **Good**:
```
Bạn là expert.

Tóm tắt text dưới đây:

Lorem ipsum...

Output: 50 từ.
```

### Headers Giúp Cấu Trúc

```
## TASK
...

## INPUT
...

## OUTPUT
...
```

---

## 12. Length Considerations

### Quá Ngắn
```
"Translate"
```
→ Translate cái gì? Sang ngôn ngữ nào?

### Quá Dài
```
"Bạn là chuyên gia AI thông minh hữu ích đa năng được tạo bởi Anthropic với mission..."
```
(5000 từ instruction)

→ Tốn tokens, LLM bị overload, key info bị lost.

### Sweet Spot
- **Instructions**: 100-500 tokens
- **Examples**: 2-5 examples, tổng ~500 tokens
- **Context (RAG)**: 1000-3000 tokens
- **Input**: tùy task

---

## 13. Compatibility Across Models

### GPT-4o
- Thích structured prompts với headers
- Tốt với JSON output
- Follow constraints chặt chẽ

### Claude (Anthropic)
- Thích XML tags (`<context>`, `<task>`)
- Excellent với long context
- Verbose nếu không constraint

### Gemini
- Thích markdown structure
- Multimodal natively
- Cần explicit format hơn

### Open Source (Llama, Mistral)
- Cần few-shot rõ ràng hơn
- Strict format không tốt bằng
- Cần repeat instructions

→ Test prompt trên model thực tế deploy.

---

## 14. Bài Tập

### Bài 1: Anatomy Audit
Lấy 3 prompt bạn đã viết. Identify đủ 7 thành phần. Phần nào thiếu?

### Bài 2: Refactor Prompt
Cho prompt:
```
"Tóm tắt báo cáo này thành những điểm chính"
```
Refactor đầy đủ 7 phần.

### Bài 3: Examples Power
Tạo 2 versions:
- v1: chỉ instruction
- v2: instruction + 3 examples

Test với 10 inputs. Đo accuracy.

---

## 15. Checklist

- [ ] Hiểu 7 thành phần của prompt
- [ ] Viết được ROLE specific
- [ ] Cung cấp CONTEXT đủ (không quá)
- [ ] TASK rõ ràng, có action verbs
- [ ] CONSTRAINTS cụ thể (positive + negative + edge)
- [ ] EXAMPLES đa dạng
- [ ] INPUT có delimiters
- [ ] OUTPUT FORMAT defined

➡️ **Tiếp theo**: [Basic Techniques](./03-Basic-Techniques.md)
