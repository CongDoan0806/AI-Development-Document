# Basic Techniques - Zero-shot, Few-shot, Instruction-based

## 1. Zero-Shot Prompting

**Zero-shot** = LLM trả lời mà KHÔNG cần examples, chỉ dựa vào instruction.

### Ví Dụ
```
Translate this to Vietnamese: "Hello, how are you?"
```

→ Output: "Xin chào, bạn khỏe không?"

### Khi Nào Dùng
- ✅ Task phổ biến (translate, summarize)
- ✅ Model đã "biết" task (pre-trained on similar)
- ✅ Format output đơn giản

### Khi Nào KHÔNG Dùng
- ❌ Task domain-specific
- ❌ Output format phức tạp/non-standard
- ❌ Need consistency cao

### Tips Zero-Shot

#### Tip 1: Cụ Thể Format
```
❌ "Translate to Vietnamese"
✅ "Translate to Vietnamese. Output format:
   Original: [English]
   Translation: [Vietnamese]"
```

#### Tip 2: Define Edge Cases
```
✅ "Translate. If text already in Vietnamese, return as-is.
   If contains code, don't translate code parts."
```

#### Tip 3: Add "Think Step By Step"
```
Q: Tom có 5 quả táo, cho Mary 2 quả, mua thêm 3. Tom còn mấy?
A: Let's think step by step.
```

→ Chi tiết ở [04-Chain-of-Thought](./04-Chain-of-Thought.md)

---

## 2. Few-Shot Prompting

**Few-shot** = cho LLM xem **1-5 examples** trước khi yêu cầu task.

### Ví Dụ Basic

```
Convert to formal English:

Informal: "wassup bro?"
Formal: "Hello, how are you?"

Informal: "no way!"
Formal: "That is not possible."

Informal: "gonna be late"
Formal: "I will be late."

Informal: "thx for the help"
Formal:
```

→ Output: "Thank you for your help."

### One-Shot, Two-Shot, ...

| Variant | Examples |
|---------|----------|
| Zero-shot | 0 |
| One-shot | 1 |
| Few-shot | 2-5 |
| Many-shot | 10-100+ (mới phổ biến với long context) |

### Tại Sao Hiệu Quả

Examples giúp LLM:
- 🎯 Hiểu format output mong muốn
- 📐 Match style (formal/casual)
- 🔍 Học pattern subtle
- 📊 Hiệu chỉnh confidence/distribution

---

## 3. Chọn Examples

### 3.1 Đa Dạng

❌ **Bad**: 3 examples đều là positive sentiment
```
"Great!" → positive
"Awesome!" → positive
"Love it!" → positive
```

✅ **Good**: cover all categories
```
"Great!" → positive
"Terrible" → negative
"It's okay" → neutral
"Best ever!" → positive  (variety in positive)
"Hate this" → negative
```

### 3.2 Đại Diện (Representative)

Examples nên match **distribution thật** của data sản phẩm:

```
Production data: 60% positive, 30% neutral, 10% negative
→ Examples: 3 positive, 2 neutral, 1 negative (proportional)
```

### 3.3 Order Matters

**Cuối cùng → ảnh hưởng nhiều nhất** (recency bias).

Đặt example "khó nhất" hoặc edge case cuối cùng:

```
Easy: "Tôi yêu sản phẩm" → positive
Medium: "Sản phẩm ổn nhưng đắt" → neutral
Hard: "Sản phẩm bình thường, không đáng tiền" → negative
```

### 3.4 Dynamic Example Selection

Thay vì hardcode examples, **chọn dynamically** based on input:

```python
from langchain_community.example_selectors import SemanticSimilarityExampleSelector

selector = SemanticSimilarityExampleSelector.from_examples(
    all_examples,        # Pool 100+ examples
    embeddings,
    vectorstore,
    k=3,                 # Chọn 3 examples gần nhất
)

# Cho mỗi input, chọn 3 examples relevant nhất
similar = selector.select_examples({"input": user_query})
```

→ Tăng accuracy 5-15% so với static examples.

---

## 4. Instruction-Based Prompting

Cho instructions clear, không cần examples.

### Pattern 1: Detailed Instructions

```
Task: Generate product description.

Instructions:
1. Length: 100-150 words
2. Tone: friendly, professional
3. Highlight: 3 key features
4. Include: target audience
5. End with: 1 call-to-action sentence

Avoid:
- Superlatives without proof ("the best")
- Technical jargon
- Comparing to competitors

Product info: [...]
```

### Pattern 2: Numbered Steps

```
Generate test cases for the function. Follow these steps:

Step 1: Identify input types and ranges
Step 2: List edge cases (empty, null, boundary, invalid)
Step 3: Define expected output for each
Step 4: Write test code in pytest format

Function:
[...]
```

### Pattern 3: Constraints + Format

```
Summarize the article.

Constraints:
- Max 3 bullet points
- Each bullet < 15 words
- Capture: WHO did WHAT, WHEN, WHY
- Use simple language (grade 8 level)

Output format:
- [bullet 1]
- [bullet 2]
- [bullet 3]

Article:
[...]
```

---

## 5. So Sánh Zero-Shot vs Few-Shot

### Task: Sentiment Classification

**Zero-shot**:
```
Classify sentiment: "Sản phẩm bình thường, giá hơi cao"
```
→ Output: "neutral" hoặc "mixed" hoặc "neutral-negative" (không nhất quán)

**Few-shot**:
```
Classify sentiment as one of: positive, negative, neutral

"Tôi yêu sản phẩm" → positive
"Quá tệ" → negative
"Bình thường" → neutral

"Sản phẩm bình thường, giá hơi cao" →
```
→ Output: "neutral" (consistent format)

### Performance

| Task | Zero-shot | Few-shot (5) |
|------|-----------|--------------|
| Translate (common) | 90% | 92% |
| Sentiment | 70% | 88% |
| Domain extraction | 50% | 85% |
| Custom format | 40% | 90% |

→ **Few-shot win lớn cho custom/domain tasks**.

---

## 6. Few-Shot Best Practices

### 6.1 Consistent Format Across Examples

❌ **Bad**:
```
"Great!" → positive
"Terrible" - it's negative
sentiment("okay"): neutral
```

✅ **Good**:
```
"Great!" → positive
"Terrible" → negative
"Okay" → neutral
```

### 6.2 Use Delimiter

```
Examples:
###
Input: "..."
Output: "..."
###
Input: "..."
Output: "..."
###
```

### 6.3 Match Real Use Case

Examples nên giống **real input** nhất có thể:
- Cùng length
- Cùng style
- Cùng language
- Cùng noise level

### 6.4 Hard Negative Examples

Thêm example "tricky" để teach edge cases:

```
"Sản phẩm tốt nhưng tôi không thích màu" → positive
(Note: tổng thể positive vì khen sản phẩm, ý kiến cá nhân về màu không phải đánh giá sản phẩm)
```

---

## 7. Instruction Tuning Mode

Một số LLM được train với **specific format**. Follow format đó cho best result.

### OpenAI Format
```python
messages = [
    {"role": "system", "content": "You are..."},
    {"role": "user", "content": "..."},
]
```

### Claude Format
```python
# Recommend dùng XML tags
prompt = """<instructions>
Bạn là...
</instructions>

<task>
...
</task>

<input>
...
</input>"""
```

### Llama 3 Format
```
<|begin_of_text|><|start_header_id|>system<|end_header_id|>
You are a helpful assistant.
<|eot_id|><|start_header_id|>user<|end_header_id|>
What is 2+2?
<|eot_id|><|start_header_id|>assistant<|end_header_id|>
```

---

## 8. Length-Adaptive Prompting

Adjust prompt length based on task complexity:

### Simple Task (few-shot 1-2)
```
Q: 5 + 3 = ?
A: 8

Q: 7 * 2 = ?
A:
```

### Medium Task (few-shot 3-5 + brief instructions)
```
Convert to formal English.

"Hey there" → "Hello"
"Gonna" → "Going to"
"Wanna" → "Want to"

"Lemme know"
```

### Complex Task (full anatomy)
```
[ROLE + CONTEXT + DETAILED TASK + CONSTRAINTS + 5 EXAMPLES + INPUT + OUTPUT FORMAT]
```

---

## 9. Negation Prompting

LLM thường ưu tiên **làm gì đó** hơn là **không làm**. Cẩn thận với negation.

### Bad
```
"Don't write about animals"
```
→ LLM có thể vẫn đề cập animals.

### Better
```
"Write only about plants. Avoid mentioning animals."
```

### Best - Pair Positive
```
"Topic must be: plants, flowers, gardens.
NOT include: animals, pets, wildlife.
If you mention animal accidentally, replace with 'organism'."
```

---

## 10. Demo: Sentiment Classifier

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Zero-shot
zero_shot = "Phân loại sentiment: {text}"

# Few-shot
few_shot = """Phân loại sentiment thành: positive, negative, neutral.

Examples:
"Sản phẩm tuyệt vời!" → positive
"Quá tệ, đừng mua" → negative
"Bình thường" → neutral
"Giá tốt nhưng giao chậm" → neutral
"Xuất sắc!" → positive

Text: {text}
Sentiment:"""

# Instruction-based
instruction = """Bạn là expert phân tích sentiment.

Quy tắc:
1. Output CHỈ 1 trong: positive, negative, neutral
2. Không giải thích
3. Mixed feelings → neutral
4. Uncertain → neutral

Text: {text}
Sentiment:"""

# Test
test_cases = [
    "Yêu lắm!",
    "Bình thường",
    "Tệ, hỏng ngay",
    "Khen sản phẩm nhưng chê giao hàng",  # Mixed
]

for tc in test_cases:
    print(f"\nText: {tc}")
    for name, prompt in [("zero", zero_shot), ("few", few_shot), ("inst", instruction)]:
        out = llm.invoke(prompt.format(text=tc)).content
        print(f"  {name}: {out[:50]}")
```

---

## 11. Bài Tập

### Bài 1: Compare Approaches
Task: Convert date format from various formats to ISO 8601.
- Build zero-shot, few-shot (3 examples), instruction-based versions
- Test với 10 inputs đa dạng
- Đo accuracy

### Bài 2: Few-Shot Selection
Pool 50 sentiment examples. Implement dynamic selection (semantic similarity). So sánh:
- Random 3 examples
- Static 3 examples
- Dynamic 3 examples

### Bài 3: Edge Case Examples
Cho task "Extract phone number from text". List 10 edge cases và viết few-shot examples cover các edge cases đó.

---

## 12. Checklist

- [ ] Hiểu khi nào dùng zero-shot vs few-shot
- [ ] Viết được examples đa dạng
- [ ] Dynamic example selection
- [ ] Instruction-based prompts với constraints
- [ ] Avoid negation pitfalls
- [ ] Format-specific cho từng model

➡️ **Tiếp theo**: [Chain-of-Thought](./04-Chain-of-Thought.md)
