# Multi-Step Reasoning - Lý Luận Nhiều Bước

## 1. Multi-Step Reasoning Là Gì?

**Multi-step reasoning** = giải task cần nhiều bước lý luận trung gian, không thể "1 shot".

**Ví dụ**:
```
Q: "Nếu Alice có 3 quả táo, cô ấy cho Bob 1 quả, rồi mua thêm gấp đôi số còn lại, hỏi Alice có bao nhiêu quả?"

Step 1: Alice có 3 quả
Step 2: Cho Bob 1 → còn 2 quả
Step 3: Mua gấp đôi 2 = 4 quả mới
Step 4: Tổng: 2 + 4 = 6 quả
```

LLM 1-shot thường sai, cần **explicit reasoning steps**.

---

## 2. Chain-of-Thought (CoT)

### 2.1 Zero-Shot CoT

Thêm "Let's think step by step":

```python
prompt = """
Q: {question}

A: Hãy suy nghĩ từng bước:
"""

llm.invoke(prompt.format(question="..."))
```

### 2.2 Few-Shot CoT

Cho ví dụ kèm reasoning:

```python
prompt = """
Q: Tom có 5 viên kẹo, cho Mary 2 viên, mua thêm 3 viên. Tom có mấy viên?
A: Bắt đầu 5 - 2 = 3. Mua thêm 3 → 3 + 3 = 6. Đáp án: 6.

Q: Lan có 10 quyển sách, tặng bạn 3 quyển, mẹ mua thêm gấp đôi số còn lại. Lan có mấy quyển?
A: Hãy suy nghĩ từng bước:
"""
```

---

## 3. ReAct Pattern

**ReAct** = **Re**asoning + **Act**ing. Kết hợp:
- **Thought**: suy luận
- **Action**: gọi tool
- **Observation**: kết quả từ tool

```
Question: Năm sinh của tác giả Harry Potter cộng với 100 là gì?

Thought: Cần tìm năm sinh JK Rowling
Action: wikipedia("J.K. Rowling")
Observation: Born 31 July 1965

Thought: 1965 + 100 = 2065
Action: none
Final Answer: 2065
```

### 3.1 Implementation

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

@tool
def calculate(expression: str) -> str:
    """Tính biểu thức toán"""
    return str(eval(expression))

@tool
def wikipedia(query: str) -> str:
    """Search Wikipedia"""
    # Mock
    if "rowling" in query.lower():
        return "J.K. Rowling, born 31 July 1965, British author of Harry Potter"
    return "Not found"

agent = create_react_agent(
    model=llm,
    tools=[calculate, wikipedia],
)

result = agent.invoke({
    "messages": [("user", "Tác giả Harry Potter sinh năm nào, cộng với 100 = ?")]
})

for m in result["messages"]:
    m.pretty_print()
```

---

## 4. Self-Ask With Search

Pattern: agent tự đặt câu hỏi phụ, search, rồi answer.

```python
self_ask_prompt = """
Trả lời câu hỏi bằng cách tự đặt sub-question và search.

Format:
Question: {question}
Are follow up questions needed: Yes/No
If yes:
  Follow up: <sub-question>
  Intermediate answer: <result from search>
  ...
So the final answer is: <answer>

Question: Thủ tướng Việt Nam hiện tại sinh năm nào?
Are follow up questions needed: Yes
Follow up: Thủ tướng Việt Nam hiện tại là ai?
Intermediate answer: Phạm Minh Chính
Follow up: Phạm Minh Chính sinh năm nào?
Intermediate answer: 1958
So the final answer is: 1958

Question: {real_question}
"""
```

---

## 5. Iterative Refinement

Cải thiện câu trả lời qua nhiều lần.

```python
from langchain_core.prompts import ChatPromptTemplate

# Initial answer
draft_prompt = ChatPromptTemplate.from_template("Trả lời ngắn: {question}")
draft = draft_prompt | llm | StrOutputParser()

# Improve
improve_prompt = ChatPromptTemplate.from_template("""
Cải thiện câu trả lời dựa trên feedback:

Question: {question}
Current answer: {answer}
Feedback: {feedback}

Improved answer:
""")
improver = improve_prompt | llm | StrOutputParser()

# Feedback
feedback_prompt = ChatPromptTemplate.from_template("""
Critique câu trả lời:
Q: {question}
A: {answer}

Liệt kê 2-3 vấn đề có thể cải thiện (hoặc 'good enough'):
""")
critic = feedback_prompt | llm | StrOutputParser()

# Loop
def iterative_refine(question, max_iter=3):
    answer = draft.invoke({"question": question})
    
    for i in range(max_iter):
        fb = critic.invoke({"question": question, "answer": answer})
        if "good enough" in fb.lower():
            break
        answer = improver.invoke({
            "question": question,
            "answer": answer,
            "feedback": fb,
        })
    
    return answer
```

---

## 6. Tree of Thoughts (ToT)

Explore multiple reasoning paths:

```
Question: Sudoku puzzle

Thought 1: Try filling [1,1] = 5
  Thought 1.1: Then [1,2] = ... 
  Thought 1.2: Then [1,3] = ...
Thought 2: Try [1,1] = 3
  ...

Evaluate each path, prune bad ones, continue best.
```

### Simplified ToT

```python
class Thought(BaseModel):
    content: str
    confidence: float

class ThoughtBranch(BaseModel):
    thoughts: List[Thought]

# Generate multiple thoughts
brainstorm_prompt = ChatPromptTemplate.from_template("""
Cho problem, generate 3 different approaches (each as a thought).

Problem: {problem}

3 thoughts với confidence:
""")
brainstormer = brainstorm_prompt | llm.with_structured_output(ThoughtBranch)

# Evaluate
def evaluate_thought(thought, problem):
    eval_prompt = f"Rate approach 0-10:\nProblem: {problem}\nApproach: {thought}"
    return llm.invoke(eval_prompt).content

# Expand best
def tot_solve(problem, depth=3):
    branches = brainstormer.invoke({"problem": problem})
    best = max(branches.thoughts, key=lambda t: t.confidence)
    
    if depth == 0:
        return best.content
    
    # Recurse on best
    return tot_solve(f"{problem}\nApproach: {best.content}", depth - 1)
```

---

## 7. Tool-Augmented Reasoning

Combine reasoning + tools cho exact answer:

### Example: Math Problem

```python
@tool
def python_calc(code: str) -> str:
    """Run Python code"""
    try:
        # Safe eval
        return str(eval(code))
    except Exception as e:
        return f"Error: {e}"

@tool
def lookup_constant(name: str) -> str:
    """Lookup math constant"""
    consts = {"pi": 3.14159, "e": 2.71828, "phi": 1.61803}
    return str(consts.get(name.lower(), "Unknown"))

agent = create_react_agent(llm, [python_calc, lookup_constant])

# "Tính diện tích hình tròn bán kính 5"
# → Thought: Need pi
# → Action: lookup_constant("pi") → 3.14159
# → Thought: area = pi * r^2 = 3.14159 * 25
# → Action: python_calc("3.14159 * 25") → 78.53975
# → Answer: 78.54
```

---

## 8. Reasoning Models (o1, Claude Opus Extended)

OpenAI o1, Claude với extended thinking → tự reason internally, ít cần prompt.

```python
# o1 model
llm = ChatOpenAI(model="o1-preview")
# Không cần "think step by step", model tự reason

# Claude Opus 4.7 extended thinking
llm = ChatAnthropic(
    model="claude-opus-4-7",
    max_tokens=8000,
    thinking={"type": "enabled", "budget_tokens": 5000},
)
```

→ Trade-off: chậm hơn, đắt hơn, nhưng chính xác hơn cho complex reasoning.

---

## 9. Verification & Self-Consistency

### 9.1 Self-Consistency

Gọi LLM nhiều lần, lấy majority vote:

```python
def self_consistency(question, n=5):
    answers = []
    for _ in range(n):
        ans = llm.invoke(f"Q: {question}\nA: think step by step then give final number").content
        # Extract số cuối
        import re
        match = re.search(r'(\d+)\s*$', ans)
        if match:
            answers.append(int(match.group(1)))
    
    # Majority vote
    from collections import Counter
    return Counter(answers).most_common(1)[0][0]
```

### 9.2 Verification

```python
verify_prompt = """
Verify the answer:

Question: {q}
Proposed answer: {a}

Step 1: Check if answer makes sense
Step 2: Re-solve from scratch
Step 3: Compare

If answers match → CORRECT
Otherwise → WRONG, give correct answer
"""
```

---

## 10. Multi-Hop QA

Câu hỏi cần combine info từ nhiều nguồn:

```python
# "Đạo diễn của phim Inception có quốc tịch gì?"

# Step 1: Tìm đạo diễn → Christopher Nolan
# Step 2: Quốc tịch Christopher Nolan → British-American

@tool
def search_movie(name: str) -> str:
    movies = {"Inception": "Director: Christopher Nolan, Year: 2010"}
    return movies.get(name, "Not found")

@tool
def search_person(name: str) -> str:
    people = {"Christopher Nolan": "Nationality: British-American, born 1970"}
    return people.get(name, "Not found")

agent = create_react_agent(llm, [search_movie, search_person])
```

---

## 11. Demo: Math Problem Solver

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

@tool
def python_eval(expression: str) -> str:
    """Evaluate Python math expression safely.
    Examples: '5 * 4', '2 ** 8', 'math.sqrt(16)'"""
    import math
    try:
        return str(eval(expression, {"math": math, "__builtins__": {}}))
    except Exception as e:
        return f"Error: {e}"

@tool
def algebra_solve(equation: str, variable: str = "x") -> str:
    """Solve algebra equation. Use sympy syntax."""
    try:
        from sympy import symbols, solve, sympify
        x = symbols(variable)
        # equation: "x**2 - 4 = 0"
        lhs, rhs = equation.split("=")
        eq = sympify(lhs) - sympify(rhs)
        solutions = solve(eq, x)
        return str(solutions)
    except Exception as e:
        return f"Error: {e}"

agent = create_react_agent(
    model=llm,
    tools=[python_eval, algebra_solve],
    state_modifier="""Bạn là chuyên gia giải toán. Quy tắc:
1. Phân tích đề bài
2. Đặt biến và viết phương trình
3. Dùng tool để giải
4. Verify đáp án
5. Trình bày lời giải chi tiết"""
)

# Test
problems = [
    "Tổng 2 số là 20, tích là 75. Tìm 2 số đó.",
    "Một xe đi 60km/h trong 2h, sau đó 80km/h trong 3h. Quãng đường trung bình?",
    "Lan có gấp 3 lần số kẹo Hà. Sau khi Lan cho Hà 5 viên, hai bạn có số kẹo bằng nhau. Tìm số kẹo ban đầu.",
]

for p in problems:
    print(f"\n=== {p} ===")
    result = agent.invoke({"messages": [("user", p)]})
    print(result["messages"][-1].content)
```

---

## 12. Best Practices

✅ **Explicit reasoning trigger**: "Hãy suy nghĩ từng bước"

✅ **Verify với tool**: cho mọi math/code problem

✅ **Limit iterations**: tránh loop vô hạn (max 5-10)

✅ **Show reasoning to user**: dễ debug, trust

✅ **Self-consistency cho critical**: gọi nhiều lần

✅ **Use reasoning model khi cần**: o1, Claude Opus extended thinking

---

## 13. Bài Tập

### Bài 1: Math Olympiad
Build agent giải 5 bài olympiad lớp 8 dùng python_eval + sympy. Compare với plain LLM.

### Bài 2: Multi-Hop Trivia
"Diễn viên chính của phim đã đạt Oscar 2020 sinh ở thành phố nào?"
Build agent với search tool.

### Bài 3: Self-Consistency Test
Cho 20 math problems, đo accuracy với:
- 1 LLM call
- Self-consistency n=5
- Tool-augmented (python_eval)

---

## 14. Checklist

- [ ] Chain-of-Thought prompting
- [ ] ReAct pattern
- [ ] Self-ask
- [ ] Iterative refinement
- [ ] Tree of Thoughts cơ bản
- [ ] Self-consistency
- [ ] Multi-hop reasoning
- [ ] Tool-augmented math

➡️ **Tiếp theo**: [Agent Memory](./05-Agent-Memory.md)
