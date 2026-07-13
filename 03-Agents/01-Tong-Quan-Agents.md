# Tổng Quan AI Agents

## 1. AI Agent Là Gì?

**Agent** = hệ thống dùng LLM để **tự quyết định** action cần làm (thay vì chỉ trả lời text).

```
Traditional LLM:
  Input → LLM → Text Output

Agent:
  Input → LLM (Reasoning) → Action 1 → Observation → 
  LLM → Action 2 → Observation → ... → Final Answer
```

**Agent có thể**:
- 🔧 Gọi tools (function, API, database)
- 🧠 Lý luận đa bước (multi-step reasoning)
- 💾 Nhớ ngữ cảnh
- 🤝 Tương tác với agent khác
- 🛠️ Tự sửa lỗi

---

## 2. Khi Nào Dùng Agent?

### Cần Agent
- ✅ Task cần nhiều bước
- ✅ Cần truy cập tools/APIs động
- ✅ Workflow phức tạp, có branching
- ✅ Cần lý luận dạng "nếu... thì..."

### Không Cần Agent
- ❌ Q&A đơn giản (dùng RAG)
- ❌ Task 1 bước (dùng LCEL chain)
- ❌ Output đơn giản (dùng structured output)
- ❌ Latency critical (agent chậm hơn)

---

## 3. Lịch Sử Agent Patterns

### 3.1 ReAct (2022) - Reasoning + Acting

```
Question: Thủ đô của Pháp có dân số bao nhiêu?

Thought: Tôi cần tìm thủ đô Pháp trước
Action: search("thủ đô Pháp")
Observation: Paris

Thought: Bây giờ tìm dân số Paris
Action: search("dân số Paris")
Observation: 2.1 triệu

Final Answer: Thủ đô Pháp là Paris, dân số 2.1 triệu.
```

### 3.2 Plan-and-Execute (2023)

```
Step 1: Lập plan
- Tìm thủ đô Pháp
- Tìm dân số thủ đô

Step 2: Thực hiện từng step
```

### 3.3 Tool Calling (Native, 2024+)

OpenAI/Anthropic train model để **trực tiếp** trả về tool calls (không cần ReAct format).

→ Đây là **cách hiện đại nhất**.

---

## 4. Kiến Trúc Agent Hiện Đại

```
┌──────────────────────────────────────┐
│  Agent Loop                          │
│                                      │
│  1. User Input                       │
│  2. LLM with tools                   │
│     ├─ Trả về tool_calls hoặc       │
│     └─ Trả về final answer          │
│  3. Nếu có tool_calls:              │
│     a. Execute tools                 │
│     b. Append results to messages    │
│     c. Loop back to step 2           │
│  4. Return final answer              │
└──────────────────────────────────────┘
```

---

## 5. LangChain Agent Frameworks

### 5.1 LangChain Legacy Agents (vẫn dùng trong tutorial cũ)

API cũ với `initialize_agent`, `AgentType`. Vẫn hoạt động nhưng được khuyến cáo migrate sang LangGraph.

```python
from langchain.agents import load_tools, initialize_agent, AgentType
from langchain_community.chat_models import ChatCohere

chat = ChatCohere(model="command", max_tokens=256, temperature=0.1)

# Load built-in tools
tools = load_tools(["serpapi", "llm-math", "wikipedia"], llm=chat)

# Initialize agent
agent = initialize_agent(
    tools,
    llm=chat,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True,
    handle_parsing_errors=True,  # Tự retry nếu LLM output sai format
    max_iterations=10,
)

# Use
result = agent.invoke("Ai là tổng thống Mỹ hiện tại? Lấy tuổi của ông ấy, tính căn bậc 2.")
```

**Các AgentType chính**:
| AgentType | Mô tả |
|-----------|-------|
| `ZERO_SHOT_REACT_DESCRIPTION` | ReAct generic, dùng cho LLM completion |
| `CHAT_ZERO_SHOT_REACT_DESCRIPTION` | ReAct cho Chat model |
| `CONVERSATIONAL_REACT_DESCRIPTION` | ReAct + memory |
| `OPENAI_FUNCTIONS` | OpenAI function calling |
| `OPENAI_MULTI_FUNCTIONS` | OpenAI parallel function calling |
| `STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION` | Structured tool inputs |

**Khi nào dùng legacy API**:
- ✅ Đọc/maintain code cũ
- ✅ Tutorial/khóa học cũ
- ✅ Prototype nhanh với built-in tools (Wolfram, SerpAPI)

**Khi nào migrate sang LangGraph**:
- ✅ Code mới
- ✅ Production
- ✅ Cần state phức tạp, multi-agent, HITL

### 5.2 `create_react_agent` (Modern - LangChain) - Deprecated

API trung gian, tạo agent runnable từ prompt + tools.

```python
from langchain.agents import create_react_agent, AgentExecutor
from langchain import hub

prompt = hub.pull("hwchase17/react")
agent = create_react_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
result = executor.invoke({"input": "..."})
```

### 5.3 LangGraph ⭐ (Recommended 2024+)
- Stateful, flexible
- Cho production agent
- Dùng `from langgraph.prebuilt import create_react_agent`
- Sẽ học chi tiết ở Phase 4

### 5.4 OpenAI Assistants API
- Managed bởi OpenAI
- Built-in tools (code interpreter, file search)
- Less flexible nhưng dễ dùng

---

## 5.5 Demo: Legacy Agent Với Cohere

```python
import os
os.environ["COHERE_API_KEY"] = "..."
os.environ["SERPAPI_API_KEY"] = "..."
os.environ["WOLFRAM_ALPHA_APPID"] = "..."

from langchain_community.chat_models import ChatCohere
from langchain.agents import load_tools, initialize_agent, AgentType

chat = ChatCohere(model="command", max_tokens=256, temperature=0.1)

# Multi-tool agent
tools = load_tools(
    ["serpapi", "llm-math", "wikipedia", "arxiv", "wolfram-alpha"],
    llm=chat,
)

agent = initialize_agent(
    tools,
    chat,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True,
    handle_parsing_errors=True,
    prompt_truncation="AUTO",  # Cohere-specific: tự cắt prompt nếu quá dài
)

# Test
agent.invoke("Asthenosphere là gì? Và tính 25 * 4 + sqrt(144)")
# → Dùng Wolfram cho định nghĩa
# → Dùng llm-math cho tính toán

agent.invoke("Paper arXiv 2303.15056 nói về gì?")
# → Dùng arxiv tool

agent.invoke("Tin tức mới nhất về AI?")
# → Dùng SerpAPI
```

**Tip cho Cohere với legacy agent**:
- `prompt_truncation="AUTO"`: tự cắt prompt khi vượt context
- `handle_parsing_errors=True`: bắt buộc, vì Cohere có thể trả format hơi khác chuẩn ReAct
- `temperature` thấp (0-0.2) để output ổn định format

---

## 6. Hello World Agent

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

load_dotenv()

# 1. Define tools
@tool
def get_weather(city: str) -> str:
    """Lấy thông tin thời tiết của một thành phố"""
    weather_db = {
        "Hà Nội": "25°C, nắng",
        "TPHCM": "32°C, mưa",
        "Đà Nẵng": "28°C, mây",
    }
    return weather_db.get(city, "Không có dữ liệu")

@tool
def calculate(expression: str) -> str:
    """Tính biểu thức toán học. Input: '2+3*4'"""
    try:
        return str(eval(expression))
    except:
        return "Lỗi tính toán"

# 2. Create agent
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
agent = create_react_agent(llm, [get_weather, calculate])

# 3. Use
result = agent.invoke({
    "messages": [("user", "Hà Nội thời tiết thế nào? Và 25 nhân 4 bằng bao nhiêu?")]
})

for msg in result["messages"]:
    msg.pretty_print()
```

**Output flow**:
```
User: Hà Nội thời tiết thế nào? Và 25 nhân 4 bằng bao nhiêu?

AI: [tool_calls: get_weather("Hà Nội"), calculate("25*4")]

Tool: 25°C, nắng
Tool: 100

AI: Hà Nội đang 25°C, nắng. 25 × 4 = 100.
```

---

## 7. Roadmap Phase 3

1. ✅ Tổng quan (file này)
2. ⏭️ [Tool Calling](./02-Tool-Calling.md) - Định nghĩa và dùng tools
3. ⏭️ [Agent Planning](./03-Agent-Planning.md) - Plan trước, execute sau
4. ⏭️ [Multi-Step Reasoning](./04-Multi-Step-Reasoning.md) - ReAct, thought process
5. ⏭️ [Agent Memory](./05-Agent-Memory.md) - Lưu trữ history
6. ⏭️ [Error Handling](./06-Error-Handling.md) - Retry, fallback
7. ⏭️ [Debug & Tracing](./07-Debug-Tracing.md) - LangSmith, observability

---

## 8. Use Cases Thực Tế

### 8.1 Research Assistant
- Search web → tổng hợp
- Đọc PDF → trả lời câu hỏi
- So sánh thông tin nhiều nguồn

### 8.2 Code Agent
- Generate code
- Test code
- Fix bugs tự động

### 8.3 SQL Agent
- User hỏi câu hỏi tự nhiên
- Agent generate SQL
- Execute và trả kết quả

### 8.4 Customer Support
- Tra cứu database
- Gọi API tạo ticket
- Trả lời FAQ

### 8.5 Personal Assistant
- Quản lý lịch
- Gửi email
- Đặt món, đặt vé

---

## 9. Challenges Của Agent

### 9.1 Reliability
LLM có thể:
- Gọi sai tool
- Pass wrong arguments
- Loop vô hạn
- Hallucinate kết quả tool

### 9.2 Cost
- Mỗi step = 1 LLM call
- Agent có thể chạy 5-10 steps
- → đắt hơn nhiều so với 1 LLM call

### 9.3 Latency
- Mỗi step ~2-5s
- Tổng có thể 10-30s

### 9.4 Debugging
- Khó trace lỗi
- Non-deterministic
- → cần LangSmith hoặc OpenTelemetry

---

## 10. Best Practices

### 10.1 Bắt Đầu Đơn Giản
- Chain trước, agent sau
- Ít tool trước (3-5), nhiều tool sau

### 10.2 Tool Design
- Tên rõ ràng, descriptive
- Docstring chi tiết
- Type hints đầy đủ
- Return string hoặc dict simple

### 10.3 Constraints
- Max iterations (5-10)
- Timeout
- Cost budget

### 10.4 Validation
- Validate tool inputs
- Sanity check outputs
- Final answer review

### 10.5 Logging
- Log mỗi step
- Save trace
- Replay khi debug

---

## 11. Demo Nâng Cao: Multi-Tool Agent

```python
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from datetime import datetime

# === Tools ===

@tool
def get_current_time() -> str:
    """Lấy thời gian hiện tại"""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

@tool
def search_web(query: str) -> str:
    """Search web với query"""
    # Mock - thực tế dùng Tavily, Serper, ...
    return f"Top result for '{query}': ..."

@tool
def get_weather(city: str) -> str:
    """Get weather for a city"""
    return f"{city}: 28°C, sunny"

@tool
def book_meeting(title: str, time: str, attendees: str) -> str:
    """Book a meeting in calendar.
    
    Args:
        title: Meeting title
        time: Time in format YYYY-MM-DD HH:MM
        attendees: Comma-separated emails
    """
    return f"Booked: {title} at {time} with {attendees}"

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send email"""
    return f"Email sent to {to}: {subject}"

# === Agent ===

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

agent = create_react_agent(
    model=llm,
    tools=[get_current_time, search_web, get_weather, book_meeting, send_email],
    state_modifier="""Bạn là personal assistant. 
    
    Quy tắc:
    - Dùng tool để lấy thông tin cụ thể
    - Trước khi book meeting, check thời gian hiện tại
    - Trả lời ngắn gọn bằng tiếng Việt"""
)

# Use
result = agent.invoke({
    "messages": [
        ("user", "Book một meeting về 'Sprint planning' vào 14:00 ngày mai với a@example.com, sau đó gửi email confirm")
    ]
})

for m in result["messages"]:
    m.pretty_print()
```

---

## 12. Architecture Patterns

### 12.1 Single Agent
1 LLM + nhiều tools.

```
User → Agent → Tools → Agent → Answer
```

### 12.2 Hierarchical Agents
Boss agent điều phối worker agents.

```
User → Supervisor
         ├─ Researcher Agent
         ├─ Coder Agent
         └─ Reviewer Agent
```

### 12.3 Multi-Agent Collaboration
Các agent độc lập trao đổi.

```
Agent A ⇄ Agent B
   ↓        ↓
Shared State
```

→ Sẽ học chi tiết ở [Phase 4 - LangGraph](../04-LangGraph/06-Multi-Agent-System.md)

---

## 13. Bài Tập

### Bài 1: Math Agent
Tạo agent với tools: add, subtract, multiply, divide. Test: "(5+3)*2 - 10/2"

### Bài 2: Mini Research Assistant
Agent với tools: web_search, summarize, save_to_file. Test: "Tìm 3 bài về AI agents và lưu thành file"

### Bài 3: Calendar + Email Agent
Tools: get_calendar, book_event, send_email, get_contact.
Test: "Book lịch với John tuần sau và gửi email confirm"

---

## 14. Checklist

- [ ] Hiểu Agent vs Chain khác nhau
- [ ] Phân biệt ReAct, Tool Calling
- [ ] Build được agent đơn giản với `create_react_agent`
- [ ] Biết khi nào nên/không nên dùng agent
- [ ] Hiểu agent loop
- [ ] Định nghĩa được tools với decorator @tool

➡️ **Tiếp theo**: [Tool Calling](./02-Tool-Calling.md)
