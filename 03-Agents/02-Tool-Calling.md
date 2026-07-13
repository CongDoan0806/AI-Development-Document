# Tool Calling - Định Nghĩa Và Sử Dụng Tools

## 1. Tool Là Gì?

**Tool** = function mà LLM có thể "gọi". LLM không thực thi function trực tiếp, mà:
1. LLM **đề xuất** gọi tool với arguments
2. Code của bạn **thực thi** tool
3. Kết quả gửi lại LLM

```python
# Tool định nghĩa
def get_weather(city: str) -> str:
    return weather_api.fetch(city)

# LLM nhìn thấy:
{
    "name": "get_weather",
    "description": "...",
    "parameters": {"city": "string"}
}

# LLM trả về:
{
    "tool_calls": [
        {"name": "get_weather", "args": {"city": "Hà Nội"}}
    ]
}

# Code execute:
result = get_weather("Hà Nội")  # "25°C, nắng"

# Gửi lại LLM:
"Tool result: 25°C, nắng"
# LLM tiếp tục với info này
```

---

## 2. Tạo Tool Đơn Giản

### 2.1 Decorator @tool

```python
from langchain_core.tools import tool

@tool
def add(a: int, b: int) -> int:
    """Cộng 2 số nguyên"""
    return a + b

# Inspect
print(add.name)         # "add"
print(add.description)  # "Cộng 2 số nguyên"
print(add.args_schema.schema())
# {"properties": {"a": {"type": "integer"}, "b": {"type": "integer"}}, ...}
```

### 2.2 Quy Tắc Tool Tốt

✅ **Tên rõ ràng**: `get_user_orders` ✓, `do_stuff` ✗
✅ **Docstring chi tiết**: mô tả khi nào dùng, input là gì, output là gì
✅ **Type hints**: bắt buộc cho mọi tham số
✅ **Trả về string đơn giản**: dễ cho LLM hiểu

```python
@tool
def get_user_orders(user_id: int, status: str = "all") -> str:
    """
    Lấy danh sách đơn hàng của user.
    
    Args:
        user_id: ID của user (số nguyên)
        status: Trạng thái filter. Một trong: 'all', 'pending', 'completed', 'cancelled'
    
    Returns:
        Danh sách đơn hàng dạng string, hoặc 'No orders found' nếu không có
    """
    orders = db.query(user_id=user_id, status=status)
    if not orders:
        return "No orders found"
    return "\n".join(f"#{o.id}: {o.title} - {o.status}" for o in orders)
```

---

## 3. Pydantic Schema Cho Tool

Khi cần validation phức tạp:

```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool

class SearchInput(BaseModel):
    query: str = Field(description="Search query")
    max_results: int = Field(default=10, ge=1, le=50, description="Max results")
    category: str = Field(default="all", description="Filter category")

@tool(args_schema=SearchInput)
def search(query: str, max_results: int = 10, category: str = "all") -> str:
    """Search trong database với category và max results."""
    results = db.search(query, limit=max_results, category=category)
    return "\n".join(results)
```

---

## 4. Bind Tools Vào LLM

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

tools = [add, get_user_orders, search]
llm_with_tools = llm.bind_tools(tools)

# LLM giờ có thể trả về tool_calls
response = llm_with_tools.invoke("Tính 5 + 3")
print(response.tool_calls)
# [{"name": "add", "args": {"a": 5, "b": 3}, "id": "..."}]
```

### 4.1 Execute Tool Manually

```python
# Map tool name → tool object
tool_map = {t.name: t for t in tools}

# Execute
for tool_call in response.tool_calls:
    tool = tool_map[tool_call["name"]]
    result = tool.invoke(tool_call["args"])
    print(f"{tool_call['name']}({tool_call['args']}) = {result}")
```

---

## 5. Tool Calling Loop Đầy Đủ

```python
from langchain_core.messages import HumanMessage, ToolMessage

def run_agent(question, llm_with_tools, tool_map):
    messages = [HumanMessage(content=question)]
    
    while True:
        # 1. Gọi LLM
        ai_msg = llm_with_tools.invoke(messages)
        messages.append(ai_msg)
        
        # 2. Nếu không có tool call → done
        if not ai_msg.tool_calls:
            return ai_msg.content
        
        # 3. Execute tools
        for tc in ai_msg.tool_calls:
            tool = tool_map[tc["name"]]
            result = tool.invoke(tc["args"])
            messages.append(ToolMessage(
                content=str(result),
                tool_call_id=tc["id"]
            ))
        
        # 4. Loop tiếp

answer = run_agent("Tính 25 * 4 và lấy weather Hà Nội", llm_with_tools, tool_map)
print(answer)
```

→ Trong thực tế dùng `create_react_agent` (LangGraph) làm việc này cho bạn.

---

## 6. Built-in Tools

LangChain có sẵn nhiều tools:

### 6.1 Web Search - Tavily

```python
# pip install langchain-community tavily-python
from langchain_community.tools import TavilySearchResults

search = TavilySearchResults(
    max_results=3,
    tavily_api_key="..."
)

result = search.invoke("AI news 2024")
```

### 6.2 Wikipedia

```python
from langchain_community.tools import WikipediaQueryRun
from langchain_community.utilities import WikipediaAPIWrapper

wiki = WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper(lang="vi"))
result = wiki.invoke("Việt Nam")
```

### 6.3 Python REPL

```python
from langchain_experimental.tools import PythonREPLTool

python = PythonREPLTool()
result = python.invoke("print(sum(range(100)))")
```

### 6.4 Shell Commands

```python
from langchain_community.tools import ShellTool

shell = ShellTool()
result = shell.invoke({"commands": ["ls -la"]})
```

⚠️ **Cẩn thận**: cho LLM access shell rất rủi ro!

### 6.5 SQL

```python
from langchain_community.utilities import SQLDatabase
from langchain_community.tools.sql_database.tool import (
    InfoSQLDatabaseTool,
    ListSQLDatabaseTool,
    QuerySQLDataBaseTool,
)

db = SQLDatabase.from_uri("sqlite:///data.db")
sql_tools = [
    InfoSQLDatabaseTool(db=db),
    ListSQLDatabaseTool(db=db),
    QuerySQLDataBaseTool(db=db),
]
```

### 6.6 Email - Gmail

```python
from langchain_community.tools.gmail import GmailGetThread, GmailSendMessage
# Yêu cầu OAuth setup
```

### 6.7 File I/O

```python
from langchain_community.agent_toolkits import FileManagementToolkit

toolkit = FileManagementToolkit(root_dir="./workspace")
file_tools = toolkit.get_tools()
# read_file, write_file, list_directory, copy_file, ...
```

### 6.8 Arxiv - Search Research Papers

```bash
pip install arxiv
```

```python
from langchain_community.tools.arxiv.tool import ArxivQueryRun
from langchain_community.utilities import ArxivAPIWrapper

arxiv = ArxivQueryRun(api_wrapper=ArxivAPIWrapper())
result = arxiv.invoke("2303.15056")  # Paper ID
# Hoặc query bằng từ khóa
result = arxiv.invoke("attention is all you need")

# Cách nhanh hơn dùng load_tools (legacy nhưng tiện)
from langchain.agents import load_tools
tools = load_tools(["arxiv"])
```

### 6.9 SerpAPI - Google Search

Đăng ký tại https://serpapi.com/ để lấy API key.

```bash
pip install google-search-results
```

```python
import os
os.environ["SERPAPI_API_KEY"] = "..."

from langchain_community.utilities import SerpAPIWrapper

search = SerpAPIWrapper()
result = search.run("Thủ tướng Việt Nam hiện tại")

# Dùng qua load_tools
from langchain.agents import load_tools
tools = load_tools(["serpapi"])

# Hoặc làm tool
from langchain_core.tools import Tool

google_tool = Tool(
    name="google_search",
    description="Search Google for current information",
    func=search.run,
)
```

### 6.10 Wolfram Alpha - Math & Science

Tốt cho **math, physics, chemistry, geography**.

Đăng ký AppID tại https://developer.wolframalpha.com/

```bash
pip install wolframalpha
```

```python
import os
os.environ["WOLFRAM_ALPHA_APPID"] = "..."

from langchain_community.utilities import WolframAlphaAPIWrapper

wolfram = WolframAlphaAPIWrapper()
result = wolfram.run("What is the integral of x^2 from 0 to 5?")
# → "5^3/3 = 125/3 ≈ 41.67"

# Qua load_tools
from langchain.agents import load_tools
tools = load_tools(["wolfram-alpha"])
```

**Wolfram Alpha trả lời được**:
- "What is the speed of light?"
- "Population of Vietnam"
- "Integral of sin(x)"
- "Distance from Earth to Moon"
- "Asthenosphere definition"

### 6.11 LLM-Math - Tính Toán Bằng LLM + Python

Tool để LLM tự generate code Python tính toán expression.

```python
from langchain.agents import load_tools
from langchain_openai import ChatOpenAI

# llm-math cần truyền llm để generate code
llm = ChatOpenAI(model="gpt-4o-mini")
tools = load_tools(["llm-math"], llm=llm)

# Tool nhận expression và evaluate
# Ví dụ: "100 * 25 / 5 + sqrt(144)"
```

### 6.12 DuckDuckGo Search (Free, Không API Key)

```python
from langchain_community.tools import DuckDuckGoSearchRun

search = DuckDuckGoSearchRun()
result = search.invoke("LangChain framework")
```

### 6.13 Combine Tools Phổ Biến

```python
from langchain.agents import load_tools
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# Load nhiều tool cùng lúc
tools = load_tools(
    ["serpapi", "llm-math", "wikipedia", "arxiv", "wolfram-alpha"],
    llm=llm,  # llm-math cần
)
```

---

## 7. Toolkits - Group Of Tools

Một số toolkit cung cấp sẵn nhiều tools liên quan:

```python
from langchain_community.agent_toolkits import SQLDatabaseToolkit
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("sqlite:///data.db")
toolkit = SQLDatabaseToolkit(db=db, llm=llm)
tools = toolkit.get_tools()

# Tự động có: list_tables, info_table, query_sql, query_checker
```

---

## 8. Custom Tool Nâng Cao

### 8.1 Async Tool

```python
import asyncio
import aiohttp

@tool
async def fetch_url(url: str) -> str:
    """Fetch content từ URL (async)"""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()
```

### 8.2 Tool Với State

```python
from langchain_core.tools import tool

class Database:
    def __init__(self):
        self.data = {}
    
    def save(self, key, value):
        self.data[key] = value
        return f"Saved {key}"

db = Database()

@tool
def save_data(key: str, value: str) -> str:
    """Lưu key-value vào database"""
    return db.save(key, value)

@tool
def get_data(key: str) -> str:
    """Lấy value theo key"""
    return db.data.get(key, "Not found")
```

### 8.3 Tool Với Side Effects

```python
@tool
def send_notification(user_id: int, message: str) -> str:
    """Gửi notification đến user. Lưu ý: action không undo được."""
    notification_service.send(user_id, message)
    log_action("notification_sent", user_id, message)
    return f"Sent to user {user_id}"
```

### 8.4 Multi-Argument Tool

```python
from typing import List

@tool
def schedule_meeting(
    title: str,
    start_time: str,
    duration_minutes: int,
    attendees: List[str],
    location: str = "Online"
) -> str:
    """Schedule a meeting.
    
    Args:
        title: Meeting title
        start_time: ISO format e.g. '2024-12-01T14:00:00'
        duration_minutes: Duration in minutes
        attendees: List of attendee emails
        location: Meeting location, default 'Online'
    """
    meeting = calendar.create(
        title=title,
        start=start_time,
        duration=duration_minutes,
        attendees=attendees,
        location=location,
    )
    return f"Created meeting #{meeting.id}"
```

---

## 9. Class-Based Tool (Nâng Cao)

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Type

class WeatherInput(BaseModel):
    city: str = Field(description="City name")
    units: str = Field(default="celsius", description="celsius or fahrenheit")

class WeatherTool(BaseTool):
    name: str = "get_weather"
    description: str = "Get current weather for a city"
    args_schema: Type[BaseModel] = WeatherInput
    
    def _run(self, city: str, units: str = "celsius") -> str:
        # Sync implementation
        return f"{city}: 25°{'C' if units == 'celsius' else 'F'}"
    
    async def _arun(self, city: str, units: str = "celsius") -> str:
        # Async implementation
        return self._run(city, units)

weather = WeatherTool()
```

---

## 10. StructuredTool - Tạo Tool Từ Function

```python
from langchain_core.tools import StructuredTool

def my_function(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

tool = StructuredTool.from_function(
    func=my_function,
    name="custom_add",
    description="Add two integers",
)
```

---

## 11. Returning Complex Objects

LLM thấy output dưới dạng string. Nếu tool trả object, LLM sẽ thấy stringified.

```python
@tool
def get_user(user_id: int) -> dict:
    """Lấy thông tin user"""
    return {"id": user_id, "name": "An", "email": "a@example.com"}

# LLM thấy: '{"id": 1, "name": "An", "email": "a@example.com"}'
```

**Tip**: format string đẹp cho LLM dễ đọc:
```python
@tool
def get_user(user_id: int) -> str:
    user = db.get(user_id)
    return f"User #{user.id}: {user.name} ({user.email})"
```

---

## 12. ToolMessage - Trả Về Custom Data

```python
from langchain_core.messages import ToolMessage

@tool
def my_tool(x: int) -> ToolMessage:
    """Tool returning structured data"""
    return ToolMessage(
        content=f"Result: {x * 2}",
        tool_call_id="placeholder",  # Sẽ được override
        artifact={"raw_data": [1, 2, 3]},  # Data không show cho LLM
    )
```

---

## 13. Tool Error Handling

```python
from langchain_core.tools import ToolException

@tool
def divide(a: float, b: float) -> float:
    """Chia 2 số"""
    if b == 0:
        raise ToolException("Cannot divide by zero")
    return a / b

# LLM sẽ thấy error và tự retry với input khác
```

### Handle Errors In Agent

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    llm,
    tools,
    # Tự động handle tool errors
)
```

---

## 14. Tool Calling Với Different Models

### 14.1 OpenAI
```python
# Native tool calling
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)
```

### 14.2 Anthropic Claude
```python
# Native tool calling
llm = ChatAnthropic(model="claude-sonnet-4-6").bind_tools(tools)
```

### 14.3 Gemini
```python
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash").bind_tools(tools)
```

### 14.4 Ollama Local
```python
# Llama 3.1+ hỗ trợ tool calling
llm = ChatOllama(model="llama3.1").bind_tools(tools)
```

---

## 15. Parallel Tool Calling

LLM có thể gọi nhiều tools song song trong 1 turn:

```python
response = llm_with_tools.invoke("Get weather for Hà Nội AND TPHCM AND Đà Nẵng")
print(len(response.tool_calls))  # 3
# Execute parallel
results = await asyncio.gather(*[
    tool_map[tc["name"]].ainvoke(tc["args"])
    for tc in response.tool_calls
])
```

---

## 16. Demo: Complete Tool Calling Flow

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, ToolMessage
from typing import List

load_dotenv()

# === Tools ===
@tool
def search_product(name: str) -> str:
    """Search product by name"""
    products = {
        "iphone": "iPhone 15 Pro - $999, in stock",
        "macbook": "MacBook Pro M3 - $1999, in stock",
        "ipad": "iPad Air - $599, out of stock",
    }
    for key, val in products.items():
        if key in name.lower():
            return val
    return "Product not found"

@tool
def get_shipping_cost(city: str, weight_kg: float) -> str:
    """Calculate shipping cost"""
    rates = {"Hà Nội": 50000, "TPHCM": 80000, "Đà Nẵng": 70000}
    base = rates.get(city, 100000)
    cost = base + (weight_kg - 0.5) * 20000
    return f"Shipping to {city}: {cost:.0f} VND"

@tool
def create_order(product: str, quantity: int, address: str) -> str:
    """Create order"""
    order_id = f"ORD-{hash((product, quantity, address)) % 10000:04d}"
    return f"Order created: {order_id}, {product} x{quantity}, ship to {address}"

# === LLM ===
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
tools = [search_product, get_shipping_cost, create_order]
llm_with_tools = llm.bind_tools(tools)
tool_map = {t.name: t for t in tools}

# === Loop ===
def run_agent(question: str, max_iterations: int = 5):
    messages = [HumanMessage(content=question)]
    
    for i in range(max_iterations):
        ai_msg = llm_with_tools.invoke(messages)
        messages.append(ai_msg)
        
        print(f"\n--- Iteration {i+1} ---")
        if ai_msg.content:
            print(f"AI: {ai_msg.content}")
        
        if not ai_msg.tool_calls:
            return ai_msg.content
        
        for tc in ai_msg.tool_calls:
            print(f"Tool call: {tc['name']}({tc['args']})")
            try:
                result = tool_map[tc["name"]].invoke(tc["args"])
                print(f"  Result: {result}")
            except Exception as e:
                result = f"Error: {e}"
                print(f"  Error: {e}")
            
            messages.append(ToolMessage(
                content=str(result),
                tool_call_id=tc["id"]
            ))
    
    return "Max iterations reached"

# Test
answer = run_agent(
    "Tôi muốn mua iPhone, ship đến Hà Nội. Đơn nặng 0.5kg. Tính tổng chi phí và tạo đơn ship đến 123 Đường ABC, Hà Nội."
)
print(f"\n=== Final ===\n{answer}")
```

---

## 17. Best Practices

### 17.1 Tool Naming
- ✅ `get_user_by_id`, `send_email`, `search_products`
- ❌ `func1`, `tool_a`, `do`

### 17.2 Description
Tốt:
```python
"""Search products by name. Returns list of matching products with price and stock status.
Use this when user asks about specific products or wants to compare prices."""
```

Tồi:
```python
"""Searches"""
```

### 17.3 Input Validation
```python
@tool
def transfer_money(amount: float, to_account: str) -> str:
    """Transfer money to account"""
    if amount <= 0:
        return "Error: amount must be positive"
    if amount > 1_000_000:
        return "Error: amount too large, requires manual approval"
    # Process...
```

### 17.4 Idempotency
Tools nên idempotent nếu có thể (gọi nhiều lần cùng args = cùng kết quả).

### 17.5 Read-Only Tools
Khi có thể, ưu tiên read-only. Write tools cần confirmation:

```python
@tool
def delete_user(user_id: int, confirm: bool = False) -> str:
    """Delete user. Requires confirm=True."""
    if not confirm:
        return "Please call again with confirm=True to delete"
    db.delete_user(user_id)
    return f"Deleted user {user_id}"
```

---

## 18. Bài Tập

### Bài 1: Calculator Tools
Build calculator agent với tools: add, subtract, multiply, divide, power, sqrt.

### Bài 2: API Wrapper Agent
Wrap 3 public APIs thành tools (e.g., REST Countries, GitHub API, JSONPlaceholder). Agent có thể truy vấn cả 3.

### Bài 3: Bank Account Agent
Tools: get_balance, deposit, withdraw, transfer. Implement validation, confirmation. Test edge cases.

---

## 19. Checklist

- [ ] Tạo tool với @tool decorator
- [ ] Pydantic schema cho complex tool
- [ ] Bind tools vào LLM
- [ ] Execute tool manually
- [ ] Built-in tools (Tavily, Wikipedia, SQL)
- [ ] Toolkits
- [ ] Async tool
- [ ] Error handling
- [ ] Parallel tool calling

➡️ **Tiếp theo**: [Agent Planning](./03-Agent-Planning.md)
