# Persistence - Lưu State Của Graph

## 1. Tại Sao Cần Persistence?

- 💬 **Chatbot**: nhớ conversation qua nhiều session
- 🔄 **Resume**: tiếp tục task dài (giờ, ngày)
- ⏸️ **Human-in-the-loop**: pause chờ approval
- 🔍 **Debug**: replay từ state cụ thể
- 🔀 **Time travel**: quay lại state cũ

LangGraph dùng **Checkpointer** để persist state.

---

## 2. Checkpointer Types

| Checkpointer | Use case |
|--------------|----------|
| `MemorySaver` | Dev/testing |
| `SqliteSaver` | Local app, small scale |
| `PostgresSaver` | Production |
| `RedisSaver` | High performance |

---

## 3. MemorySaver (Quick Start)

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent

memory = MemorySaver()

agent = create_react_agent(llm, tools, checkpointer=memory)

# Use với thread_id
config = {"configurable": {"thread_id": "user_123"}}

# Conversation 1
agent.invoke({"messages": [("user", "Tôi tên An")]}, config)

# Conversation 2 (nhớ tên)
result = agent.invoke({"messages": [("user", "Tên tôi là gì?")]}, config)
# → "An"
```

⚠️ **MemorySaver lưu trong RAM** → restart app sẽ mất.

---

## 4. SqliteSaver - Persistent Local

```python
# pip install langgraph-checkpoint-sqlite
from langgraph.checkpoint.sqlite import SqliteSaver

# Tạo checkpointer
checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

# Hoặc context manager
with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    agent = create_react_agent(llm, tools, checkpointer=checkpointer)
    # ...
```

### Async Version

```python
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

async with AsyncSqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    agent = create_react_agent(llm, tools, checkpointer=checkpointer)
    result = await agent.ainvoke({...}, config)
```

---

## 5. PostgresSaver - Production

```python
# pip install langgraph-checkpoint-postgres psycopg2-binary
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:pass@localhost:5432/checkpoints"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    # Setup schema (run once)
    checkpointer.setup()
    
    agent = create_react_agent(llm, tools, checkpointer=checkpointer)
    result = agent.invoke({...}, config)
```

### Connection Pool

```python
from psycopg_pool import ConnectionPool

pool = ConnectionPool(DB_URI, max_size=20)

with pool.connection() as conn:
    checkpointer = PostgresSaver(conn)
    # Use checkpointer
```

---

## 6. RedisSaver - High Performance

```python
# pip install langgraph-checkpoint-redis
from langgraph.checkpoint.redis import RedisSaver

with RedisSaver.from_conn_string("redis://localhost:6379") as checkpointer:
    checkpointer.setup()
    agent = create_react_agent(llm, tools, checkpointer=checkpointer)
```

---

## 7. Thread & Checkpoint

### Concepts

- **Thread**: 1 conversation/session, identified by `thread_id`
- **Checkpoint**: snapshot of state tại 1 thời điểm
- Mỗi thread có nhiều checkpoints (mỗi step = 1 checkpoint)

```python
config = {
    "configurable": {
        "thread_id": "user_123_session_1",
        # Optional:
        "checkpoint_id": "specific-checkpoint-uuid",
    }
}
```

---

## 8. Inspect State

```python
# Current state
state = app.get_state(config)
print(state.values)        # State data
print(state.next)          # Next nodes to run
print(state.config)        # Checkpoint config
print(state.metadata)      # Step number, source, ...
print(state.created_at)
print(state.parent_config) # Previous checkpoint
```

---

## 9. State History

```python
# All checkpoints in thread
history = list(app.get_state_history(config))

for h in history:
    print(f"Step {h.metadata['step']}: {h.values}")
```

---

## 10. Time Travel

Quay lại state cũ và rẽ nhánh:

```python
# Get history
history = list(app.get_state_history(config))

# Quay về state thứ 3
target_config = history[3].config

# Continue từ điểm đó
# Có thể với input mới
new_config = {**target_config}
result = app.invoke({"new_input": "..."}, config=new_config)
```

---

## 11. Update State

```python
# Modify state hiện tại
app.update_state(
    config,
    {"messages": [HumanMessage("Updated message")]},
    as_node="user_edit",  # Tag là node nào tạo update này
)

# Continue
result = app.invoke(None, config=config)
```

---

## 12. Cross-Thread Memory (Store)

Checkpointer lưu **per-thread**. Để share data giữa threads (user profile), dùng **Store**.

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# Put
store.put(
    namespace=("users", "user_123"),
    key="profile",
    value={"name": "An", "preferences": ["coffee"]},
)

# Get
profile = store.get(("users", "user_123"), "profile")
print(profile.value)
```

### Postgres Store

```python
from langgraph.store.postgres import PostgresStore

with PostgresStore.from_conn_string(DB_URI) as store:
    store.setup()
    # Use store
```

### Tích Hợp Vào Graph

```python
def my_node(state, *, store):
    user_id = state["user_id"]
    profile = store.get(("users", user_id), "profile")
    # Use profile
    
    store.put(("users", user_id), "last_interaction", {"timestamp": "..."})
    return {...}

agent = create_react_agent(
    llm,
    tools,
    checkpointer=checkpointer,
    store=store,
)
```

---

## 13. Search Cross-Thread Memory

```python
# Semantic search trong store
results = store.search(
    namespace_prefix=("users",),
    query="users who like coffee",
    limit=10,
)
```

(Cần store hỗ trợ embeddings: `PostgresStore` với pgvector)

---

## 14. Demo: Multi-Session Chatbot

```python
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.store.memory import InMemoryStore
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

llm = ChatOpenAI(model="gpt-4o-mini")

# Cross-thread store
store = InMemoryStore()

@tool
def save_preference(user_id: str, preference: str) -> str:
    """Save user preference"""
    existing = store.get(("users", user_id), "prefs")
    prefs = existing.value if existing else {"preferences": []}
    prefs["preferences"].append(preference)
    store.put(("users", user_id), "prefs", prefs)
    return f"Saved: {preference}"

@tool
def get_preferences(user_id: str) -> str:
    """Get user preferences"""
    result = store.get(("users", user_id), "prefs")
    if result:
        return str(result.value["preferences"])
    return "No preferences"

# Persistent checkpointer
with SqliteSaver.from_conn_string("chatbot.db") as checkpointer:
    agent = create_react_agent(
        llm,
        [save_preference, get_preferences],
        checkpointer=checkpointer,
    )
    
    # User 1 - Session 1
    config1 = {"configurable": {"thread_id": "user1_session1"}}
    print(agent.invoke({
        "messages": [("user", "Tôi user_id=user1. Tôi thích cà phê đen")]
    }, config1))
    
    # User 1 - Session 2 (mới, thread khác nhưng cùng user)
    config2 = {"configurable": {"thread_id": "user1_session2"}}
    print(agent.invoke({
        "messages": [("user", "Tôi user_id=user1. Sở thích của tôi là gì?")]
    }, config2))
    # → Vẫn nhớ "cà phê đen" qua store
```

---

## 15. Migration & Schema Changes

Khi thay đổi state schema:

```python
# Cách 1: New table/namespace
checkpointer_v2 = PostgresSaver.from_conn_string(DB_URI + "_v2")

# Cách 2: Migration script
def migrate(old_state):
    return {
        "messages": old_state["messages"],
        "new_field": "default",  # Add
    }
```

---

## 16. Backup & Restore

```python
# Backup
import shutil
shutil.copy("chatbot.db", "backup_2024-12-01.db")

# Postgres: pg_dump
# Redis: BGSAVE
```

---

## 17. Best Practices

✅ **Dev**: MemorySaver

✅ **Local app**: SqliteSaver

✅ **Production**: PostgresSaver hoặc RedisSaver

✅ **Cleanup old threads**: implement TTL

✅ **Use Store cho user data persistent across threads**

✅ **Backup regularly**

---

## 18. Checklist

- [ ] MemorySaver
- [ ] SqliteSaver, PostgresSaver, RedisSaver
- [ ] Thread vs Checkpoint
- [ ] get_state, get_state_history
- [ ] update_state
- [ ] Time travel
- [ ] Cross-thread Store
- [ ] Async checkpointer

🎉 **Hoàn thành Phase 4 - LangGraph!**

➡️ **Tiếp theo**: [Phase 5 - MCP](../05-MCP/01-Gioi-Thieu-MCP.md)
