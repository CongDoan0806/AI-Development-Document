# Advanced Reasoning Techniques

## 1. Tree of Thoughts (ToT)

**Tree of Thoughts** = Khám phá nhiều **reasoning paths** thay vì 1 đường duy nhất, evaluate, chọn best.

### Ý Tưởng

```
CoT: 1 path linear
   Start → Thought 1 → Thought 2 → Answer

ToT: cây nhiều paths
   Start
   ├── Approach A → ... → Score
   ├── Approach B → ... → Score
   └── Approach C → ... → Score
   → Pick best
```

### Implementation

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel
from typing import List

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

class Thought(BaseModel):
    approach: str
    reasoning: str
    confidence: float

class ThoughtsList(BaseModel):
    thoughts: List[Thought]

# Step 1: Generate multiple approaches
def generate_thoughts(problem, n=3):
    prompt = f"""
    Problem: {problem}
    
    Generate {n} different approaches to solve this problem.
    For each: describe approach, reasoning, confidence (0-1).
    """
    return llm.with_structured_output(ThoughtsList).invoke(prompt)

# Step 2: Evaluate
def evaluate_thought(thought, problem):
    prompt = f"""
    Problem: {problem}
    Approach: {thought.approach}
    Reasoning: {thought.reasoning}
    
    Evaluate this approach:
    - Feasibility (0-10)
    - Likely correctness (0-10)
    - Efficiency (0-10)
    
    Output JSON: {{"feasibility": ..., "correctness": ..., "efficiency": ..., "overall": ...}}
    """
    return llm.invoke(prompt)

# Step 3: Select best & continue
def tot_solve(problem, depth=2):
    if depth == 0:
        return llm.invoke(f"Solve: {problem}").content
    
    thoughts = generate_thoughts(problem)
    scored = [(t, evaluate_thought(t, problem)) for t in thoughts.thoughts]
    best = max(scored, key=lambda x: x[1]["overall"])
    
    # Recurse với best approach
    return tot_solve(
        f"{problem}\nUsing approach: {best[0].approach}",
        depth - 1
    )
```

### Use Cases
- Math olympiad problems
- Strategy games (chess, Sudoku)
- Creative writing exploration
- Complex optimization

---

## 2. ReAct (Reasoning + Acting)

**ReAct** kết hợp **suy luận** (Thought) + **hành động** (Action) + **quan sát** (Observation).

→ Chi tiết ở [Phase 3 - Agents](../03-Agents/04-Multi-Step-Reasoning.md)

### Format

```
Thought: Tôi cần tìm tuổi của X
Action: search("X age")
Observation: 35 years old

Thought: Bây giờ tính 35 + 10 = 45
Action: calculator("35 + 10")
Observation: 45

Thought: Có đủ info để trả lời
Final Answer: 45
```

### Prompt Template

```
Trả lời câu hỏi dùng các tools sau:
{tool_descriptions}

Theo format chính xác:
Question: {question}
Thought: [bạn suy nghĩ gì]
Action: [tool_name]
Action Input: [input cho tool]
Observation: [kết quả từ tool]
... (lặp Thought/Action/Observation)
Thought: Tôi đã có đủ info
Final Answer: [câu trả lời]

Question: {input}
```

---

## 3. Self-Consistency

→ Đã giới thiệu ở [CoT](./04-Chain-of-Thought.md#6-self-consistency-với-cot).

**Recap**: gọi LLM N lần với temperature > 0, lấy majority vote.

```python
def self_consistency(prompt, n=5, temperature=0.7):
    answers = []
    for _ in range(n):
        ans = llm.invoke(prompt, temperature=temperature).content
        answers.append(extract_final(ans))
    
    from collections import Counter
    return Counter(answers).most_common(1)[0][0]
```

---

## 4. Reflexion - Self-Critique & Improve

**Reflexion** = sau khi generate answer, **tự critique** và **revise**.

### Pattern

```
Step 1: GENERATE - Draft answer
Step 2: REFLECT - Identify issues
Step 3: REVISE - Generate improved version
Step 4: (Optional) Repeat
```

### Implementation

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# 1. Generate
draft_prompt = ChatPromptTemplate.from_template(
    "Trả lời câu hỏi: {question}"
)
draft_chain = draft_prompt | llm | StrOutputParser()

# 2. Reflect
reflect_prompt = ChatPromptTemplate.from_template("""
Critique câu trả lời sau:

Question: {question}
Answer: {answer}

Tìm:
1. Factual errors
2. Missing information  
3. Unclear reasoning
4. Better alternatives

Critique:
""")
reflect_chain = reflect_prompt | llm | StrOutputParser()

# 3. Revise
revise_prompt = ChatPromptTemplate.from_template("""
Revise câu trả lời dựa trên critique:

Question: {question}
Original answer: {answer}
Critique: {critique}

Improved answer:
""")
revise_chain = revise_prompt | llm | StrOutputParser()

# Pipeline
def reflexion(question, iterations=2):
    answer = draft_chain.invoke({"question": question})
    
    for i in range(iterations):
        critique = reflect_chain.invoke({
            "question": question,
            "answer": answer,
        })
        
        # If critique says "good", stop
        if "no major issues" in critique.lower():
            break
        
        answer = revise_chain.invoke({
            "question": question,
            "answer": answer,
            "critique": critique,
        })
    
    return answer

# Use
final = reflexion("Giải thích quantum entanglement cho người không học vật lý")
```

---

## 5. Step-Back Prompting

**Step-back** = trước khi giải question cụ thể, hỏi câu **tổng quát hơn** để có background.

### Ví Dụ

```
Specific Q: "Năm 2009, Estella Leopold học trường nào?"

Step-back Q: "Estella Leopold có background giáo dục thế nào?"
→ Lấy general info về Estella Leopold's education
→ Sau đó answer câu cụ thể có context
```

### Implementation

```python
stepback_prompt = ChatPromptTemplate.from_template("""
Cho câu hỏi cụ thể, đặt câu hỏi tổng quát hơn để lấy background:

Specific question: {question}
Step-back question:
""")

stepback = stepback_prompt | llm | StrOutputParser()

# Pipeline
def stepback_qa(question):
    # 1. Generate stepback question
    general_q = stepback.invoke({"question": question})
    
    # 2. Answer general
    general_answer = llm.invoke(general_q).content
    
    # 3. Answer specific với context
    final = llm.invoke(f"""
    Context: {general_answer}
    
    Specific question: {question}
    Answer:
    """).content
    
    return final
```

**Hiệu quả**: tăng accuracy cho factual queries 5-15%.

---

## 6. Skeleton-of-Thought (SoT)

Generate **skeleton/outline** trước, sau đó fill từng phần (có thể parallel).

```
Step 1: Generate outline (5 main points)
Step 2: Generate detail cho mỗi point (parallel)
Step 3: Combine
```

### Implementation

```python
import asyncio

outline_prompt = """
Generate outline cho bài essay về: {topic}
Output JSON: ["point 1", "point 2", ...]
"""

detail_prompt = """
Topic: {topic}
Point to elaborate: {point}

Viết 1 đoạn 100-150 từ:
"""

async def sot_essay(topic):
    # 1. Outline
    outline = await llm.with_structured_output(List[str]).ainvoke(
        outline_prompt.format(topic=topic)
    )
    
    # 2. Parallel generation
    details = await asyncio.gather(*[
        llm.ainvoke(detail_prompt.format(topic=topic, point=p))
        for p in outline
    ])
    
    # 3. Combine
    return "\n\n".join(d.content for d in details)
```

---

## 7. ReWOO (Reasoning WithOut Observation)

Tách reasoning khỏi execution. LLM lập plan với placeholder, sau đó execute.

### Pattern

```
Plan:
Step 1: search("X") → #E1
Step 2: search(#E1 author) → #E2  
Step 3: calculate(#E2 year + 100)

Execute:
#E1 = "X" search results
#E2 = "Author Y" 
Final = year_of_Y + 100
```

**Lợi ích**: chỉ 1 LLM call lập plan, sau đó N tool calls (không cần LLM mỗi step).

---

## 8. Constitutional AI

LLM **tự apply rules** (constitution) để critique và revise output.

### Constitution Example
```
Principles:
1. Be helpful
2. Don't share PII
3. Don't promote illegal activity
4. Cite sources for facts
5. Acknowledge uncertainty
```

### Workflow
```
1. Generate response
2. Self-critique: "Does this violate any principle?"
3. If yes → revise
4. Repeat until clean
```

### Implementation

```python
constitution = """
Principles:
1. Be helpful and accurate
2. Don't share PII (names, addresses, phones)
3. Don't make up facts
4. Cite sources
5. Acknowledge uncertainty with "I'm not sure"
"""

critique_prompt = f"""
{constitution}

Response: {{response}}

Does this response violate any principle? List violations or "NO_VIOLATIONS".
"""

revise_prompt = f"""
{constitution}

Original response: {{response}}
Violations: {{violations}}

Revised response (fix violations):
"""

def constitutional_response(question):
    response = llm.invoke(question).content
    
    for _ in range(3):
        violations = llm.invoke(critique_prompt.format(response=response)).content
        if "NO_VIOLATIONS" in violations:
            break
        
        response = llm.invoke(revise_prompt.format(
            response=response,
            violations=violations
        )).content
    
    return response
```

---

## 9. Prompt Chaining

Chia task phức tạp thành **chain of simpler prompts**.

### Pattern: Sequential

```python
# Prompt 1: Extract entities
entities = llm.invoke(f"Extract entities from: {text}").content

# Prompt 2: Classify each
classifications = llm.invoke(f"Classify each entity: {entities}").content

# Prompt 3: Summarize
summary = llm.invoke(f"Summarize findings: {classifications}").content
```

### Pattern: Conditional

```python
# Prompt 1: Classify question type
q_type = llm.invoke(f"Classify: factual/opinion/code: {question}").content

# Branch based on type
if q_type == "factual":
    result = factual_chain.invoke(question)
elif q_type == "code":
    result = code_chain.invoke(question)
else:
    result = opinion_chain.invoke(question)
```

### Pattern: Map-Reduce

```python
# Split into chunks
chunks = split_document(long_doc)

# Map: process each chunk
summaries = [
    llm.invoke(f"Summarize: {chunk}").content
    for chunk in chunks
]

# Reduce: combine
final = llm.invoke(f"Combine these summaries: {summaries}").content
```

---

## 10. Meta-Prompting

LLM **viết prompt** cho LLM khác.

### Use Case
- Tự generate prompt cho new task
- Optimize prompts dựa trên examples
- Adapt prompts cho domain

### Example

```python
meta_prompt = """
Bạn là expert prompt engineer.

Task description: {task}
Examples of desired input-output: {examples}

Generate a prompt template that:
1. Includes role, instructions, format
2. Will work with GPT-4o-mini  
3. Is concise (under 200 words)
4. Uses {{variables}} as placeholders

Output prompt template:
"""

# Generate optimized prompt
optimized = llm.invoke(meta_prompt.format(
    task="Classify customer emails into 5 categories",
    examples="...",
)).content
```

---

## 11. Combining Techniques

Production prompts thường combine nhiều techniques:

### Example: Advanced Research Agent

```python
research_prompt = """
[ROLE]
Bạn là research assistant với PhD level expertise.

[INSTRUCTIONS]
Khi nhận research question:

1. STEP-BACK: Ask broader context question first
2. PLAN: List 3-5 sub-questions to explore
3. REACT: For each sub-question, use tools to find info
4. SELF-CONSISTENCY: For critical facts, verify từ 2+ nguồn
5. SYNTHESIZE: Combine findings
6. REFLECT: Critique your answer, identify gaps
7. REVISE: Final polished answer

[CONSTRAINTS]
- Cite sources [n]
- Acknowledge uncertainty
- Don't include opinions, only facts

[FORMAT]
## Background
[step-back context]

## Findings
[per sub-question]

## Synthesis
[combined answer]

## Confidence
[scale 0-1] + reasons
"""
```

---

## 12. Bài Tập

### Bài 1: ToT For Strategy
Implement ToT để chơi tic-tac-toe simple. Explore 3 moves at each turn, evaluate, pick best.

### Bài 2: Reflexion Loop
Build Q&A system với reflexion (max 3 iterations). Test với 20 tricky questions.

### Bài 3: Constitutional AI
Define 10 principles for medical advice chatbot. Implement constitutional pipeline.

---

## 13. So Sánh Techniques

| Technique | Best For | Cost | Quality Gain |
|-----------|----------|------|--------------|
| **CoT** | Reasoning, math | 2-3× | +20% |
| **Self-Consistency** | Critical accuracy | 5× | +5-15% |
| **ToT** | Optimization, strategy | 10×+ | +10-20% |
| **ReAct** | Tool use | 3-5× | Required |
| **Reflexion** | Quality output | 3× | +10-15% |
| **Step-back** | Factual queries | 2× | +5-15% |

---

## 14. Checklist

- [ ] Hiểu Tree of Thoughts
- [ ] ReAct pattern
- [ ] Self-Consistency
- [ ] Reflexion - self-critique loop
- [ ] Step-back prompting
- [ ] Skeleton-of-Thought
- [ ] Constitutional AI
- [ ] Prompt chaining (sequential, conditional, map-reduce)
- [ ] Meta-prompting

➡️ **Tiếp theo**: [Role & Personas](./06-Role-Personas.md)
