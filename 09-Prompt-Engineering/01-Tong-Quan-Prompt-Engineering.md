# Tổng Quan Prompt Engineering

## 1. Prompt Engineering Là Gì?

**Prompt Engineering** = nghệ thuật và khoa học **viết câu lệnh (prompt)** để hướng LLM tạo ra output mong muốn.

```
Bad prompt:                    Good prompt:
"Viết code"             vs     "Viết Python function calculate_tax(income: float) -> float
                                tính thuế TNCN VN theo bậc thang, có type hints, docstring"
↓                              ↓
Code chung chung               Code đúng yêu cầu, có thể dùng ngay
```

**Same LLM, same model → output khác nhau hoàn toàn** chỉ vì prompt khác.

---

## 2. Tại Sao Quan Trọng?

### 2.1 Performance Boost

Nghiên cứu cho thấy:
- Zero-shot prompt cơ bản → 60% accuracy
- + Chain-of-Thought → 75%
- + Few-shot examples → 85%
- + Role + Structure + Constraints → 95%

→ **Prompt engineering có thể tăng 30-40% performance** mà không cần đổi model.

### 2.2 Rẻ Hơn Fine-tuning

| Approach | Cost | Time | Result |
|----------|------|------|--------|
| Fine-tuning | $1000+ | Vài ngày | +5-15% |
| Prompt engineering | $0 | Vài giờ | +20-40% |

→ Luôn **prompt engineering trước**, fine-tune sau (nếu cần).

### 2.3 Universal Skill

- Áp dụng cho mọi LLM (GPT, Claude, Gemini, Llama)
- Áp dụng cho mọi task
- Không cần code phức tạp
- Skill transferable

---

## 3. Triết Lý Cốt Lõi

### 3.1 LLM Là "Người Đọc Hướng Dẫn"

LLM **không đọc được ý nghĩ** của bạn. Nó chỉ làm theo những gì được viết.

❌ **Sai**: "Hỏi đó nhỉ, làm gì đó hay hay đi"

✅ **Đúng**: "Tóm tắt đoạn text dưới đây thành 3 bullet points, mỗi point dưới 20 từ"

### 3.2 Cụ Thể > Mơ Hồ

| Mơ hồ | Cụ thể |
|-------|--------|
| "Viết ngắn" | "Tối đa 100 từ" |
| "Trang trọng" | "Văn phong business email, không dùng slang" |
| "Code tốt" | "Code Python có type hints, docstring, handle edge cases" |
| "Phân tích" | "List 3 ưu, 3 nhược, 1 recommendation cuối cùng" |

### 3.3 Examples > Description

```
Description: "Output ở format YAML"
   ↓
Có thể LLM hiểu sai format

Example: "Output theo format này:
name: An
age: 25
tags:
  - dev
  - python"
   ↓
LLM follow theo example chính xác
```

### 3.4 Steps > Direct Answer

Yêu cầu LLM **suy luận từng bước** thay vì trả lời ngay:

```
❌ "Tính: 15% của 240 là bao nhiêu?"
   → LLM có thể bịa ra số sai

✅ "Tính 15% của 240. Hãy suy nghĩ từng bước, viết phép tính ra trước rồi mới ra đáp án."
   → LLM viết "15% = 0.15. 0.15 * 240 = 36"
```

---

## 4. Anatomy Của Một Prompt Hoàn Chỉnh

```
┌────────────────────────────────────────┐
│ 1. ROLE (Bạn là gì)                    │
├────────────────────────────────────────┤
│ 2. CONTEXT (Background info)           │
├────────────────────────────────────────┤
│ 3. TASK (Yêu cầu cụ thể)               │
├────────────────────────────────────────┤
│ 4. CONSTRAINTS (Quy tắc, giới hạn)     │
├────────────────────────────────────────┤
│ 5. EXAMPLES (Optional - few-shot)      │
├────────────────────────────────────────┤
│ 6. INPUT (Data cần xử lý)              │
├────────────────────────────────────────┤
│ 7. OUTPUT FORMAT (Format mong muốn)    │
└────────────────────────────────────────┘
```

→ Chi tiết ở [02-Anatomy](./02-Anatomy-Cua-Prompt.md)

---

## 5. Các Cấp Độ Prompt Engineering

### Level 1: Beginner
- Viết prompt cụ thể, có context
- Dùng few-shot examples
- Cho LLM "thời gian suy nghĩ" (CoT)

### Level 2: Intermediate
- Role playing
- Structured output (JSON, XML)
- Multi-step prompting
- Self-critique

### Level 3: Advanced
- Tree of Thoughts
- ReAct (Reasoning + Acting)
- Self-Consistency
- Constitutional AI
- Prompt chaining

### Level 4: Expert
- Adversarial prompts (red-teaming)
- Prompt injection defense
- Token-level optimization
- Meta-prompting

---

## 6. Các Loại Prompt Phổ Biến

### 6.1 Instructional
```
"Translate this to Vietnamese: Hello world"
```

### 6.2 Conversational
```
System: You are a helpful assistant.
User: How are you?
```

### 6.3 Completion (legacy)
```
"The capital of France is"
   → "Paris"
```

### 6.4 Q&A
```
Context: ...
Question: ...
Answer:
```

### 6.5 Code Generation
```
"Write Python function to..."
```

### 6.6 Structured Output
```
"Extract entities from text, output JSON: ..."
```

---

## 7. Common Mistakes

### ❌ Mistake 1: Mơ Hồ

```
"Make it better"
"Improve this"
"Help me with this code"
```

✅ Fix:
```
"Improve readability of this code by:
1. Add type hints
2. Add docstring with examples
3. Extract magic numbers to constants"
```

### ❌ Mistake 2: Quá Nhiều Yêu Cầu Cùng Lúc

```
"Translate to Vietnamese, summarize in 50 words,
extract keywords, classify sentiment, and format as JSON"
```

→ LLM bị overload, chất lượng giảm.

✅ Fix: chia thành **prompt chain** (sẽ học sau).

### ❌ Mistake 3: Không Cho Examples

```
"Generate test cases for this function"
   → Format/style ngẫu nhiên
```

✅ Fix:
```
"Generate test cases for this function. Examples:
- Input: [1,2,3], Output: 6
- Input: [], Output: 0
Now generate 5 more test cases."
```

### ❌ Mistake 4: Không Định Nghĩa "Tốt"

```
"Pick the best option"
   → LLM tự decide "tốt" là gì
```

✅ Fix:
```
"Pick the best option based on:
1. Performance (40%)
2. Cost (30%)
3. Maintainability (30%)
Score each option, then pick highest."
```

### ❌ Mistake 5: Quên Edge Cases

```
"Validate email"
   → LLM xử lý format hợp lệ thường
```

✅ Fix:
```
"Validate email. Handle these cases:
- Empty string
- Multiple @ signs
- Unicode characters
- IP-based domains
Return: {valid: bool, reason: str}"
```

---

## 8. Tools Để Test Prompt

### 8.1 OpenAI Playground
https://platform.openai.com/playground - test prompt với GPT models

### 8.2 Anthropic Workbench
https://console.anthropic.com/workbench - test với Claude

### 8.3 LangSmith Hub
https://smith.langchain.com/hub - share/discover prompts

### 8.4 PromptPerfect
https://promptperfect.jina.ai - auto-optimize prompt

### 8.5 Khác
- **Promptable**: A/B test
- **PromptLayer**: track production prompts
- **Helicone**: prompt observability

---

## 9. Lộ Trình Học

```
Week 1: 
  - 02-Anatomy (structure cơ bản)
  - 03-Basic Techniques (zero/few-shot)
  
Week 2:
  - 04-CoT (chain of thought)
  - 05-Advanced Reasoning
  
Week 3:
  - 06-Role/Personas
  - 07-Output Formatting
  
Week 4:
  - 08-Task Templates
  - 09-Anti-Hallucination
  
Week 5:
  - 10-Security
  - 11-Optimization
  - 12-Templates Library
```

---

## 10. Mindset Cần Có

### 10.1 Empathy
Đặt mình vào vị trí LLM: "Nếu mình là người mới đọc prompt này, có hiểu phải làm gì không?"

### 10.2 Iteration
Prompt đầu tiên rarely perfect. Cần **iterate** dựa trên output:
```
v1 → test → quan sát lỗi → fix prompt → v2 → ...
```

### 10.3 Test Thoroughly
- Happy path
- Edge cases (empty, very long, malformed)
- Adversarial (prompt injection)

### 10.4 Data-Driven
Đo lường bằng metrics, không "feel":
- Accuracy
- Format compliance
- Hallucination rate
- Latency, cost

---

## 11. Bài Tập

### Bài 1: Improve A Prompt
Cho prompt sau, viết version cải thiện:
```
"Summarize this"
```
(Input: bài báo dài)

### Bài 2: Define Quality Criteria
Cho task "Generate product description", define 5 tiêu chí "good" prompt sẽ optimize.

### Bài 3: Find Failure Modes
Test 1 prompt với 10 input đa dạng, list các trường hợp nó fail.

---

## 12. Checklist

- [ ] Hiểu prompt engineering quan trọng thế nào
- [ ] Phân biệt được prompt tốt vs xấu
- [ ] Biết các levels và roadmap học
- [ ] Setup tool để test prompt (Playground)
- [ ] Tránh được common mistakes
- [ ] Có mindset đúng (iterate, test, measure)

➡️ **Tiếp theo**: [Anatomy Của Prompt](./02-Anatomy-Cua-Prompt.md)
