# Multi-Agent System

## 1. Tại Sao Cần Multi-Agent?

1 agent với 20 tools = confusing. Tách thành 3 agents chuyên môn:
- 🔍 **Researcher**: tìm thông tin
- 💻 **Coder**: viết code
- 📝 **Reviewer**: kiểm tra chất lượng

**Lợi ích**:
- Specialization → tốt hơn
- Modularity → maintain dễ
- Parallel work
- Distinct prompts cho từng role

---

## 2. Patterns

### 2.1 Supervisor (Hierarchical)

```
       Supervisor
      /    |    \
 Worker1 Worker2 Worker3
```

Supervisor route task tới worker phù hợp.

### 2.2 Network (Collaborative)

```
Agent A ⇄ Agent B
   ⇅       ⇅
Agent C ⇄ Agent D
```

Agents tự decide gửi cho ai tiếp.

### 2.3 Sequential Pipeline

```
A → B → C → D
```

Mỗi agent xử lý phần của mình.

---

## 3. Supervisor Pattern

```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from pydantic import BaseModel

llm = ChatOpenAI(model="gpt-4o-mini")

class State(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str

# === Supervisor ===
class Router(BaseModel):
    """Decide next worker"""
    next: Literal["researcher", "coder", "reviewer", "FINISH"]

def supervisor(state):
    response = llm.with_structured_output(Router).invoke([
        ("system", """Bạn là supervisor. Dựa vào conversation, chọn worker tiếp theo:
        - researcher: tìm thông tin
        - coder: viết code
        - reviewer: review code
        - FINISH: task hoàn tất"""),
        *state["messages"],
    ])
    return {"next_agent": response.next}

# === Workers ===
def researcher(state):
    response = llm.invoke([
        ("system", "Bạn là researcher. Tìm thông tin cần thiết."),
        *state["messages"],
    ])
    return {"messages": [response]}

def coder(state):
    response = llm.invoke([
        ("system", "Bạn là Python expert. Viết code chất lượng."),
        *state["messages"],
    ])
    return {"messages": [response]}

def reviewer(state):
    response = llm.invoke([
        ("system", "Bạn là code reviewer. Đưa ra feedback."),
        *state["messages"],
    ])
    return {"messages": [response]}

# === Routing ===
def route(state):
    if state["next_agent"] == "FINISH":
        return END
    return state["next_agent"]

# === Graph ===
graph = StateGraph(State)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", researcher)
graph.add_node("coder", coder)
graph.add_node("reviewer", reviewer)

graph.set_entry_point("supervisor")
graph.add_conditional_edges("supervisor", route, {
    "researcher": "researcher",
    "coder": "coder",
    "reviewer": "reviewer",
    END: END,
})

# Workers report back to supervisor
graph.add_edge("researcher", "supervisor")
graph.add_edge("coder", "supervisor")
graph.add_edge("reviewer", "supervisor")

app = graph.compile()

# Run
result = app.invoke({
    "messages": [("user", "Viết function Python tính fibonacci và review nó")]
})

for m in result["messages"]:
    m.pretty_print()
```

---

## 4. Network Pattern (Hand-off)

```python
from langgraph.types import Command

class State(TypedDict):
    messages: Annotated[list, add_messages]

def agent_a(state):
    response = llm.invoke([
        ("system", """Bạn là Agent A chuyên về phần A.
        Nếu cần Agent B, kết thúc với "TRANSFER_TO_B"
        Nếu cần Agent C, kết thúc với "TRANSFER_TO_C"
        Nếu xong, kết thúc với "DONE" """),
        *state["messages"],
    ])
    
    content = response.content
    if "TRANSFER_TO_B" in content:
        return Command(goto="agent_b", update={"messages": [response]})
    elif "TRANSFER_TO_C" in content:
        return Command(goto="agent_c", update={"messages": [response]})
    return Command(goto=END, update={"messages": [response]})

# Tương tự cho agent_b, agent_c

graph = StateGraph(State)
graph.add_node("agent_a", agent_a)
graph.add_node("agent_b", agent_b)
graph.add_node("agent_c", agent_c)

graph.set_entry_point("agent_a")

app = graph.compile()
```

---

## 5. Subgraph Pattern

Agent = subgraph độc lập với state riêng:

```python
# === Subgraph: Research Team ===
class ResearchState(TypedDict):
    query: str
    sources: list
    summary: str

def search(state):
    return {"sources": ["doc1", "doc2"]}

def synthesize(state):
    return {"summary": "Combined info"}

research_graph = StateGraph(ResearchState)
research_graph.add_node("search", search)
research_graph.add_node("synthesize", synthesize)
research_graph.set_entry_point("search")
research_graph.add_edge("search", "synthesize")
research_graph.add_edge("synthesize", END)
research_subgraph = research_graph.compile()

# === Main Graph ===
class MainState(TypedDict):
    messages: Annotated[list, add_messages]
    research_summary: str

def call_research_team(state):
    last_msg = state["messages"][-1].content
    
    # Call subgraph
    result = research_subgraph.invoke({"query": last_msg})
    
    return {"research_summary": result["summary"]}

main_graph = StateGraph(MainState)
main_graph.add_node("research", call_research_team)
main_graph.set_entry_point("research")
main_graph.add_edge("research", END)
```

---

## 6. Sequential Pipeline

```python
class PipelineState(TypedDict):
    raw_input: str
    cleaned: str
    analyzed: dict
    final_report: str

def cleaner(state):
    cleaned = state["raw_input"].strip().lower()
    return {"cleaned": cleaned}

def analyzer(state):
    # NER, sentiment, ...
    return {"analyzed": {"sentiment": "positive", "entities": [...]}}

def reporter(state):
    report = f"Analysis: {state['analyzed']}"
    return {"final_report": report}

graph = StateGraph(PipelineState)
graph.add_node("clean", cleaner)
graph.add_node("analyze", analyzer)
graph.add_node("report", reporter)

graph.set_entry_point("clean")
graph.add_edge("clean", "analyze")
graph.add_edge("analyze", "report")
graph.add_edge("report", END)
```

---

## 7. Demo: Customer Support Multi-Agent

```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from pydantic import BaseModel

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class State(TypedDict):
    messages: Annotated[list, add_messages]
    category: str
    resolved: bool

class Categorize(BaseModel):
    category: Literal["billing", "technical", "general"]

# === Triage Agent ===
def triage(state):
    response = llm.with_structured_output(Categorize).invoke([
        ("system", "Phân loại customer issue."),
        *state["messages"],
    ])
    return {"category": response.category}

# === Specialist Agents ===
def billing_specialist(state):
    response = llm.invoke([
        ("system", """Bạn là chuyên gia billing. 
        Xử lý: refund, payment, invoice, subscription.
        Nếu vấn đề ngoài scope, nói 'NEEDS_ESCALATION'."""),
        *state["messages"],
    ])
    return {
        "messages": [response],
        "resolved": "NEEDS_ESCALATION" not in response.content,
    }

def tech_specialist(state):
    response = llm.invoke([
        ("system", """Bạn là technical support.
        Xử lý: bugs, integration, API issues.
        Nếu cần escalate, nói 'NEEDS_ESCALATION'."""),
        *state["messages"],
    ])
    return {
        "messages": [response],
        "resolved": "NEEDS_ESCALATION" not in response.content,
    }

def general_support(state):
    response = llm.invoke([
        ("system", "Bạn là general customer support. Trả lời thân thiện."),
        *state["messages"],
    ])
    return {"messages": [response], "resolved": True}

# === Escalation ===
def human_escalation(state):
    return {
        "messages": [("assistant", "Đã chuyển case cho human support. Sẽ liên hệ trong 24h.")],
        "resolved": True,
    }

# === Routing ===
def route_by_category(state):
    return {
        "billing": "billing",
        "technical": "tech",
        "general": "general",
    }[state["category"]]

def check_resolved(state):
    if state["resolved"]:
        return END
    return "escalate"

# === Graph ===
graph = StateGraph(State)
graph.add_node("triage", triage)
graph.add_node("billing", billing_specialist)
graph.add_node("tech", tech_specialist)
graph.add_node("general", general_support)
graph.add_node("escalate", human_escalation)

graph.set_entry_point("triage")
graph.add_conditional_edges("triage", route_by_category, {
    "billing": "billing",
    "tech": "tech",
    "general": "general",
})

# Check resolved sau billing/tech
graph.add_conditional_edges("billing", check_resolved, {"escalate": "escalate", END: END})
graph.add_conditional_edges("tech", check_resolved, {"escalate": "escalate", END: END})
graph.add_edge("general", END)
graph.add_edge("escalate", END)

app = graph.compile()

# Test
queries = [
    "Tôi muốn refund đơn hàng #12345",
    "API trả về 500 error khi tôi call endpoint /users",
    "Cảm ơn dịch vụ của các bạn!",
]

for q in queries:
    print(f"\n{'='*50}\nQ: {q}")
    result = app.invoke({"messages": [("user", q)], "resolved": False})
    print(f"Category: {result['category']}")
    print(f"Response: {result['messages'][-1].content}")
```

---

## 8. Shared State vs Separate State

### Shared State
Tất cả agents thấy cùng messages → context full nhưng có thể confusing.

### Separate State (Subgraph)
Mỗi agent có state riêng, chỉ trao đổi qua interface → modular hơn.

---

## 9. Best Practices

✅ **Clear roles**: mỗi agent 1 chuyên môn

✅ **Limit handoffs**: max 5-7 transfers tránh loop

✅ **Supervisor cho complex workflows**

✅ **Subgraph cho independent teams**

✅ **Pass minimal context**: tránh agent quá tải

✅ **Monitor handoffs**: log để debug

---

## 10. Checklist

- [ ] Supervisor pattern
- [ ] Network pattern với Command
- [ ] Subgraph composition
- [ ] Sequential pipeline
- [ ] Specialist agents với system prompt riêng
- [ ] Escalation pattern

➡️ **Tiếp theo**: [Persistence](./07-Persistence.md)
