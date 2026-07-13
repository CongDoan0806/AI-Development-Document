# Control Flow - Branching & Routing

## 1. Conditional Edges

```python
from langgraph.graph import StateGraph, END

def router(state):
    if state["score"] > 0.8:
        return "high"
    elif state["score"] > 0.5:
        return "medium"
    return "low"

graph.add_conditional_edges(
    "evaluator",   # From node
    router,        # Routing function
    {
        "high": "approve",
        "medium": "review",
        "low": "reject",
    }
)
```

---

## 2. Multiple Conditional Branches

```python
def supervisor(state):
    last = state["messages"][-1]
    if "code" in last.content.lower():
        return "coder"
    elif "search" in last.content.lower():
        return "researcher"
    elif "done" in last.content.lower():
        return END
    return "supervisor"  # Loop back

graph.add_conditional_edges("supervisor", supervisor, {
    "coder": "coder",
    "researcher": "researcher",
    "supervisor": "supervisor",
    END: END,
})
```

---

## 3. Parallel Execution (Branches)

Nhiều nodes chạy song song:

```python
from langgraph.graph import StateGraph, END

class State(TypedDict):
    input: str
    result_a: str
    result_b: str
    result_c: str
    final: str

def branch_a(state):
    return {"result_a": f"A({state['input']})"}

def branch_b(state):
    return {"result_b": f"B({state['input']})"}

def branch_c(state):
    return {"result_c": f"C({state['input']})"}

def combine(state):
    return {"final": f"{state['result_a']} + {state['result_b']} + {state['result_c']}"}

graph = StateGraph(State)
graph.add_node("a", branch_a)
graph.add_node("b", branch_b)
graph.add_node("c", branch_c)
graph.add_node("combine", combine)

# Branches từ START → 3 nodes song song
graph.set_entry_point("a")  # Cần entry, sẽ split
# Cách đúng: dùng START
from langgraph.graph import START
graph.add_edge(START, "a")
graph.add_edge(START, "b")
graph.add_edge(START, "c")

# 3 nodes → combine
graph.add_edge("a", "combine")
graph.add_edge("b", "combine")
graph.add_edge("c", "combine")
graph.add_edge("combine", END)

app = graph.compile()

# a, b, c chạy SONG SONG
result = app.invoke({"input": "test"})
```

---

## 4. Map-Reduce Pattern

Process list items song song, sau đó combine:

```python
from langgraph.constants import Send
from typing import TypedDict, Annotated, List
from operator import add

class State(TypedDict):
    items: List[str]
    summaries: Annotated[List[str], add]
    final: str

class ItemState(TypedDict):
    item: str

def fan_out(state: State):
    """Tạo task cho mỗi item"""
    return [Send("process_item", {"item": item}) for item in state["items"]]

def process_item(state: ItemState) -> dict:
    """Process 1 item"""
    summary = f"Processed: {state['item']}"
    return {"summaries": [summary]}

def combine(state: State) -> dict:
    return {"final": "\n".join(state["summaries"])}

graph = StateGraph(State)
graph.add_node("process_item", process_item)
graph.add_node("combine", combine)

graph.add_conditional_edges(START, fan_out, ["process_item"])
graph.add_edge("process_item", "combine")
graph.add_edge("combine", END)

app = graph.compile()

result = app.invoke({
    "items": ["item1", "item2", "item3"],
    "summaries": [],
    "final": "",
})
```

`Send()` cho phép spawn dynamic số tasks (không cố định trước).

---

## 5. Loop Patterns

### 5.1 Self-Loop (Retry)

```python
def attempt(state):
    result = try_something()
    return {"result": result, "success": result is not None}

def check(state):
    if state["success"]:
        return END
    return "attempt"  # Loop back

graph.add_node("attempt", attempt)
graph.add_conditional_edges("attempt", check, {"attempt": "attempt", END: END})
```

### 5.2 Loop With Counter

```python
class State(TypedDict):
    counter: int
    done: bool

def increment(state):
    return {"counter": state["counter"] + 1}

def check(state):
    if state["counter"] >= 5 or state["done"]:
        return END
    return "increment"

graph.add_conditional_edges("increment", check, {
    "increment": "increment",
    END: END,
})
```

### 5.3 Recursion Limit

```python
# Để tránh infinite loop
config = {"recursion_limit": 25}
result = app.invoke({"counter": 0}, config=config)
```

---

## 6. Skip Steps

```python
def maybe_search(state):
    if state.get("cached_result"):
        return "use_cache"
    return "search"

graph.add_conditional_edges("decide", maybe_search, {
    "use_cache": END,  # Skip search
    "search": "search",
})
```

---

## 7. Early Exit

```python
def check_safety(state):
    if "dangerous" in state["input"].lower():
        return "reject"
    return "continue"

graph.add_conditional_edges("input_check", check_safety, {
    "reject": "reject_handler",  # Early exit
    "continue": "main_flow",
})
```

---

## 8. Demo: Adaptive RAG

```python
from typing import TypedDict, Annotated, List
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class RAGState(TypedDict):
    question: str
    classification: str  # "factual", "complex", "opinion"
    documents: List
    answer: str
    needs_web_search: bool

# Nodes
def classify_question(state):
    # LLM classify
    q = state["question"]
    if "?" in q and len(q.split()) < 10:
        return {"classification": "factual"}
    elif "tại sao" in q.lower() or "vì sao" in q.lower():
        return {"classification": "complex"}
    return {"classification": "opinion"}

def vector_search(state):
    # Quick vector search
    return {"documents": ["doc1", "doc2"], "needs_web_search": False}

def deep_research(state):
    # Multi-query, hybrid search, rerank
    return {"documents": ["thorough_doc1", "thorough_doc2", "thorough_doc3"]}

def web_search(state):
    # Tavily/Google
    return {"documents": state.get("documents", []) + ["web_result"]}

def generate_answer(state):
    return {"answer": f"Based on {len(state['documents'])} docs: ..."}

def route_after_classify(state):
    if state["classification"] == "factual":
        return "vector_search"
    elif state["classification"] == "complex":
        return "deep_research"
    return "vector_search"

def need_web(state):
    if state.get("needs_web_search") or len(state["documents"]) < 2:
        return "web_search"
    return "generate"

# Graph
graph = StateGraph(RAGState)
graph.add_node("classify", classify_question)
graph.add_node("vector_search", vector_search)
graph.add_node("deep_research", deep_research)
graph.add_node("web_search", web_search)
graph.add_node("generate", generate_answer)

graph.set_entry_point("classify")
graph.add_conditional_edges("classify", route_after_classify, {
    "vector_search": "vector_search",
    "deep_research": "deep_research",
})
graph.add_conditional_edges("vector_search", need_web, {
    "web_search": "web_search",
    "generate": "generate",
})
graph.add_edge("deep_research", "generate")
graph.add_edge("web_search", "generate")
graph.add_edge("generate", END)

app = graph.compile()

# Test
print(app.invoke({"question": "Thủ đô Việt Nam?"}))
print(app.invoke({"question": "Tại sao bầu trời màu xanh?"}))
```

---

## 9. Checklist

- [ ] Conditional edges
- [ ] Parallel branches với START
- [ ] Map-reduce với Send
- [ ] Self-loop và loop với counter
- [ ] Recursion limit
- [ ] Skip & early exit patterns

➡️ **Tiếp theo**: [Retry & Fallback](./04-Retry-Fallback.md)
