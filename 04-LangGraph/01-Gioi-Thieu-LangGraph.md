# Giới Thiệu LangGraph

## 1. LangGraph Là Gì?

**LangGraph** = framework xây dựng **stateful, multi-actor** applications với LLM, dựa trên ý tưởng **directed graph**.

```
LangChain (Chain): linear, stateless
   prompt → llm → parser

LangGraph: stateful graph với nodes và edges
   ┌────┐    ┌────┐    ┌────┐
   │ A  │───→│ B  │───→│ C  │
   └────┘    └────┘    └─┬──┘
              ▲          │
              └──────────┘
              (loop back)
```

**Đặc điểm**:
- ✅ **Stateful**: lưu state qua các bước
- ✅ **Cyclic**: hỗ trợ loop (cần cho agent)
- ✅ **Branching**: conditional routing
- ✅ **Streaming**: stream từng node
- ✅ **Persistence**: checkpoint state
- ✅ **Human-in-the-loop**: pause/resume

---

## 2. Tại Sao Cần LangGraph?

### Vấn Đề Của Chain (LCEL)

```python
chain = prompt | llm | parser
# Linear, không có loop, không state ngoài input/output
```

Không thể làm:
- Loop "tool call → execute → tool call → ..." (cần cho agent)
- Conditional routing ("nếu user hỏi A → chain X, B → chain Y")
- Pause để chờ user input
- Share state giữa nhiều agents

### LangGraph Solve

```
                    ┌─────────┐
                    │  Start  │
                    └────┬────┘
                         ↓
                  ┌──────────┐
                  │ Classify │
                  └────┬─────┘
                       │
            ┌──────────┼──────────┐
            ↓          ↓          ↓
        ┌──────┐  ┌──────┐  ┌────────┐
        │ Tech │  │ Food │  │ Other  │
        │Agent │  │Agent │  │ Agent  │
        └──┬───┘  └──┬───┘  └───┬────┘
           └─────────┼──────────┘
                     ↓
                  ┌─────┐
                  │ END │
                  └─────┘
```

---

## 3. Khái Niệm Chính

### 3.1 State
Dữ liệu được pass qua các nodes:

```python
from typing import TypedDict, List

class State(TypedDict):
    messages: List[str]
    counter: int
    user_id: str
```

### 3.2 Node
Function nhận state, trả về state update:

```python
def my_node(state: State) -> dict:
    return {"counter": state["counter"] + 1}
```

### 3.3 Edge
Connection giữa nodes:

```python
# Static edge: A → B
graph.add_edge("A", "B")

# Conditional edge: A → B hoặc C tùy state
graph.add_conditional_edges("A", router_func, {"yes": "B", "no": "C"})
```

### 3.4 Graph
Tập hợp nodes và edges:

```python
from langgraph.graph import StateGraph

graph = StateGraph(State)
graph.add_node("A", node_a)
graph.add_node("B", node_b)
graph.add_edge("A", "B")
graph.set_entry_point("A")
graph.set_finish_point("B")

app = graph.compile()
```

---

## 4. Hello World LangGraph

```python
from typing import TypedDict
from langgraph.graph import StateGraph, END

# 1. Define state
class CounterState(TypedDict):
    count: int

# 2. Define nodes
def increment(state: CounterState) -> dict:
    print(f"Incrementing from {state['count']}")
    return {"count": state["count"] + 1}

def double(state: CounterState) -> dict:
    print(f"Doubling from {state['count']}")
    return {"count": state["count"] * 2}

# 3. Build graph
graph = StateGraph(CounterState)
graph.add_node("inc", increment)
graph.add_node("dbl", double)

graph.set_entry_point("inc")
graph.add_edge("inc", "dbl")
graph.add_edge("dbl", END)

# 4. Compile
app = graph.compile()

# 5. Run
result = app.invoke({"count": 5})
print(f"Final: {result}")
# Incrementing from 5
# Doubling from 6
# Final: {"count": 12}
```

---

## 5. State Update Semantics

### 5.1 Default - Replace

```python
class State(TypedDict):
    count: int

def node(state):
    return {"count": 10}  # Thay thế count = 10
```

### 5.2 Reducer - Append

Dùng `Annotated` để chỉ định reducer:

```python
from typing import Annotated
from operator import add

class State(TypedDict):
    messages: Annotated[list, add]  # Append vào list
    count: int  # Replace

def node(state):
    return {
        "messages": ["new message"],  # Append
        "count": 5,  # Replace
    }
```

### 5.3 Built-in Reducers

```python
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # Smart append for messages
```

`add_messages` smart:
- Append messages mới
- Update existing message bằng ID
- Handle ToolMessage match với AIMessage

---

## 6. Conditional Edges

```python
class State(TypedDict):
    value: int

def node_a(state):
    return {"value": state["value"] * 2}

def node_b(state):
    return {"value": state["value"] + 100}

def node_c(state):
    return {"value": state["value"] - 1}

def router(state):
    if state["value"] > 100:
        return "big"
    elif state["value"] > 10:
        return "medium"
    else:
        return "small"

graph = StateGraph(State)
graph.add_node("a", node_a)
graph.add_node("big_handler", node_b)
graph.add_node("small_handler", node_c)

graph.set_entry_point("a")
graph.add_conditional_edges(
    "a",
    router,
    {
        "big": "big_handler",
        "medium": END,
        "small": "small_handler",
    }
)
graph.add_edge("big_handler", END)
graph.add_edge("small_handler", END)

app = graph.compile()

print(app.invoke({"value": 1}))   # 1*2=2, small → -1 = 1
print(app.invoke({"value": 10}))  # 10*2=20, medium → END = 20
print(app.invoke({"value": 100})) # 100*2=200, big → +100 = 300
```

---

## 7. Visualization

```python
# ASCII art
print(app.get_graph().draw_ascii())

# Mermaid (cho docs)
print(app.get_graph().draw_mermaid())

# PNG file
app.get_graph().draw_mermaid_png(output_file_path="graph.png")
```

---

## 8. Mini Agent Example

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage

# Tools
@tool
def get_weather(city: str) -> str:
    """Get weather"""
    return f"{city}: 28°C"

@tool
def calculate(expr: str) -> str:
    """Calculate"""
    return str(eval(expr))

tools = [get_weather, calculate]
tool_map = {t.name: t for t in tools}

llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

# State
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

# Nodes
def call_llm(state):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def call_tools(state):
    last_msg = state["messages"][-1]
    tool_results = []
    for tc in last_msg.tool_calls:
        result = tool_map[tc["name"]].invoke(tc["args"])
        tool_results.append(ToolMessage(
            content=str(result),
            tool_call_id=tc["id"]
        ))
    return {"messages": tool_results}

def should_continue(state):
    last = state["messages"][-1]
    if last.tool_calls:
        return "tools"
    return END

# Graph
graph = StateGraph(AgentState)
graph.add_node("llm", call_llm)
graph.add_node("tools", call_tools)

graph.set_entry_point("llm")
graph.add_conditional_edges("llm", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "llm")

app = graph.compile()

# Run
result = app.invoke({
    "messages": [HumanMessage(content="Weather Hà Nội và 5 * 7 = ?")]
})

for m in result["messages"]:
    m.pretty_print()
```

---

## 9. Prebuilt: create_react_agent

Đỡ phải code agent loop từ đầu:

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model=llm,
    tools=tools,
    state_modifier="Bạn là trợ lý AI.",  # Optional system message
)

result = agent.invoke({"messages": [("user", "Hello")]})
```

---

## 10. Streaming

### 10.1 Stream Updates

```python
for chunk in app.stream({"messages": [HumanMessage("Hello")]}):
    print(chunk)
# {"llm": {"messages": [AIMessage(...)]}}
# {"tools": {"messages": [ToolMessage(...)]}}
# ...
```

### 10.2 Stream Values

```python
for chunk in app.stream(
    {"messages": [HumanMessage("Hello")]},
    stream_mode="values",  # Full state mỗi step
):
    print(chunk)
```

### 10.3 Stream Tokens

```python
async for event in app.astream_events(
    {"messages": [...]},
    version="v2",
):
    if event["event"] == "on_chat_model_stream":
        chunk = event["data"]["chunk"]
        print(chunk.content, end="")
```

---

## 11. State Update Trong Node

### 11.1 Partial Update

```python
def node(state):
    return {"counter": 10}  # Chỉ update counter, các field khác giữ nguyên
```

### 11.2 Read-Modify-Write

```python
def node(state):
    current = state["counter"]
    return {"counter": current + 1}
```

### 11.3 Trả Về Nhiều Field

```python
def node(state):
    return {
        "messages": [new_msg],
        "iterations": state["iterations"] + 1,
        "last_action": "search",
    }
```

---

## 12. Subgraph

Embed graph trong graph:

```python
# Subgraph: research
research_graph = StateGraph(...)
# ...
research_subgraph = research_graph.compile()

# Main graph dùng research_subgraph như 1 node
main_graph = StateGraph(...)
main_graph.add_node("research", research_subgraph)
```

→ Sẽ học chi tiết ở [Multi-Agent](./06-Multi-Agent-System.md)

---

## 13. Cài Đặt

```bash
pip install langgraph

# Optional
pip install langgraph-checkpoint-sqlite  # Persistent state
pip install langgraph-checkpoint-postgres
```

---

## 14. Roadmap Phase 4

1. ✅ Giới thiệu (file này)
2. ⏭️ [State Machine Design](./02-State-Machine-Design.md)
3. ⏭️ [Control Flow](./03-Control-Flow.md)
4. ⏭️ [Retry & Fallback](./04-Retry-Fallback.md)
5. ⏭️ [Human-in-the-Loop](./05-Human-in-the-Loop.md)
6. ⏭️ [Multi-Agent System](./06-Multi-Agent-System.md)
7. ⏭️ [Persistence](./07-Persistence.md)

---

## 15. Bài Tập

### Bài 1: Simple Counter
Build graph với 3 nodes: increment, double, halve. Conditional routing dựa trên giá trị.

### Bài 2: Mini Chatbot
Build chatbot graph với:
- Node: receive_input
- Node: classify (chat/question/exit)
- Node: respond
- Loop until "exit"

### Bài 3: Tool Agent From Scratch
Implement agent loop với LangGraph từ đầu (không dùng create_react_agent).

---

## 16. Checklist

- [ ] Hiểu State, Node, Edge
- [ ] Build graph cơ bản
- [ ] Conditional edges
- [ ] Annotated reducer
- [ ] add_messages cho agent
- [ ] Streaming
- [ ] Visualize graph
- [ ] create_react_agent

➡️ **Tiếp theo**: [State Machine Design](./02-State-Machine-Design.md)
