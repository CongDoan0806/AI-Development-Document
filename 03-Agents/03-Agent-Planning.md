# Agent Planning - Lập Kế Hoạch

## 1. Tại Sao Cần Planning?

Agent đơn thuần phản ứng (reactive) - nghĩ và làm từng bước:
```
User: "Lên kế hoạch trip Đà Lạt 3 ngày"
Agent: search_flights → book_flight → ??? (mất plan)
```

Agent có planning - nghĩ tổng thể trước:
```
Step 1: Plan
  - Tìm flights
  - Đặt khách sạn
  - Tạo itinerary từng ngày
  - Đặt vé tham quan
Step 2: Execute từng step
```

**Benefits**:
- Coherent cho task phức tạp
- Tránh sót steps
- Có thể optimize (parallel execution)
- Dễ recover khi fail

---

## 2. Plan-and-Execute Pattern

### 2.1 Ý Tưởng

```
[Planner LLM] → Generate plan (list of steps)
       ↓
[Executor LLM] → Execute từng step với tools
       ↓
[Final Answer]
```

### 2.2 Implementation

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from pydantic import BaseModel, Field
from typing import List

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# === Planner ===
class Plan(BaseModel):
    steps: List[str] = Field(description="Các bước cần thực hiện, theo thứ tự")

planner_prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là expert lập kế hoạch. Chia task phức tạp thành các bước cụ thể, có thể thực thi.

Mỗi step phải:
- Action cụ thể (không vague)
- Có thể làm bằng 1 tool call hoặc 1 LLM call
- Theo thứ tự logic
- Tối đa 5-7 steps"""),
    ("user", "{task}")
])

planner = planner_prompt | llm.with_structured_output(Plan)

# Test
plan = planner.invoke({"task": "Tổ chức birthday party cho con 10 tuổi"})
print(plan.steps)
# [
#   "Chọn theme party (e.g., siêu nhân, công chúa)",
#   "Lên danh sách khách mời",
#   "Đặt venue/nhà hàng",
#   "Đặt bánh kem theo theme",
#   "Chuẩn bị decoration",
#   "Gửi invitation",
#   "Lên timeline ngày party"
# ]
```

---

## 3. Plan With Tool Calling

```python
@tool
def search_venue(location: str, capacity: int) -> str:
    """Search party venue"""
    return f"Venue: ABC Restaurant, fits {capacity}, in {location}"

@tool
def order_cake(theme: str, size: str) -> str:
    """Order birthday cake"""
    return f"Cake ordered: {theme} theme, {size}"

@tool
def send_invitation(recipients: List[str], message: str) -> str:
    """Send invitations"""
    return f"Invited {len(recipients)} guests"

# Executor uses tools based on plan step
executor_prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là executor. Thực hiện step cụ thể được giao bằng tools.
Trả về result ngắn gọn."""),
    ("user", "Current step: {step}\nPlan context: {context}")
])

executor = create_react_agent(llm, tools=[search_venue, order_cake, send_invitation])

# Execute plan
def execute_plan(plan: Plan, context=""):
    results = []
    for i, step in enumerate(plan.steps):
        print(f"\n=== Step {i+1}: {step} ===")
        result = executor.invoke({
            "messages": [
                ("user", f"Step: {step}\nContext so far: {context}")
            ]
        })
        final_msg = result["messages"][-1].content
        results.append({"step": step, "result": final_msg})
        context += f"\nStep {i+1} done: {final_msg}"
    return results

results = execute_plan(plan)
```

---

## 4. ReAct vs Plan-and-Execute

| | ReAct | Plan-and-Execute |
|---|-------|------------------|
| **Strategy** | Reactive, 1 step ahead | Plan all upfront |
| **Better for** | Exploratory, short tasks | Long, complex tasks |
| **Cost** | Có thể loop dài | Predictable steps |
| **Flexibility** | High - thay đổi dễ | Low - cứng theo plan |
| **Recovery** | Tự động | Cần re-plan |

---

## 5. Plan-Execute-Replan (PEER)

Khi 1 step fail → re-plan phần còn lại.

```python
def plan_execute_replan(task, max_iter=3):
    plan = planner.invoke({"task": task})
    results = []
    
    for iteration in range(max_iter):
        print(f"\n=== Iteration {iteration+1} ===")
        print(f"Plan: {plan.steps}")
        
        success = True
        for step in plan.steps:
            try:
                result = execute_step(step, results)
                results.append({"step": step, "result": result, "status": "ok"})
            except Exception as e:
                results.append({"step": step, "status": "failed", "error": str(e)})
                success = False
                break  # Stop và re-plan
        
        if success:
            return results
        
        # Re-plan
        plan = re_planner.invoke({
            "task": task,
            "completed": results,
            "failure": results[-1],
        })
    
    return results
```

---

## 6. LangGraph Plan-Execute Agent

LangGraph cung cấp template sẵn:

```python
from typing import TypedDict, Annotated, List
from langgraph.graph import StateGraph, END
from operator import add

class PlanExecuteState(TypedDict):
    input: str
    plan: List[str]
    past_steps: Annotated[List[tuple], add]
    response: str

# Nodes
def plan_step(state):
    plan = planner.invoke({"task": state["input"]})
    return {"plan": plan.steps}

def execute_step(state):
    task = state["plan"][0]
    result = executor.invoke({"messages": [("user", task)]})
    return {
        "past_steps": [(task, result["messages"][-1].content)]
    }

def should_continue(state):
    if len(state["plan"]) == 0:
        return END
    return "execute"

# Replan or finish
def replan_step(state):
    # Check if done, or update plan
    remaining = state["plan"][1:]  # Drop done step
    if not remaining:
        # Done
        return {"response": "All steps complete", "plan": []}
    return {"plan": remaining}

# Build graph
workflow = StateGraph(PlanExecuteState)
workflow.add_node("planner", plan_step)
workflow.add_node("execute", execute_step)
workflow.add_node("replan", replan_step)

workflow.set_entry_point("planner")
workflow.add_edge("planner", "execute")
workflow.add_edge("execute", "replan")
workflow.add_conditional_edges(
    "replan",
    should_continue,
    {"execute": "execute", END: END}
)

app = workflow.compile()

# Use
config = {"recursion_limit": 50}
result = app.invoke({"input": "Plan a trip to Da Lat"}, config=config)
```

→ Sẽ học LangGraph kỹ ở [Phase 4](../04-LangGraph/01-Gioi-Thieu-LangGraph.md)

---

## 7. Tree-of-Thoughts (ToT)

Thay vì 1 plan tuyến tính, generate **nhiều alternative plans**, đánh giá, chọn best.

```
Task: "Solve X"

Plan A: step1 → step2 → step3
Plan B: step1' → step2' → step3'
Plan C: ...

Evaluate each: A=0.7, B=0.9, C=0.4
→ Execute Plan B
```

```python
class MultiplePlans(BaseModel):
    plans: List[Plan]
    reasoning: str

multi_planner = planner_prompt | llm.with_structured_output(MultiplePlans)

evaluator_prompt = ChatPromptTemplate.from_template("""
Đánh giá plan này theo các tiêu chí:
1. Feasibility (0-10)
2. Efficiency (0-10)
3. Risk (0-10, càng thấp càng tốt)

Plan: {plan}
Task: {task}

Output JSON: {{"feasibility": int, "efficiency": int, "risk": int, "overall": float}}
""")

evaluator = evaluator_prompt | llm | JsonOutputParser()

def tree_of_thoughts(task):
    # 1. Generate multiple plans
    multi_plans = multi_planner.invoke({"task": task})
    
    # 2. Evaluate each
    scored = []
    for plan in multi_plans.plans:
        score = evaluator.invoke({"plan": plan.steps, "task": task})
        scored.append((plan, score["overall"]))
    
    # 3. Pick best
    best = max(scored, key=lambda x: x[1])
    return best[0]
```

---

## 8. Self-Reflection / Self-Critique

Agent đánh giá output của chính nó và sửa.

```python
critique_prompt = ChatPromptTemplate.from_template("""
Bạn là QA. Review output dưới đây và chỉ ra:
- Điểm tốt
- Vấn đề / sai sót
- Đề xuất cải thiện

Task: {task}
Output: {output}

Critique:
""")

critic = critique_prompt | llm | StrOutputParser()

revise_prompt = ChatPromptTemplate.from_template("""
Revise output dựa trên critique.

Task: {task}
Original output: {output}
Critique: {critique}

Revised output:
""")

reviser = revise_prompt | llm | StrOutputParser()

def reflect_and_improve(task, output, max_iter=3):
    for i in range(max_iter):
        critique = critic.invoke({"task": task, "output": output})
        
        # Check if good enough
        if "no issues" in critique.lower() or "perfect" in critique.lower():
            return output
        
        # Revise
        output = reviser.invoke({
            "task": task,
            "output": output,
            "critique": critique,
        })
    
    return output
```

---

## 9. Subtask Decomposition

Task lớn → break thành subtasks → solve từng cái.

```python
class Subtask(BaseModel):
    id: int
    description: str
    depends_on: List[int] = Field(default_factory=list)
    
class TaskGraph(BaseModel):
    subtasks: List[Subtask]

decompose_prompt = ChatPromptTemplate.from_template("""
Chia task thành subtasks, có dependency. Mỗi subtask phải:
- Atomic (không chia nhỏ nữa)
- Có deps đến subtask khác (nếu cần)

Task: {task}

Output JSON với subtasks và depends_on.
""")

decomposer = decompose_prompt | llm.with_structured_output(TaskGraph)

# Use
graph = decomposer.invoke({"task": "Build a web app for blog"})
for st in graph.subtasks:
    print(f"#{st.id}: {st.description} (deps: {st.depends_on})")

# Execute theo dependency order (topological sort)
def execute_graph(graph: TaskGraph):
    completed = {}
    while len(completed) < len(graph.subtasks):
        for st in graph.subtasks:
            if st.id in completed:
                continue
            # Check all deps done
            if all(d in completed for d in st.depends_on):
                result = execute_subtask(st)
                completed[st.id] = result
    return completed
```

---

## 10. Demo: Travel Planner Agent

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from pydantic import BaseModel, Field
from typing import List

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# === Tools ===
@tool
def search_flight(origin: str, destination: str, date: str) -> str:
    """Search flights"""
    return f"Flight {origin}-{destination} on {date}: $500"

@tool
def book_hotel(city: str, checkin: str, nights: int) -> str:
    """Book hotel"""
    return f"Hotel in {city} from {checkin} for {nights} nights: $300"

@tool
def get_attractions(city: str) -> str:
    """Get attractions in city"""
    return f"Top in {city}: Hồ Xuân Hương, Thung Lũng Tình Yêu, Datanla"

@tool
def make_itinerary(day: int, activities: str) -> str:
    """Save itinerary for a day"""
    return f"Day {day}: {activities}"

tools = [search_flight, book_hotel, get_attractions, make_itinerary]

# === Plan-and-Execute ===
class Step(BaseModel):
    description: str
    tool_needed: str = Field(description="Tool name or 'none' if reasoning only")

class TravelPlan(BaseModel):
    steps: List[Step]
    estimated_budget: str

planner_prompt = ChatPromptTemplate.from_messages([
    ("system", "Bạn là travel planner. Tạo plan chi tiết với tools cần dùng."),
    ("user", "Plan trip: {request}")
])
planner = planner_prompt | llm.with_structured_output(TravelPlan)

executor = create_react_agent(llm, tools)

# === Main ===
def travel_agent(request: str):
    # Plan
    plan = planner.invoke({"request": request})
    print(f"📋 Plan: {plan.estimated_budget}")
    for i, s in enumerate(plan.steps, 1):
        print(f"  {i}. {s.description} (tool: {s.tool_needed})")
    
    # Execute
    print("\n🚀 Executing...")
    context = ""
    for i, step in enumerate(plan.steps, 1):
        print(f"\nStep {i}: {step.description}")
        result = executor.invoke({
            "messages": [("user", f"{step.description}\nContext: {context}")]
        })
        msg = result["messages"][-1].content
        print(f"  → {msg}")
        context += f"\nStep {i}: {msg}"
    
    return context

# Run
result = travel_agent("3 ngày Đà Lạt từ Hà Nội, ngày 1/12/2024, ngân sách 5tr/người")
```

---

## 11. Best Practices

✅ **Limit plan steps**: 5-10 max, longer = brittle

✅ **Verify plan trước execute**: human-in-the-loop cho important tasks

✅ **Plan revision**: cho phép update plan dựa trên observations

✅ **Tool-aware planning**: planner biết available tools

✅ **State tracking**: lưu kết quả mỗi step để debug

---

## 12. Bài Tập

### Bài 1: Recipe Planner
Input: ingredients có sẵn + dietary restrictions
Output: meal plan 1 tuần với recipe và shopping list

### Bài 2: Code Project Setup
Plan-execute để setup 1 project mới:
- Init git
- Create folders
- Install dependencies
- Generate boilerplate code
- Create README

### Bài 3: Multi-Step Research
Agent tự plan research về 1 topic:
- Search 10 nguồn
- Tóm tắt từng cái
- Identify common themes
- Tạo summary cuối

---

## 13. Checklist

- [ ] Hiểu Plan-and-Execute pattern
- [ ] So sánh được với ReAct
- [ ] Build plan với structured output
- [ ] Execute plan với tools
- [ ] Plan-Execute-Replan loop
- [ ] Self-reflection
- [ ] Tree of Thoughts cơ bản
- [ ] Task decomposition

➡️ **Tiếp theo**: [Multi-Step Reasoning](./04-Multi-Step-Reasoning.md)
