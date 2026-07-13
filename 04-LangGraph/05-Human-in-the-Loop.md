# Human-in-the-Loop (HITL)

## 1. Tại Sao Cần HITL?

Agent có thể fail, hallucinate, hoặc thực hiện action không mong muốn (send email sai, delete data). HITL = **pause để chờ human approve** trước critical action.

**Use cases**:
- 💸 Approve transactions
- 📧 Confirm email before sending
- 🗑️ Confirm delete
- ✏️ Edit AI output trước khi continue
- 🔄 Re-direct agent

---

## 2. Interrupt - Pause Trước Node

### 2.1 Static Interrupt

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, END

memory = MemorySaver()

graph = StateGraph(State)
graph.add_node("draft_email", draft_email)
graph.add_node("send_email", send_email)
graph.add_edge("draft_email", "send_email")

# Pause TRƯỚC khi vào send_email
app = graph.compile(
    checkpointer=memory,
    interrupt_before=["send_email"],
)

# Run
config = {"configurable": {"thread_id": "1"}}
result = app.invoke({"input": "..."}, config=config)

# Graph dừng ở send_email
# Inspect state
state = app.get_state(config)
print(state.values)  # Draft email
print(state.next)    # ["send_email"]
```

### 2.2 Approve và Continue

```python
# User review state, sau đó tiếp tục
# Nếu OK:
result = app.invoke(None, config=config)  # Continue

# Nếu muốn modify trước:
app.update_state(config, {"email_body": "Updated body"})
result = app.invoke(None, config=config)
```

---

## 3. Interrupt Cụ Thể Bằng interrupt()

```python
from langgraph.types import interrupt, Command

def review_node(state):
    # Pause và hiện info cho user
    user_response = interrupt({
        "question": "Approve email?",
        "draft": state["email_draft"],
    })
    
    if user_response["action"] == "approve":
        return {"approved": True}
    elif user_response["action"] == "edit":
        return {"email_draft": user_response["new_text"]}
    else:
        return {"approved": False}

graph.add_node("review", review_node)
```

Khi gặp `interrupt()`, graph pause. Resume bằng:

```python
from langgraph.types import Command

# Resume với user input
app.invoke(
    Command(resume={"action": "approve"}),
    config=config,
)
```

---

## 4. Edit State

User có thể edit state trước khi continue:

```python
# 1. Get current state
state = app.get_state(config)

# 2. Edit
new_messages = state.values["messages"][:-1]  # Remove last
new_messages.append(HumanMessage(content="Corrected version"))

# 3. Update
app.update_state(
    config,
    {"messages": new_messages},
    as_node="user_edit"  # Pretend user_edit node was last
)

# 4. Continue
result = app.invoke(None, config=config)
```

---

## 5. Time Travel

Quay lại state cũ và rẽ nhánh khác:

```python
# Get all history
history = list(app.get_state_history(config))

# Quay về state 3 bước trước
old_config = history[3].config

# Continue từ điểm đó (có thể với input mới)
result = app.invoke({"new_input": "..."}, config=old_config)
```

---

## 6. Demo: Approval Workflow

```python
from typing import TypedDict, Annotated, List, Optional
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

class State(TypedDict):
    request: str
    amount: float
    recipient: str
    draft: str
    approved: Optional[bool]
    final: str

def parse_request(state):
    # Parse "Send 5000 to John"
    parts = state["request"].split()
    return {
        "amount": float(parts[1]),
        "recipient": parts[-1],
    }

def draft_transaction(state):
    draft = f"Transaction: {state['amount']} → {state['recipient']}"
    return {"draft": draft}

def execute_transaction(state):
    # Thực sự thực hiện
    return {"final": f"✅ Sent {state['amount']} to {state['recipient']}"}

def cancel(state):
    return {"final": "❌ Cancelled by user"}

def route(state):
    if state["approved"]:
        return "execute"
    return "cancel"

memory = MemorySaver()

graph = StateGraph(State)
graph.add_node("parse", parse_request)
graph.add_node("draft", draft_transaction)
graph.add_node("execute", execute_transaction)
graph.add_node("cancel", cancel)

graph.set_entry_point("parse")
graph.add_edge("parse", "draft")
# Pause trước khi quyết định execute/cancel
graph.add_conditional_edges("draft", route, {
    "execute": "execute",
    "cancel": "cancel",
})
graph.add_edge("execute", END)
graph.add_edge("cancel", END)

# Compile với interrupt
app = graph.compile(
    checkpointer=memory,
    interrupt_before=["execute", "cancel"],  # Wait before either
)

# === Demo flow ===
config = {"configurable": {"thread_id": "tx_123"}}

# Step 1: Start, sẽ pause
result = app.invoke({"request": "Send 5000 to John"}, config=config)
print("Draft:", app.get_state(config).values["draft"])

# Step 2: User review (in real app: show UI)
user_approves = input("Approve? (y/n): ")

# Step 3: Update state với decision
app.update_state(config, {"approved": user_approves == "y"})

# Step 4: Continue
result = app.invoke(None, config=config)
print(result["final"])
```

---

## 7. Streaming Với HITL

```python
# Stream để show progress tới user
for event in app.stream({"request": "..."}, config=config):
    print(event)
    # Khi gặp interrupt, stream dừng

# Sau khi user approve
for event in app.stream(None, config=config):
    print(event)
```

---

## 8. Approval Patterns

### 8.1 Approve Each Tool Call

```python
def safe_tool_executor(state):
    last_msg = state["messages"][-1]
    
    for tc in last_msg.tool_calls:
        # Check if dangerous
        if tc["name"] in DANGEROUS_TOOLS:
            user_input = interrupt({
                "tool": tc["name"],
                "args": tc["args"],
                "prompt": "Approve this tool call?"
            })
            if not user_input["approved"]:
                # Skip this tool
                continue
        
        # Execute
        result = execute_tool(tc)
        ...
```

### 8.2 Approve Plan Trước Khi Execute

```python
def planner(state):
    plan = generate_plan(state["task"])
    user_input = interrupt({
        "plan": plan,
        "prompt": "Review plan. Approve, edit, or reject?"
    })
    
    if user_input["action"] == "reject":
        return {"status": "rejected"}
    elif user_input["action"] == "edit":
        return {"plan": user_input["edited_plan"]}
    
    return {"plan": plan}
```

### 8.3 Iterative Refinement

```python
def write_essay(state):
    essay = state.get("essay", "")
    
    while True:
        if not essay:
            essay = llm.invoke(f"Write essay on {state['topic']}").content
        
        feedback = interrupt({
            "essay": essay,
            "prompt": "Feedback? (or 'done')"
        })
        
        if feedback == "done":
            return {"final_essay": essay}
        
        essay = llm.invoke(f"Revise:\n{essay}\n\nFeedback: {feedback}").content
```

---

## 9. Best Practices

✅ **Pause cho destructive actions**: delete, send, transfer

✅ **Show context**: hiển thị đủ info cho user quyết định

✅ **Allow edit**: không chỉ approve/reject, cho phép modify

✅ **Timeout**: nếu user không response sau X phút → cancel

✅ **Audit log**: lưu lại quyết định của user

---

## 10. Checklist

- [ ] `interrupt_before` cho static interrupt
- [ ] `interrupt()` function cho dynamic
- [ ] `update_state` để edit
- [ ] `Command(resume=...)` để continue
- [ ] Time travel với `get_state_history`
- [ ] Approval workflows

➡️ **Tiếp theo**: [Multi-Agent System](./06-Multi-Agent-System.md)
