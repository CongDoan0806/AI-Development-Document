# Agent Memory - Bộ Nhớ Của Agent

## 1. Tại Sao Cần Memory?

Mỗi LLM call **không có memory** mặc định. Để chatbot/agent nhớ context:
- 📝 **Short-term**: lịch sử hội thoại hiện tại
- 💾 **Long-term**: facts về user qua nhiều session
- 🎯 **Working memory**: state trung gian khi solving

```
Without memory:
User: "Tên tôi là An"
AI: "Chào An!"
User: "Tôi tên gì?"
AI: "Tôi không biết." ❌

With memory:
User: "Tên tôi là An"  → save
User: "Tôi tên gì?"   → recall → "An"
AI: "Tên bạn là An." ✓
```

---

## 2. Các Loại Memory

| Loại | Mô tả | Use case |
|------|-------|----------|
| **Buffer** | Lưu nguyên history | Short conversations |
| **Window** | Lưu N message gần nhất | Limit context size |
| **Summary** | Tóm tắt history cũ | Long conversations |
| **Entity** | Lưu facts về entities | User profile |
| **Vector** | Embed memory, search semantic | Long-term recall |
| **Knowledge Graph** | Relations giữa entities | Complex reasoning |

---

## 3. Buffer Memory - Đơn Giản Nhất

### 3.1 Manual

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage

llm = ChatOpenAI(model="gpt-4o-mini")
history = []

def chat(message):
    history.append(HumanMessage(content=message))
    response = llm.invoke(history)
    history.append(response)
    return response.content

print(chat("Tên tôi là An"))
print(chat("Bạn tên gì?"))
print(chat("Tôi vừa nói tên tôi là gì?"))
```

### 3.2 ChatMessageHistory

```python
from langchain_core.chat_history import InMemoryChatMessageHistory

history = InMemoryChatMessageHistory()

# Add messages
history.add_user_message("Hello")
history.add_ai_message("Hi there!")
history.add_user_message("My name is An")

# Get all
print(history.messages)

# Clear
history.clear()
```

### 3.3 ConversationBufferMemory (Legacy)

API cũ hơn, vẫn được dùng nhiều trong code/tutorial cũ. Tự động save buffer string.

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory()
memory.chat_memory.add_user_message("Hi!")
memory.chat_memory.add_ai_message("Hello!")
memory.chat_memory.add_user_message("Tôi muốn biết về Pizza")
memory.chat_memory.add_ai_message("Bạn muốn biết gì?")

# Xem buffer (concat string)
print(memory.buffer)
# "Human: Hi!
#  AI: Hello!
#  Human: Tôi muốn biết về Pizza
#  AI: Bạn muốn biết gì?"

# Load variables (dict cho prompt template)
print(memory.load_memory_variables({}))
# {"history": "Human: Hi!\nAI: Hello!\n..."}
```

**Khuyến nghị**: dùng `InMemoryChatMessageHistory` + `RunnableWithMessageHistory` (LCEL mới) thay vì `ConversationBufferMemory` cho code mới.

---

## 3.5 ConversationChain (Legacy)

`ConversationChain` là chain có sẵn memory, dùng default prompt thân thiện.

```python
from langchain.chains import ConversationChain
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationBufferMemory

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

conversation = ConversationChain(
    llm=llm,
    memory=ConversationBufferMemory(),
    verbose=True,  # In ra prompt mỗi lần
)

# Default prompt template
print(conversation.prompt.template)
# "The following is a friendly conversation between a human and an AI...
#  Current conversation: {history}
#  Human: {input}
#  AI:"

# Use
conversation.invoke("Ai là người đầu tiên đặt chân lên mặt trăng?")
conversation.invoke("Ông ấy đến từ quốc gia nào?")  # Nhớ "ông ấy" = Neil Armstrong
```

**Output flow**:
```
> Entering new ConversationChain chain...
Prompt after formatting:
The following is a friendly conversation between a human and an AI...

Current conversation:
Human: Ai là người đầu tiên đặt chân lên mặt trăng?
AI: Neil Armstrong, ngày 20/7/1969...
```

**Vấn đề của `ConversationBufferMemory`**:
- ❌ History grows **endlessly** → tốn token mỗi lần gọi
- ❌ Đến lúc vượt context window → lỗi
- ❌ Trả tiền theo token, mỗi turn càng đắt

→ Cần các loại memory limit history (xem section 6 - Window, 7 - Summary).

### 3.6 ConversationBufferWindowMemory (Legacy)

Giữ N message gần nhất, drop cũ.

```python
from langchain.memory import ConversationBufferWindowMemory
from langchain.chains import ConversationChain

# k=1 → chỉ giữ 1 lượt exchange gần nhất
memory = ConversationBufferWindowMemory(k=1)

conversation = ConversationChain(
    llm=llm,
    memory=memory,
    verbose=True,
)

conversation.invoke("Ai chiến thắng ICC Cricket World Cup 1975?")
# AI: West Indies. Captain Clive Lloyd...

conversation.invoke("5 + 5")  
# AI: 10
# Lúc này memory chỉ còn turn "5+5" (k=1)

conversation.invoke("Captain của đội thắng là ai?")
# AI: "Tôi không biết" - vì đã quên context cricket
```

**Khi nào dùng**:
- ✅ Chat nhanh, không cần long-term memory
- ✅ Giới hạn token cost
- ❌ Khi user hay reference back nhiều turn trước

### 3.7 LLMChain + Memory (Legacy)

Attach memory vào `LLMChain` thường (không chỉ ConversationChain):

```python
from langchain.chains import LLMChain
from langchain.prompts import ChatPromptTemplate
from langchain.memory import ConversationBufferMemory

prompt = ChatPromptTemplate.from_template(
    "Lịch sử: {history}\n\nUser: {product}\nAI:"
)

memory = ConversationBufferMemory()

chain = LLMChain(
    llm=llm,
    prompt=prompt,
    memory=memory,
)

chain.invoke("gaming laptop")
chain.invoke("playstation")
# Memory tự inject {history} vào prompt
```

⚠️ **Lưu ý**: `LLMChain` là legacy. Code mới nên dùng LCEL:
```python
chain = prompt | llm | StrOutputParser()
chain_with_history = RunnableWithMessageHistory(chain, ...)
```

---

## 4. RunnableWithMessageHistory ⭐

Wrapper tự động manage history:

```python
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.chat_history import InMemoryChatMessageHistory

# Storage
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

# Chain với placeholder cho history
prompt = ChatPromptTemplate.from_messages([
    ("system", "Bạn là trợ lý AI thân thiện."),
    MessagesPlaceholder("history"),
    ("user", "{input}"),
])

chain = prompt | llm

# Wrap với history
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history",
)

# Use
config = {"configurable": {"session_id": "user_1"}}

print(chain_with_history.invoke({"input": "Tôi tên An"}, config=config).content)
print(chain_with_history.invoke({"input": "Tôi tên gì?"}, config=config).content)
```

### 4.1 Multiple Sessions

```python
config_a = {"configurable": {"session_id": "user_a"}}
config_b = {"configurable": {"session_id": "user_b"}}

chain_with_history.invoke({"input": "I'm Alice"}, config=config_a)
chain_with_history.invoke({"input": "I'm Bob"}, config=config_b)

# Histories tách biệt
print(chain_with_history.invoke({"input": "What's my name?"}, config=config_a).content)
# "Alice"
print(chain_with_history.invoke({"input": "What's my name?"}, config=config_b).content)
# "Bob"
```

---

## 5. Persistent History

### 5.1 SQLite

```python
from langchain_community.chat_message_histories import SQLChatMessageHistory

def get_session_history(session_id):
    return SQLChatMessageHistory(
        session_id=session_id,
        connection_string="sqlite:///chat_history.db",
    )
```

### 5.2 Redis

```python
from langchain_community.chat_message_histories import RedisChatMessageHistory

def get_session_history(session_id):
    return RedisChatMessageHistory(
        session_id=session_id,
        url="redis://localhost:6379",
    )
```

### 5.3 MongoDB

```python
from langchain_mongodb.chat_message_histories import MongoDBChatMessageHistory

def get_session_history(session_id):
    return MongoDBChatMessageHistory(
        session_id=session_id,
        connection_string="mongodb://localhost:27017",
        database_name="chats",
        collection_name="messages",
    )
```

### 5.4 Postgres

```python
from langchain_postgres import PostgresChatMessageHistory

def get_session_history(session_id):
    return PostgresChatMessageHistory(
        session_id=session_id,
        connection=conn,
        table_name="chat_history",
    )
```

---

## 6. Window Memory - Giới Hạn Tokens

Lưu N messages gần nhất (tránh prompt quá dài).

```python
from langchain_core.messages import trim_messages

def get_history_window(session_id, k=10):
    full = store[session_id].messages
    return full[-k:]

# Hoặc dùng trim_messages helper
def trim_history(history_messages):
    return trim_messages(
        history_messages,
        max_tokens=2000,
        strategy="last",
        token_counter=llm,
        include_system=True,
    )
```

### Trong Chain

```python
from langchain_core.runnables import RunnablePassthrough

chain = (
    RunnablePassthrough.assign(
        history=lambda x: trim_messages(
            x["history"],
            max_tokens=2000,
            strategy="last",
            token_counter=ChatOpenAI(model="gpt-4o-mini"),
        )
    )
    | prompt
    | llm
)
```

---

## 7. Summary Memory

Khi history quá dài → tóm tắt thành summary.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

summary_prompt = ChatPromptTemplate.from_template("""
Tóm tắt hội thoại sau, giữ thông tin quan trọng:

{history}

Summary:
""")

summarizer = summary_prompt | llm | StrOutputParser()

# Memory implementation
class SummaryMemory:
    def __init__(self, llm, max_messages=10):
        self.summary = ""
        self.recent_messages = []
        self.max_messages = max_messages
        self.llm = llm
    
    def add(self, message):
        self.recent_messages.append(message)
        
        # Nếu quá dài → summarize
        if len(self.recent_messages) > self.max_messages:
            old = self.recent_messages[:self.max_messages // 2]
            self.recent_messages = self.recent_messages[self.max_messages // 2:]
            
            old_text = "\n".join(f"{m.type}: {m.content}" for m in old)
            new_summary = summarizer.invoke({
                "history": f"Previous summary: {self.summary}\n\nNew messages:\n{old_text}"
            })
            self.summary = new_summary
    
    def get(self):
        return {
            "summary": self.summary,
            "messages": self.recent_messages,
        }
```

---

## 8. Entity Memory

Lưu facts về entities (person, place, ...).

```python
class EntityMemory:
    def __init__(self, llm):
        self.entities = {}  # name -> facts
        self.llm = llm
        self.extract_prompt = ChatPromptTemplate.from_template("""
        Trích xuất entities và facts từ message:
        
        Message: {message}
        
        Output JSON: {{"entity_name": "fact1; fact2", ...}}
        Nếu không có entity, trả về {{}}
        """)
    
    def update(self, message: str):
        result = self.llm.invoke(self.extract_prompt.format(message=message))
        import json
        try:
            new_facts = json.loads(result.content)
            for entity, facts in new_facts.items():
                if entity not in self.entities:
                    self.entities[entity] = facts
                else:
                    self.entities[entity] += "; " + facts
        except:
            pass
    
    def get_facts(self, entity: str) -> str:
        return self.entities.get(entity, "")
    
    def all_facts(self) -> str:
        return "\n".join(f"{k}: {v}" for k, v in self.entities.items())

mem = EntityMemory(llm)
mem.update("Tôi tên An, làm việc ở Hà Nội, công ty Acme")
mem.update("Vợ tôi tên Hà, làm bác sĩ")
print(mem.all_facts())
# An: làm việc ở Hà Nội; công ty Acme
# Hà: vợ của An; bác sĩ
```

---

## 9. Vector Memory (Long-Term)

Lưu memories vào vector DB, search semantically khi cần.

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_core.documents import Document
from datetime import datetime

embeddings = OpenAIEmbeddings()
memory_store = Chroma(
    embedding_function=embeddings,
    persist_directory="./memory_db",
    collection_name="agent_memory",
)

def save_memory(content: str, metadata: dict = None):
    meta = metadata or {}
    meta["timestamp"] = datetime.now().isoformat()
    memory_store.add_documents([Document(page_content=content, metadata=meta)])

def recall_memory(query: str, k: int = 3):
    return memory_store.similarity_search(query, k=k)

# Use
save_memory("User likes Italian food, especially carbonara", {"type": "preference"})
save_memory("User's birthday is March 15th", {"type": "fact"})
save_memory("User is vegetarian", {"type": "preference"})

# Recall
memories = recall_memory("user food preferences")
for m in memories:
    print(m.page_content)
```

### 9.1 Memory With Score & Decay

```python
from langchain.retrievers import TimeWeightedVectorStoreRetriever

retriever = TimeWeightedVectorStoreRetriever(
    vectorstore=memory_store,
    decay_rate=0.01,  # Forget rate
    k=5,
)

# Memory cũ → score thấp dần
```

---

## 10. LangGraph Memory (Recommended Cho Production)

LangGraph có `MemorySaver` và `BaseStore` built-in:

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent

memory = MemorySaver()

agent = create_react_agent(llm, tools, checkpointer=memory)

# Same thread_id = nhớ
config = {"configurable": {"thread_id": "1"}}

agent.invoke({"messages": [("user", "I'm Alice")]}, config)
agent.invoke({"messages": [("user", "What's my name?")]}, config)
# → "Alice"
```

### Persistent Checkpointer

```python
# SQLite
from langgraph.checkpoint.sqlite import SqliteSaver
memory = SqliteSaver.from_conn_string("checkpoints.db")

# Postgres
from langgraph.checkpoint.postgres import PostgresSaver
memory = PostgresSaver.from_conn_string("postgresql://...")
```

→ Chi tiết ở [LangGraph Persistence](../04-LangGraph/07-Persistence.md)

---

## 11. Memory Hierarchy

Production agent dùng nhiều layer:

```
┌────────────────────────────────┐
│  Working Memory                │  Current turn variables
│  (Volatile, in-memory)         │
├────────────────────────────────┤
│  Short-term Memory             │  Recent messages (10-20)
│  (Buffer/Window)               │
├────────────────────────────────┤
│  Summary Memory                │  Compressed history
│  (Auto-summarized)             │
├────────────────────────────────┤
│  Long-term Memory              │  Persistent facts
│  (Vector DB / Knowledge Graph) │
└────────────────────────────────┘
```

```python
class HierarchicalMemory:
    def __init__(self, llm, embeddings):
        self.short_term = []           # Last 10 messages
        self.summary = ""
        self.entities = EntityMemory(llm)
        self.vector_memory = Chroma(embedding_function=embeddings)
    
    def add_interaction(self, user_msg, ai_msg):
        # Short-term
        self.short_term.append(("user", user_msg))
        self.short_term.append(("assistant", ai_msg))
        
        # Trim if too long
        if len(self.short_term) > 20:
            # Move to summary
            old = self.short_term[:10]
            self.short_term = self.short_term[10:]
            self.summary = self._summarize(self.summary, old)
        
        # Extract entities
        self.entities.update(user_msg)
        
        # Save important to vector
        if self._is_important(user_msg):
            self.vector_memory.add_texts([user_msg])
    
    def get_context(self, current_query):
        return {
            "summary": self.summary,
            "recent": self.short_term,
            "entities": self.entities.all_facts(),
            "relevant_long_term": [
                d.page_content
                for d in self.vector_memory.similarity_search(current_query, k=3)
            ],
        }
```

---

## 12. Demo: Personal Assistant Với Memory

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_community.chat_message_histories import SQLChatMessageHistory

llm = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là personal assistant. Quy tắc:
- Nhớ thông tin user đã share (tên, sở thích, schedule)
- Use thông tin đó để personalize response
- Thân thiện, lịch sự"""),
    MessagesPlaceholder("history"),
    ("user", "{input}"),
])

chain = prompt | llm

def get_history(session_id):
    return SQLChatMessageHistory(
        session_id=session_id,
        connection_string="sqlite:///assistant.db",
    )

assistant = RunnableWithMessageHistory(
    chain,
    get_history,
    input_messages_key="input",
    history_messages_key="history",
)

# Simulate
config = {"configurable": {"session_id": "user_001"}}

print(assistant.invoke({"input": "Tôi tên An, sống ở Hà Nội"}, config).content)
print(assistant.invoke({"input": "Tôi thích cà phê đen, không sữa"}, config).content)

# Sau vài ngày, vẫn nhớ
print(assistant.invoke({"input": "Gợi ý cho tôi 1 quán cà phê"}, config).content)
# → Sẽ gợi ý ở Hà Nội, phục vụ cà phê đen
```

---

## 13. Best Practices

✅ **Tách history per user/session**

✅ **Persistent storage**: dùng SQLite/Postgres/Redis cho production

✅ **Trim/Summary để giới hạn token cost**

✅ **Hybrid: short-term buffer + long-term vector**

✅ **Privacy**: clear memory khi user yêu cầu, anonymize sensitive data

✅ **Backup memory regularly**

---

## 14. Bài Tập

### Bài 1: Persistent Chatbot
Build chatbot dùng SQLite, test với 100 messages qua nhiều session.

### Bài 2: Smart Summary
Implement auto-summarization khi history > 20 messages. Test conversation 50 turns.

### Bài 3: Vector Memory Assistant
Build assistant nhớ facts về user lâu dài (vector store + entity memory).

---

## 15. Checklist

- [ ] Buffer memory cơ bản
- [ ] RunnableWithMessageHistory
- [ ] Persistent: SQLite/Redis/Postgres
- [ ] Window memory với trim_messages
- [ ] Summary memory
- [ ] Entity memory
- [ ] Vector memory cho long-term
- [ ] LangGraph checkpointer

➡️ **Tiếp theo**: [Error Handling](./06-Error-Handling.md)
