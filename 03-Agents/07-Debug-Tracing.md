# Debug & Tracing Agent

## 1. Tại Sao Khó Debug Agent?

Agent debugging khó hơn code thường vì:
- **Non-deterministic**: cùng input → output khác nhau
- **Multi-step**: 1 lỗi ở step 3 có thể do step 1 cách 5 phút trước
- **LLM "black box"**: không biết tại sao quyết định gọi tool X
- **Tool side effects**: real action (sent email, charged card)

→ Cần **tracing tools** chuyên dụng.

---

## 2. Verbose Logging (Quick Debug)

### 2.1 Set Verbose

```python
from langchain.globals import set_verbose, set_debug

set_verbose(True)  # In ra mỗi step
# set_debug(True)  # Verbose hơn nữa
```

### 2.2 In Bash/Console

```python
result = agent.invoke({"messages": [("user", "...")]})
# Console sẽ in từng step
```

### 2.3 Pretty Print Messages

```python
result = agent.invoke({"messages": [("user", "...")]})
for msg in result["messages"]:
    msg.pretty_print()
```

Output:
```
================================ Human Message =================================
Tính 5 + 3

================================== Ai Message ==================================
Tool Calls:
  add (call_abc123)
 Call ID: call_abc123
  Args:
    a: 5
    b: 3

================================= Tool Message =================================
Name: add

8

================================== Ai Message ==================================
5 + 3 = 8
```

---

## 3. LangSmith ⭐ (Tool Chính)

**LangSmith** = platform observability cho LangChain.

### 3.1 Setup

```bash
pip install langsmith
```

```env
# .env
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=lsv2_...
LANGCHAIN_PROJECT=my-agent-project
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com  # Optional
```

### 3.2 Automatic Tracing

Khi env set, mọi LangChain call tự log lên LangSmith:

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
response = llm.invoke("Hello")

# Mở https://smith.langchain.com → xem trace
```

### 3.3 Inspect Trace

LangSmith hiển thị:
- 🌳 **Tree view**: agent → LLM → tool → LLM → ...
- ⏱️ **Latency** mỗi step
- 💰 **Token usage** và cost
- 📝 **Input/Output** mỗi step
- ❌ **Errors** highlight

### 3.4 Custom Run Metadata

```python
result = agent.invoke(
    {"messages": [("user", "...")]},
    config={
        "run_name": "Customer Support Query",
        "tags": ["customer-support", "production"],
        "metadata": {
            "user_id": "user_123",
            "session_id": "sess_456",
        },
    },
)
```

### 3.5 Tracing Group

```python
from langsmith import traceable

@traceable(run_type="chain")
def my_pipeline(question):
    # Mọi LLM call trong này được group thành 1 trace
    classified = classifier.invoke(question)
    if classified == "technical":
        return tech_agent.invoke(question)
    return general_agent.invoke(question)
```

---

## 4. Streaming Events

Stream tất cả events trong agent execution:

```python
async def debug_stream(question):
    async for event in agent.astream_events(
        {"messages": [("user", question)]},
        version="v2",
    ):
        kind = event["event"]
        name = event["name"]
        
        if kind == "on_chat_model_start":
            print(f"\n🤖 LLM Call: {name}")
        elif kind == "on_chat_model_stream":
            chunk = event["data"]["chunk"]
            if chunk.content:
                print(chunk.content, end="", flush=True)
        elif kind == "on_tool_start":
            print(f"\n🔧 Tool: {name}({event['data']['input']})")
        elif kind == "on_tool_end":
            print(f"\n✅ Tool result: {event['data']['output']}")
```

---

## 5. Callbacks - Built-in

### 5.1 StdOutCallbackHandler

```python
from langchain.callbacks import StdOutCallbackHandler

result = agent.invoke(
    {"messages": [...]},
    config={"callbacks": [StdOutCallbackHandler()]}
)
```

### 5.2 Custom Callback

```python
from langchain_core.callbacks import BaseCallbackHandler
from typing import Any, Dict, List

class DebugCallback(BaseCallbackHandler):
    def __init__(self):
        self.steps = []
    
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"\n📨 LLM input: {prompts[0][:200]}...")
    
    def on_llm_end(self, response, **kwargs):
        usage = response.llm_output.get("token_usage", {})
        print(f"📩 LLM output - tokens: {usage}")
    
    def on_tool_start(self, serialized, input_str, **kwargs):
        print(f"\n🔧 Tool {serialized['name']}: {input_str}")
    
    def on_tool_end(self, output, **kwargs):
        print(f"  ↳ Result: {output[:200]}")
    
    def on_tool_error(self, error, **kwargs):
        print(f"  ↳ Error: {error}")
    
    def on_chain_error(self, error, **kwargs):
        print(f"\n💥 Chain error: {error}")

debug_cb = DebugCallback()
result = agent.invoke(
    {"messages": [...]},
    config={"callbacks": [debug_cb]}
)
```

---

## 6. Token & Cost Tracking

### 6.1 get_openai_callback

```python
from langchain.callbacks import get_openai_callback

with get_openai_callback() as cb:
    result = agent.invoke({"messages": [...]})
    
    print(f"Total Tokens: {cb.total_tokens}")
    print(f"Prompt Tokens: {cb.prompt_tokens}")
    print(f"Completion Tokens: {cb.completion_tokens}")
    print(f"Total Cost (USD): ${cb.total_cost:.4f}")
```

### 6.2 Custom Cost Tracker

```python
class CostTracker(BaseCallbackHandler):
    def __init__(self):
        self.total_input = 0
        self.total_output = 0
        self.calls = 0
    
    def on_llm_end(self, response, **kwargs):
        if hasattr(response, "llm_output") and response.llm_output:
            usage = response.llm_output.get("token_usage", {})
            self.total_input += usage.get("prompt_tokens", 0)
            self.total_output += usage.get("completion_tokens", 0)
            self.calls += 1
    
    def report(self):
        # Tính theo pricing GPT-4o-mini
        cost = (self.total_input / 1_000_000 * 0.15 +
                self.total_output / 1_000_000 * 0.60)
        return {
            "calls": self.calls,
            "input_tokens": self.total_input,
            "output_tokens": self.total_output,
            "estimated_cost_usd": cost,
        }
```

---

## 7. Trace Để Replay

Save trace để debug offline:

```python
import json

class TraceSaver(BaseCallbackHandler):
    def __init__(self, output_file):
        self.trace = []
        self.output_file = output_file
    
    def on_llm_start(self, serialized, prompts, **kwargs):
        self.trace.append({
            "type": "llm_start",
            "prompts": prompts,
        })
    
    def on_llm_end(self, response, **kwargs):
        self.trace.append({
            "type": "llm_end",
            "output": response.dict(),
        })
    
    def on_tool_start(self, serialized, input_str, **kwargs):
        self.trace.append({
            "type": "tool_start",
            "name": serialized["name"],
            "input": input_str,
        })
    
    def on_tool_end(self, output, **kwargs):
        self.trace.append({
            "type": "tool_end",
            "output": output,
        })
    
    def save(self):
        with open(self.output_file, "w") as f:
            json.dump(self.trace, f, indent=2, default=str)

saver = TraceSaver("trace.json")
result = agent.invoke({"messages": [...]}, config={"callbacks": [saver]})
saver.save()
```

---

## 8. Inspect Graph Structure

```python
# Print graph
print(agent.get_graph().draw_ascii())

# Or as Mermaid
print(agent.get_graph().draw_mermaid())

# As PNG (cần graphviz)
agent.get_graph().draw_mermaid_png(output_file_path="agent_graph.png")
```

---

## 9. LangGraph Studio (UI Debug)

LangGraph Studio = desktop app debug agents visually.

```bash
# Cài
pip install langgraph-cli

# Run studio cho project
langgraph dev
```

UI cho phép:
- Visualize graph
- Step through execution
- Modify state
- Test với input khác nhau

---

## 10. OpenTelemetry Integration

Cho production observability:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
)

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(ConsoleSpanExporter())
)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("agent_run"):
    result = agent.invoke({"messages": [...]})
```

Export to Jaeger, Datadog, New Relic, ...

---

## 11. Debugging Strategies

### 11.1 Reproduce Trước Khi Fix

```python
# Save problematic input
problem_input = {
    "messages": [("user", "Failed query")],
    "config": {"thread_id": "..."}
}

# Save vào file
import pickle
with open("problem.pkl", "wb") as f:
    pickle.dump(problem_input, f)

# Replay sau
with open("problem.pkl", "rb") as f:
    inp = pickle.load(f)
    result = agent.invoke(inp)
```

### 11.2 Isolate Component

```python
# Test từng tool riêng
print(my_tool.invoke({"arg": "value"}))

# Test LLM riêng với prompt cụ thể
print(llm.invoke([("user", "...")]).content)

# Test retriever riêng
print(retriever.invoke("query"))
```

### 11.3 Reduce Tools / Steps

Khi agent có bug, **giảm tools** xuống còn 1-2 → easier debug.

### 11.4 Print State Khi LangGraph

```python
for state in agent.stream({"messages": [...]}):
    print("State:", state)
```

---

## 12. Production Monitoring

### 12.1 Metrics Cần Track

- Request count
- Latency (p50, p95, p99)
- Token usage (per request)
- Cost (per request, per user)
- Error rate
- Tool calls per request
- Iterations per request

### 12.2 Setup Prometheus + Grafana

```python
from prometheus_client import Counter, Histogram

REQUEST_COUNT = Counter("agent_requests_total", "Total requests", ["status"])
LATENCY = Histogram("agent_latency_seconds", "Request latency")
TOKENS = Counter("agent_tokens_total", "Tokens used", ["type"])

@LATENCY.time()
def agent_invoke(question):
    with get_openai_callback() as cb:
        try:
            result = agent.invoke({"messages": [("user", question)]})
            REQUEST_COUNT.labels(status="success").inc()
            TOKENS.labels(type="input").inc(cb.prompt_tokens)
            TOKENS.labels(type="output").inc(cb.completion_tokens)
            return result
        except Exception as e:
            REQUEST_COUNT.labels(status="error").inc()
            raise
```

---

## 13. Demo: Full Observability

```python
import os
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from langchain.callbacks import get_openai_callback

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_PROJECT"] = "debug-demo"

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

@tool
def search(query: str) -> str:
    """Search"""
    return f"Results for {query}"

@tool
def calculate(expr: str) -> str:
    """Calculate"""
    return str(eval(expr))

agent = create_react_agent(llm, [search, calculate])

# Custom callback
class FullDebug(BaseCallbackHandler):
    def __init__(self):
        self.start_time = None
    
    def on_chain_start(self, serialized, inputs, **kwargs):
        import time
        self.start_time = time.time()
        print(f"\n{'='*50}\n🚀 Start: {inputs}")
    
    def on_chain_end(self, outputs, **kwargs):
        import time
        elapsed = time.time() - self.start_time
        print(f"\n✅ Done in {elapsed:.2f}s")
    
    def on_tool_start(self, serialized, input_str, **kwargs):
        print(f"\n🔧 Tool {serialized['name']}: {input_str}")
    
    def on_tool_end(self, output, **kwargs):
        print(f"  ↳ {output[:100]}")

# Run
debug = FullDebug()

with get_openai_callback() as cb:
    result = agent.invoke(
        {"messages": [("user", "Search AI trends and calculate 25*4")]},
        config={
            "callbacks": [debug],
            "run_name": "Demo Query",
            "tags": ["demo", "test"],
            "metadata": {"user": "test_user"},
        },
    )
    
    print(f"\n📊 Stats:")
    print(f"   Tokens: {cb.total_tokens}")
    print(f"   Cost: ${cb.total_cost:.4f}")

print(f"\n💬 Answer: {result['messages'][-1].content}")
print(f"\n🔗 Check trace: https://smith.langchain.com")
```

---

## 14. Best Practices

✅ **Always LangSmith trong dev**: free tier rộng rãi

✅ **Log structured**: JSON logs dễ query

✅ **Sampling cho production**: trace 10% requests thay vì 100%

✅ **Alert on errors**: Slack/email khi error rate spike

✅ **Replay critical bugs**: save input để reproduce

✅ **Per-user tracking**: cost & usage per user

---

## 15. Bài Tập

### Bài 1: Setup LangSmith
Setup LangSmith cho 1 project agent. Run 10 queries, screenshot dashboard.

### Bài 2: Custom Cost Tracker
Implement tracker tính cost realtime với multiple model pricing (GPT-4o, Sonnet, Haiku).

### Bài 3: Replay Bug
Tạo bug intentionally (vd: timeout). Save trace, replay offline để debug.

---

## 16. Checklist

- [ ] Verbose & pretty print
- [ ] LangSmith setup và inspect traces
- [ ] Callbacks (built-in và custom)
- [ ] Token & cost tracking
- [ ] Save & replay traces
- [ ] Visualize graph
- [ ] Production monitoring với Prometheus
- [ ] Debug strategies

🎉 **Hoàn thành Phase 3 - Agents!**

➡️ **Tiếp theo**: [Phase 4 - LangGraph](../04-LangGraph/01-Gioi-Thieu-LangGraph.md)
