# Project 5: SQL Agent (Natural Language To SQL)

## 1. Mục Tiêu

Build agent:
- User hỏi câu hỏi bằng tiếng Việt
- Agent generate SQL
- Execute SQL safely (read-only)
- Format result thành natural language
- Show chart nếu phù hợp

**Example**:
```
User: "Top 5 sản phẩm bán chạy nhất tháng trước"
Agent: → Generates SQL → Executes → "Sản phẩm bán chạy nhất tháng 11 là..."
```

---

## 2. Setup

```bash
pip install langchain langchain-openai sqlalchemy psycopg2-binary
pip install pandas matplotlib  # cho charts
```

```env
DATABASE_URL=postgresql://user:pass@localhost/db
```

---

## 3. SQL Database Setup

```python
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri(
    "postgresql://user:pass@localhost/sales_db",
    include_tables=["products", "orders", "customers"],  # Whitelist
    sample_rows_in_table_info=3,  # Include sample rows
)

# Inspect
print(db.dialect)
print(db.get_usable_table_names())
print(db.get_table_info(["products"]))
```

---

## 4. Basic SQL Agent

```python
from langchain_community.agent_toolkits import create_sql_agent
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o", temperature=0)

agent = create_sql_agent(
    llm=llm,
    db=db,
    agent_type="openai-tools",
    verbose=True,
)

result = agent.invoke({"input": "Có bao nhiêu khách hàng?"})
print(result["output"])
```

---

## 5. Custom SQL Agent với LangGraph

```python
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def list_tables() -> list:
    """List all tables in database"""
    return db.get_usable_table_names()

@tool
def get_schema(table_names: str) -> str:
    """Get schema for tables. Pass comma-separated names."""
    tables = [t.strip() for t in table_names.split(",")]
    return db.get_table_info(tables)

@tool
def execute_sql(query: str) -> str:
    """Execute SQL SELECT query.
    
    Args:
        query: SQL SELECT statement
    
    Returns:
        Query results as string
    """
    # Safety check
    query_lower = query.strip().lower()
    if not query_lower.startswith("select"):
        return "Error: Only SELECT queries allowed"
    
    forbidden = ["insert", "update", "delete", "drop", "alter", "truncate", "create"]
    if any(word in query_lower for word in forbidden):
        return "Error: Modification queries not allowed"
    
    # Auto-add LIMIT
    if "limit" not in query_lower:
        query += " LIMIT 100"
    
    try:
        return db.run(query)
    except Exception as e:
        return f"SQL Error: {e}"

@tool
def query_checker(query: str) -> str:
    """Check SQL query for common mistakes before executing"""
    check_prompt = f"""Check SQL query for issues:
{query}

Common issues:
- Using wrong table/column names
- Syntax errors
- Logic errors
- Missing JOINs

If OK, reply: VALID
If has issues, reply: ISSUES: <explanation>"""
    
    return llm.invoke(check_prompt).content

# Create agent
agent = create_react_agent(
    model=llm,
    tools=[list_tables, get_schema, execute_sql, query_checker],
    state_modifier=f"""Bạn là SQL Expert. Trả lời câu hỏi bằng cách query database.

DATABASE: {db.dialect}

WORKFLOW:
1. List tables nếu chưa biết schema
2. Get schema của các tables cần dùng
3. Viết SQL query
4. Check query với query_checker
5. Execute query
6. Format kết quả thành natural language tiếng Việt

QUY TẮC:
- CHỈ SELECT queries
- LIMIT 100 mặc định
- Dùng JOIN khi cần
- Aggregate (COUNT, SUM, AVG) cho summary"""
)
```

---

## 6. Use

```python
questions = [
    "Có tổng cộng bao nhiêu khách hàng?",
    "Top 5 sản phẩm doanh thu cao nhất tháng 11/2024",
    "Khách hàng nào mua nhiều nhất?",
    "Sản phẩm nào chưa bán được lần nào?",
]

for q in questions:
    print(f"\n❓ {q}")
    result = agent.invoke({"messages": [("user", q)]})
    print(f"💬 {result['messages'][-1].content}")
```

---

## 7. SQL Few-Shot Examples

Cải thiện accuracy với examples:

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate

examples = [
    {
        "input": "Có bao nhiêu khách hàng?",
        "query": "SELECT COUNT(*) AS total FROM customers"
    },
    {
        "input": "Tổng doanh thu năm 2024",
        "query": "SELECT SUM(total) AS revenue FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2024"
    },
    {
        "input": "Top 5 sản phẩm bán chạy",
        "query": """
        SELECT p.name, SUM(oi.quantity) AS total_sold
        FROM products p
        JOIN order_items oi ON p.id = oi.product_id
        GROUP BY p.name
        ORDER BY total_sold DESC
        LIMIT 5
        """
    },
]

example_prompt = ChatPromptTemplate.from_messages([
    ("user", "{input}"),
    ("assistant", "SQL: {query}"),
])

few_shot = FewShotChatMessagePromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
)

system = ChatPromptTemplate.from_messages([
    ("system", "You are SQL expert. Generate SQL."),
    few_shot,
    ("user", "{question}"),
])
```

---

## 8. Schema-Aware Retrieval

Database lớn có hàng trăm tables → không đưa hết schema vào prompt. Dùng RAG cho schema:

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_core.documents import Document

# Index schemas
schemas = []
for table in db.get_usable_table_names():
    info = db.get_table_info([table])
    schemas.append(Document(
        page_content=info,
        metadata={"table_name": table},
    ))

schema_store = Chroma.from_documents(
    schemas,
    OpenAIEmbeddings(),
    collection_name="schemas",
)

@tool
def find_relevant_tables(question: str) -> str:
    """Find tables related to question"""
    docs = schema_store.similarity_search(question, k=5)
    return "\n\n".join(d.page_content for d in docs)
```

---

## 9. Visualization

```python
import pandas as pd
import matplotlib.pyplot as plt
import io
import base64

@tool
def execute_and_chart(query: str, chart_type: str = "bar") -> str:
    """Execute query và tạo chart. chart_type: bar, line, pie."""
    try:
        df = pd.read_sql(query, db._engine)
        
        # Auto-detect columns
        if len(df.columns) < 2:
            return df.to_string()
        
        # Create chart
        fig, ax = plt.subplots(figsize=(10, 6))
        
        if chart_type == "bar":
            df.plot.bar(x=df.columns[0], y=df.columns[1], ax=ax)
        elif chart_type == "line":
            df.plot.line(x=df.columns[0], y=df.columns[1], ax=ax)
        elif chart_type == "pie":
            df.set_index(df.columns[0])[df.columns[1]].plot.pie(ax=ax)
        
        # Save
        buf = io.BytesIO()
        plt.savefig(buf, format="png", bbox_inches="tight")
        buf.seek(0)
        img_b64 = base64.b64encode(buf.read()).decode()
        
        return f"![chart](data:image/png;base64,{img_b64})\n\n{df.to_markdown()}"
    except Exception as e:
        return f"Error: {e}"
```

---

## 10. Streamlit UI

```python
import streamlit as st
import pandas as pd

st.title("💾 SQL Assistant")

# Show schema
with st.sidebar:
    st.subheader("📋 Database Schema")
    for table in db.get_usable_table_names():
        with st.expander(table):
            st.code(db.get_table_info([table]))

# Chat history
if "messages" not in st.session_state:
    st.session_state.messages = []

for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        if "df" in msg:
            st.dataframe(msg["df"])
        if "chart" in msg:
            st.image(msg["chart"])
        st.markdown(msg["content"])

# Input
if question := st.chat_input("Ask in Vietnamese..."):
    st.session_state.messages.append({"role": "user", "content": question})
    with st.chat_message("user"):
        st.markdown(question)
    
    with st.chat_message("assistant"):
        with st.spinner("Querying database..."):
            result = agent.invoke({"messages": [("user", question)]})
        
        answer = result["messages"][-1].content
        st.markdown(answer)
        
        # Try to extract data
        # ... (parse last execute_sql result)
    
    st.session_state.messages.append({"role": "assistant", "content": answer})
```

---

## 11. Security Considerations

### 11.1 Read-Only Database User

```sql
-- Tạo user chỉ có SELECT
CREATE USER readonly_user WITH PASSWORD 'pass';
GRANT CONNECT ON DATABASE mydb TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
```

```python
db = SQLDatabase.from_uri("postgresql://readonly_user:pass@localhost/mydb")
```

### 11.2 Query Timeout

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://...",
    connect_args={"options": "-c statement_timeout=30000"},  # 30s
)
```

### 11.3 Data Privacy

```python
# Mask PII columns
@tool
def execute_sql_safe(query: str) -> str:
    """Execute với PII masking"""
    df = pd.read_sql(query, engine)
    
    # Mask sensitive
    if "email" in df.columns:
        df["email"] = df["email"].apply(lambda x: x[:3] + "***@***" if x else x)
    if "phone" in df.columns:
        df["phone"] = df["phone"].apply(lambda x: x[:3] + "****" if x else x)
    
    return df.to_string()
```

---

## 12. Caching Common Queries

```python
from functools import lru_cache
import hashlib

@lru_cache(maxsize=100)
def cached_query(query_hash):
    # Use hash for cache key
    pass

@tool
def execute_sql_cached(query: str) -> str:
    """Execute SQL với cache"""
    query_hash = hashlib.md5(query.encode()).hexdigest()
    cached = cache.get(query_hash)
    if cached:
        return cached
    
    result = db.run(query)
    cache.set(query_hash, result, ttl=300)  # 5 min
    return result
```

---

## 13. Multi-Turn With Memory

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
agent = create_react_agent(llm, tools, checkpointer=memory)

config = {"configurable": {"thread_id": "user_1"}}

# Multi-turn
agent.invoke({"messages": [("user", "Sản phẩm nào bán chạy nhất?")]}, config)
agent.invoke({"messages": [("user", "Lọc theo tháng trước")]}, config)  # Nhớ "sản phẩm bán chạy"
agent.invoke({"messages": [("user", "Cho biết số lượng")]}, config)
```

---

## 14. Demo Complete

```python
import os
from langchain_openai import ChatOpenAI
from langchain_community.utilities import SQLDatabase
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

# Setup
db = SQLDatabase.from_uri(os.environ["DATABASE_URL"])
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

@tool
def list_tables() -> str:
    return ", ".join(db.get_usable_table_names())

@tool
def get_schema(tables: str) -> str:
    return db.get_table_info([t.strip() for t in tables.split(",")])

@tool
def run_query(query: str) -> str:
    if "select" not in query.lower():
        return "Only SELECT allowed"
    try:
        return db.run(query)[:5000]
    except Exception as e:
        return f"Error: {e}"

agent = create_react_agent(
    llm,
    [list_tables, get_schema, run_query],
    state_modifier="""SQL Expert. Use tools to answer questions.

Process:
1. Identify tables needed
2. Get their schema
3. Write SQL
4. Execute
5. Explain results in Vietnamese"""
)

# Test
questions = [
    "Bảng nào có trong database?",
    "Top 5 customers theo doanh thu",
    "Sản phẩm nào chưa bán được?",
]

for q in questions:
    print(f"\n=== {q} ===")
    result = agent.invoke({"messages": [("user", q)]})
    print(result["messages"][-1].content)
```

---

## 15. Bài Tập

### Bài 1: Multi-Database
Agent có thể query nhiều databases khác nhau.

### Bài 2: Auto-Suggest Charts
LLM tự quyết định bar/line/pie chart phù hợp.

### Bài 3: Permissions
Per-user access control: user A chỉ thấy data của họ.

---

## 16. Checklist

- [ ] SQLDatabase setup
- [ ] Read-only safety
- [ ] Custom SQL tools
- [ ] Schema-aware retrieval cho large DB
- [ ] Visualizations
- [ ] Multi-turn memory
- [ ] UI Streamlit/web

➡️ **Tiếp theo**: [Code Review Agent](./06-Code-Review-Agent.md)
