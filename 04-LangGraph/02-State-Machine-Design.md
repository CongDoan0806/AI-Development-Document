# State Machine Design

## 1. State Schema

State là "trái tim" của LangGraph. Schema tốt → code dễ hiểu, maintain.

```python
from typing import TypedDict, Annotated, List, Optional
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[List, add_messages]
    current_step: str
    iterations: int
    user_id: str
    retrieved_docs: Optional[List]
    final_answer: Optional[str]
```

---

## 2. Pydantic State (Validate)

```python
from pydantic import BaseModel, Field
from typing import List

class AgentState(BaseModel):
    messages: List = Field(default_factory=list)
    iterations: int = 0
    completed: bool = False
    
    class Config:
        arbitrary_types_allowed = True

# Use trong StateGraph
graph = StateGraph(AgentState)
```

**Lợi ích**: validate type tự động, default values, IDE autocomplete.

---

## 3. Multiple State Schemas

### 3.1 Input vs Output Schema

```python
class InputState(TypedDict):
    question: str

class OutputState(TypedDict):
    answer: str

class InternalState(TypedDict):
    question: str
    answer: str
    intermediate: List

graph = StateGraph(InternalState, input=InputState, output=OutputState)
```

User chỉ thấy `question` (input) và `answer` (output), không thấy `intermediate`.

### 3.2 Private State

```python
class PublicState(TypedDict):
    messages: list

class PrivateState(TypedDict):
    debug_info: list

graph = StateGraph(PublicState)

def node_with_private(state: PrivateState) -> PublicState:
    # Node có thể đọc private state
    return {"messages": [...]}
```

---

## 4. Reducers

### 4.1 Built-in Reducers

```python
from operator import add
from typing import Annotated

class State(TypedDict):
    # Replace (default)
    name: str
    
    # Append list
    items: Annotated[list, add]
    
    # Smart message append
    messages: Annotated[list, add_messages]
```

### 4.2 Custom Reducer

```python
def merge_dict(left: dict, right: dict) -> dict:
    return {**left, **right}

class State(TypedDict):
    config: Annotated[dict, merge_dict]

# Node return:
# {"config": {"new_key": "value"}}
# → merge với existing config
```

### 4.3 Conditional Reducer

```python
def keep_latest(left: int, right: int) -> int:
    return right if right > left else left

class State(TypedDict):
    max_score: Annotated[int, keep_latest]
```

---

## 5. Common State Patterns

### 5.1 Agent State

```python
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
```

### 5.2 RAG State

```python
class RAGState(TypedDict):
    question: str
    documents: List[Document]
    answer: str
    sources: List[str]
```

### 5.3 Workflow State

```python
class WorkflowState(TypedDict):
    input: str
    plan: List[str]
    current_step: int
    step_results: Annotated[List[dict], add]
    final_output: Optional[str]
```

### 5.4 Multi-Agent State

```python
class TeamState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str
    research_results: Optional[str]
    code_results: Optional[str]
    review_feedback: Optional[str]
```

---

## 6. Best Practices

✅ **Keep state minimal**: chỉ field cần share giữa nodes

✅ **Use Pydantic cho complex state**: validation tự động

✅ **Choose right reducer**: 
- `add` cho list (append)
- `add_messages` cho chat messages
- Default (replace) cho single values

✅ **Document state**: comment mỗi field

✅ **Versioning**: nếu thay đổi state schema, cần migrate persistent state

---

## 7. Demo: Complex State

```python
from typing import TypedDict, Annotated, List, Optional
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from operator import add
from pydantic import BaseModel
from datetime import datetime

class ResearchTask(BaseModel):
    topic: str
    sources_needed: int
    deadline: Optional[datetime] = None

class State(TypedDict):
    # Public input
    task: ResearchTask
    
    # Working state
    messages: Annotated[List, add_messages]
    search_queries: Annotated[List[str], add]
    found_sources: Annotated[List[dict], add]
    
    # Counters
    iteration: int
    api_calls: int
    
    # Output
    final_report: Optional[str]
    status: str  # "pending", "in_progress", "complete", "failed"

# Nodes
def plan_research(state: State) -> dict:
    task = state["task"]
    queries = [f"{task.topic} {q}" for q in ["overview", "details", "examples"]]
    return {
        "search_queries": queries,
        "status": "in_progress",
        "iteration": state.get("iteration", 0) + 1,
    }

def search(state: State) -> dict:
    # Search for each query
    found = []
    for q in state["search_queries"]:
        # Mock search
        found.append({"query": q, "results": ["doc1", "doc2"]})
    return {
        "found_sources": found,
        "api_calls": state.get("api_calls", 0) + len(state["search_queries"]),
    }

def synthesize(state: State) -> dict:
    sources_count = len(state["found_sources"])
    return {
        "final_report": f"Report on {state['task'].topic} based on {sources_count} sources",
        "status": "complete",
    }

# Build
graph = StateGraph(State)
graph.add_node("plan", plan_research)
graph.add_node("search", search)
graph.add_node("synthesize", synthesize)

graph.set_entry_point("plan")
graph.add_edge("plan", "search")
graph.add_edge("search", "synthesize")
graph.add_edge("synthesize", END)

app = graph.compile()

# Run
result = app.invoke({
    "task": ResearchTask(topic="LangGraph", sources_needed=5),
    "iteration": 0,
    "api_calls": 0,
})

print(f"Status: {result['status']}")
print(f"API calls: {result['api_calls']}")
print(f"Report: {result['final_report']}")
```

---

## 8. Checklist

- [ ] TypedDict vs Pydantic state
- [ ] Built-in reducers (add, add_messages)
- [ ] Custom reducer
- [ ] Input/Output state schema
- [ ] State patterns cho common use cases

➡️ **Tiếp theo**: [Control Flow](./03-Control-Flow.md)
