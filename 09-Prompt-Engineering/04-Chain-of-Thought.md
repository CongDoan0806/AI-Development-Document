# Chain-of-Thought (CoT) Prompting

## 1. CoT Là Gì?

**Chain-of-Thought** = yêu cầu LLM **suy luận từng bước** trước khi đưa đáp án, thay vì "1 shot".

### Trước CoT
```
Q: An có 5 quả táo. Cho Bình 2 quả. Mua thêm 6 quả. An có mấy?
A: 9
```

### Với CoT
```
Q: An có 5 quả táo. Cho Bình 2 quả. Mua thêm 6 quả. An có mấy?
A: Hãy suy nghĩ từng bước.
   Bước 1: An có 5 quả.
   Bước 2: Cho Bình 2 quả → còn 5 - 2 = 3 quả.
   Bước 3: Mua thêm 6 quả → 3 + 6 = 9 quả.
   Đáp án: 9 quả.
```

→ **Accuracy tăng đáng kể** cho task reasoning (math, logic, multi-step).

---

## 2. Tại Sao CoT Hiệu Quả?

### 2.1 LLM "Suy Nghĩ" Bằng Tokens
LLM generate text từng token. Mỗi token là 1 "bước nghĩ".

```
1 shot answer: 1 token output để decide
CoT answer: 100+ tokens để reason, sau đó conclude
```

→ Nhiều "thinking time" → ít error.

### 2.2 Break Down Complex Problems
Problem phức tạp → chia thành sub-problems → dễ giải hơn.

### 2.3 Tránh "Shortcut Errors"
LLM không bị áp lực trả lời ngay → ít bịa.

---

## 3. Zero-Shot CoT

Đơn giản thêm cụm từ trigger:

### Tiếng Anh
```
"Let's think step by step."
"Let's work this out step by step."
"Take a deep breath and think carefully."
```

### Tiếng Việt
```
"Hãy suy nghĩ từng bước."
"Hãy phân tích cẩn thận."
"Hãy lập luận từ đầu đến cuối."
```

### Ví Dụ
```
Q: Roger có 5 quả bóng tennis. Anh mua thêm 2 hộp, mỗi hộp 3 quả. 
   Roger có tổng bao nhiêu quả?

A: Let's think step by step.
   - Roger có 5 quả ban đầu.
   - Anh mua 2 hộp × 3 quả/hộp = 6 quả.
   - Tổng: 5 + 6 = 11 quả.
   Đáp án: 11.
```

---

## 4. Few-Shot CoT

Cho examples có reasoning detail:

```
Q: Sân có 23 quả bóng. 5 quả bị bay đi, 8 quả mới được mua thêm. Sân có bao nhiêu?
A: Bắt đầu với 23 quả. 5 quả bay đi → 23 - 5 = 18. 8 quả mới → 18 + 8 = 26. Đáp án: 26.

Q: Có 4 con chim. Mỗi con đẻ 3 trứng. Tổng có bao nhiêu trứng?
A: 4 con × 3 trứng/con = 12 trứng. Đáp án: 12.

Q: Lan có 10 quyển sách. Tặng bạn 3 quyển, mẹ mua thêm gấp đôi số còn lại. Lan có bao nhiêu?
A:
```

→ Output:
```
Lan ban đầu có 10 quyển. 
Tặng 3 quyển → còn 10 - 3 = 7 quyển. 
Mẹ mua gấp đôi 7 = 14 quyển mới. 
Tổng: 7 + 14 = 21 quyển. Đáp án: 21.
```

---

## 5. Structured CoT

Bắt LLM theo format reasoning cụ thể:

```
Task: Solve word problem.

Format:
1. UNDERSTAND: Identify what's given and what's asked
2. PLAN: List steps to solve
3. SOLVE: Execute each step with calculation
4. VERIFY: Sanity check the answer
5. ANSWER: Final number

Problem: {problem}
```

Output:
```
1. UNDERSTAND:
   - Given: An có 5 táo, cho 2, mua thêm 6
   - Asked: An có mấy?

2. PLAN:
   Step 1: Start = 5
   Step 2: Subtract 2 (given to Binh)
   Step 3: Add 6 (bought more)

3. SOLVE:
   5 - 2 = 3
   3 + 6 = 9

4. VERIFY:
   - Did we use all info? Yes.
   - Logical? Yes (gave away then bought).
   
5. ANSWER: 9 quả táo
```

---

## 6. Self-Consistency Với CoT

**Self-consistency** = gọi LLM nhiều lần, lấy majority vote.

```python
def self_consistency(question, n=5):
    cot_prompt = f"Q: {question}\nA: Let's think step by step."
    
    answers = []
    for _ in range(n):
        # Temperature > 0 để có variation
        output = llm.invoke(cot_prompt, temperature=0.7).content
        # Extract số cuối cùng
        final = extract_final_answer(output)
        answers.append(final)
    
    # Majority vote
    from collections import Counter
    most_common = Counter(answers).most_common(1)[0]
    return most_common[0]

# Test
answer = self_consistency("Tom có... [complex problem]", n=5)
# Nếu 4/5 lần ra 9, 1/5 ra 8 → answer = 9
```

**Hiệu quả**: tăng accuracy 5-15% cho math problems.

**Trade-off**: chi phí × N lần.

---

## 7. Least-to-Most Prompting

Chia problem khó thành **sub-problems từ dễ đến khó**:

```
Problem: "Trên đường đến trường, học sinh đi 1.2km. Đi xe buýt 4km, 
         sau đó đi bộ 800m về nhà. Tính tổng quãng đường?"

Decompose:
- Sub 1: Quãng đường đến trường là gì?
- Sub 2: Quãng đường từ trường về nhà là gì?
- Sub 3: Tổng quãng đường?

Solve:
- Sub 1: 1.2km = 1200m
- Sub 2: 4km + 800m = 4000m + 800m = 4800m
- Sub 3: 1200m + 4800m = 6000m = 6km

Answer: 6km
```

```python
decompose_prompt = """
Chia problem này thành sub-problems theo thứ tự từ dễ đến khó:

Problem: {problem}

Sub-problems:
1. ...
2. ...
3. ...
"""

solve_prompt = """
Giải sub-problem dựa trên kết quả các sub-problem trước:

Sub-problem: {sub}
Previous results: {prev}

Solution:
"""
```

---

## 8. Plan-and-Solve Prompting

Tách thành 2 phase rõ ràng:

```
Phase 1 - PLAN:
"Trước khi giải, hãy lập plan các bước cần làm. Không giải vội."

Phase 2 - SOLVE:
"Bây giờ thực hiện plan đã lập."
```

### Implementation

```python
plan_prompt = """
Problem: {problem}

Hãy lập plan giải problem này. KHÔNG giải vội, chỉ liệt kê các bước.

Plan:
1. ...
2. ...
3. ...
"""

solve_prompt = """
Problem: {problem}
Plan: {plan}

Bây giờ thực hiện từng bước theo plan:
"""

# Use
plan = llm.invoke(plan_prompt.format(problem=p)).content
solution = llm.invoke(solve_prompt.format(problem=p, plan=plan)).content
```

---

## 9. CoT-SC (Self-Consistency + CoT)

Combine CoT + self-consistency:

```python
def cot_sc(question, n=5):
    """CoT với self-consistency"""
    paths = []
    for _ in range(n):
        cot = llm.invoke(
            f"Q: {question}\nA: Let's think step by step.",
            temperature=0.7,
        ).content
        paths.append(cot)
    
    # Extract final answers
    answers = [extract_final(p) for p in paths]
    
    # Majority vote
    return most_common(answers), paths
```

---

## 10. CoT Không Phải Lúc Nào Cũng Tốt

### Khi KHÔNG Dùng CoT
- ❌ Task simple (1+1=?)
- ❌ Format extraction (chỉ cần output JSON)
- ❌ Stream low-latency cần
- ❌ Cost-sensitive (CoT tốn nhiều output tokens)

### Khi DÙNG CoT
- ✅ Math problems
- ✅ Logic puzzles
- ✅ Multi-step reasoning
- ✅ Complex decision making
- ✅ Code debugging

---

## 11. CoT Cho Different Task Types

### 11.1 Math
```
Hãy giải step-by-step:
- Đặt biến
- Viết phương trình
- Giải
- Verify

Problem: {problem}
```

### 11.2 Logic
```
Phân tích logic step-by-step:
- Identify premises
- Apply rules
- Derive conclusion
- Check consistency

Problem: {problem}
```

### 11.3 Classification
```
Phân loại {item} với reasoning:
- List relevant features
- Match với criteria của mỗi category
- Chọn category có nhiều match nhất
- Reasoning final

Item: {item}
```

### 11.4 Code Debugging
```
Debug code này step-by-step:
- Đọc code, identify input/output expected
- Trace execution với sample input
- Identify mismatch
- Propose fix
- Verify fix

Code: {code}
```

---

## 12. Implicit vs Explicit CoT

### Implicit CoT (cũ)
```
"Let's think step by step."
```
→ LLM tự decide cách reason.

### Explicit CoT (kiểm soát hơn)
```
"Solve step by step. Use this exact format:
Step 1: [Identify variables]
Step 2: [Set up equation]
Step 3: [Solve]
Step 4: [Verify]
Final answer: [Number]"
```

---

## 13. Reasoning Models (o1, Claude Extended Thinking)

OpenAI o1 và Claude với extended thinking → **CoT built-in**, không cần prompt riêng.

```python
# OpenAI o1
llm = ChatOpenAI(model="o1-preview")
response = llm.invoke("Complex problem")
# Model tự reason, không cần "step by step"

# Claude Opus 4.7 extended thinking
llm = ChatAnthropic(
    model="claude-opus-4-7",
    max_tokens=8000,
    thinking={"type": "enabled", "budget_tokens": 5000},
)
response = llm.invoke("Complex problem")
# Có reasoning trace riêng (không phí output tokens)
```

**Trade-off**:
- ✅ Better reasoning out of box
- ❌ Slower (5-30s)
- ❌ Đắt hơn (reasoning tokens)

---

## 14. Demo: CoT Improvement

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

problem = """
Một bể nước có thể chứa 1000 lít. Vòi A đổ vào 50 lít/phút.
Vòi B đổ vào 30 lít/phút. Vòi C xả ra 20 lít/phút.
Nếu cả 3 vòi cùng mở khi bể trống, sau bao lâu bể đầy?
"""

# Without CoT
direct = llm.invoke(f"Q: {problem}\nA:").content

# With Zero-Shot CoT
cot = llm.invoke(f"Q: {problem}\nA: Let's think step by step.").content

# With Structured CoT
structured_cot = llm.invoke(f"""
Solve this problem step-by-step:

Step 1: Identify rates of inflow and outflow
Step 2: Calculate net flow rate
Step 3: Calculate time to fill

Problem: {problem}
""").content

print("=== Direct ===\n", direct)
print("\n=== CoT ===\n", cot)
print("\n=== Structured CoT ===\n", structured_cot)
```

Output:
```
=== Direct ===
Khoảng 16 phút

=== CoT ===
Bước 1: Tốc độ vào: A (50) + B (30) = 80 lít/phút
Bước 2: Tốc độ ra: 20 lít/phút
Bước 3: Net = 80 - 20 = 60 lít/phút
Bước 4: Thời gian = 1000 / 60 ≈ 16.67 phút
Đáp án: 16.67 phút (~16 phút 40 giây)

=== Structured CoT ===
Step 1: 
  - Inflow: A = 50 L/min, B = 30 L/min, Total = 80 L/min
  - Outflow: C = 20 L/min

Step 2:
  - Net = Inflow - Outflow = 80 - 20 = 60 L/min

Step 3:
  - Time = Volume / Net rate = 1000 / 60 = 16.67 phút
  - = 16 phút 40 giây

Answer: 16 phút 40 giây
```

---

## 15. Bài Tập

### Bài 1: CoT Performance
Test 20 math problems với:
- No CoT
- Zero-shot CoT
- Few-shot CoT (3 examples)
- Self-Consistency CoT (n=5)

Đo accuracy của mỗi approach.

### Bài 2: Structured CoT
Build prompt structured CoT cho task "Recommend a restaurant" với criteria: cuisine, budget, location, occasion.

### Bài 3: Least-to-Most
Chọn 1 multi-step word problem. Implement decomposition + sequential solving.

---

## 16. Checklist

- [ ] Hiểu CoT và tại sao hiệu quả
- [ ] Zero-shot CoT với trigger phrases
- [ ] Few-shot CoT với detailed examples
- [ ] Structured CoT với format
- [ ] Self-consistency để boost accuracy
- [ ] Least-to-most decomposition
- [ ] Plan-and-Solve pattern
- [ ] Biết khi nào KHÔNG nên dùng CoT

➡️ **Tiếp theo**: [Advanced Reasoning](./05-Advanced-Reasoning.md)
