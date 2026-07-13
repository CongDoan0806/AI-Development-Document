# Streaming - Real-Time Response

## 1. Streaming Là Gì?

Thay vì chờ LLM xong → trả full response, **stream** từng token khi LLM tạo ra:

```
Without streaming:
[wait 5s] → "Full response here"

With streaming:
[0.5s] "Hello"
[0.6s] " there"
[0.8s] ", how"
[1.0s] " are you?"
```

**Benefits**:
- 🚀 Perceived speed (~10x faster UX)
- ⏱️ Lower time-to-first-token (TTFT)
- 💬 Natural chat experience

---

## 2. LangChain Streaming

### 2.1 Sync Stream

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

for chunk in llm.stream("Viết bài thơ về AI"):
    print(chunk.content, end="", flush=True)
```

### 2.2 Async Stream

```python
import asyncio

async def stream_demo():
    async for chunk in llm.astream("Viết bài thơ"):
        print(chunk.content, end="", flush=True)

asyncio.run(stream_demo())
```

### 2.3 Stream Chain

```python
chain = prompt | llm | StrOutputParser()

for chunk in chain.stream({"topic": "AI"}):
    print(chunk, end="", flush=True)
```

---

## 3. Stream Events

Stream với info chi tiết hơn (LangChain v2):

```python
async def stream_events():
    async for event in chain.astream_events(
        {"topic": "AI"},
        version="v2",
    ):
        kind = event["event"]
        
        if kind == "on_chat_model_start":
            print("\n[LLM started]")
        elif kind == "on_chat_model_stream":
            chunk = event["data"]["chunk"]
            print(chunk.content, end="", flush=True)
        elif kind == "on_chat_model_end":
            print(f"\n[LLM done, tokens: {event['data']['output'].usage_metadata}]")
        elif kind == "on_chain_end":
            print("\n[Chain complete]")
```

### Filter Events

```python
async for event in chain.astream_events(
    {"input": "..."},
    version="v2",
    include_names=["my_llm"],  # Chỉ events từ runnable tên này
    include_types=["chat_model"],
    include_tags=["important"],
):
    ...
```

---

## 4. FastAPI Streaming Endpoint

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

app = FastAPI()
llm = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_template("{question}")
chain = prompt | llm | StrOutputParser()

async def generate(question: str):
    async for chunk in chain.astream({"question": question}):
        yield chunk

@app.get("/chat")
async def chat(question: str):
    return StreamingResponse(
        generate(question),
        media_type="text/plain",
    )
```

### Server-Sent Events (SSE)

```python
@app.get("/chat-sse")
async def chat_sse(question: str):
    async def event_stream():
        async for chunk in chain.astream({"question": question}):
            yield f"data: {chunk}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
    )
```

### Client (JavaScript)

```javascript
const eventSource = new EventSource("/chat-sse?question=Hello");

eventSource.onmessage = (event) => {
    if (event.data === "[DONE]") {
        eventSource.close();
        return;
    }
    document.getElementById("output").innerText += event.data;
};
```

---

## 5. Stream RAG Chain

```python
from langchain_core.runnables import RunnablePassthrough

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# Stream
async for chunk in rag_chain.astream("question"):
    print(chunk, end="")
```

---

## 6. Stream JSON Output

`JsonOutputParser` hỗ trợ stream partial JSON:

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()
chain = prompt | llm | parser

async for partial in chain.astream({"input": "..."}):
    print(partial)

# Output dần:
# {"title": "He"}
# {"title": "Hello"}
# {"title": "Hello", "body": ""}
# {"title": "Hello", "body": "World"}
```

---

## 7. Stream LangGraph Agent

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(llm, tools)

# Stream updates (each node)
for chunk in agent.stream({"messages": [...]}, stream_mode="updates"):
    print(chunk)

# Stream values (full state)
for chunk in agent.stream({"messages": [...]}, stream_mode="values"):
    print(chunk["messages"][-1])

# Stream messages
for msg in agent.stream({"messages": [...]}, stream_mode="messages"):
    print(msg)

# Stream tokens
async for event in agent.astream_events({"messages": [...]}, version="v2"):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="")
```

---

## 8. WebSocket Streaming

```python
from fastapi import FastAPI, WebSocket

@app.websocket("/ws/chat")
async def chat_websocket(ws: WebSocket):
    await ws.accept()
    
    while True:
        try:
            data = await ws.receive_json()
            question = data["question"]
            
            async for chunk in chain.astream({"question": question}):
                await ws.send_json({"type": "chunk", "content": chunk})
            
            await ws.send_json({"type": "done"})
        except Exception as e:
            await ws.send_json({"type": "error", "message": str(e)})
            break
```

---

## 9. Streamlit App

```python
import streamlit as st
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

st.title("Chat with AI")

if "messages" not in st.session_state:
    st.session_state.messages = []

# Show history
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.write(msg["content"])

# Input
if prompt := st.chat_input("Type message"):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.write(prompt)
    
    # Stream response
    with st.chat_message("assistant"):
        placeholder = st.empty()
        full_response = ""
        for chunk in llm.stream(prompt):
            full_response += chunk.content
            placeholder.write(full_response + "▌")
        placeholder.write(full_response)
    
    st.session_state.messages.append({"role": "assistant", "content": full_response})
```

---

## 10. Gradio App

```python
import gradio as gr

def chat_stream(message, history):
    response = ""
    for chunk in chain.stream({"input": message}):
        response += chunk
        yield response

demo = gr.ChatInterface(chat_stream)
demo.launch()
```

---

## 11. Handle Disconnections

```python
async def stream_safe(question: str):
    try:
        async for chunk in chain.astream({"question": question}):
            yield chunk
    except asyncio.CancelledError:
        # Client disconnected
        print("Client disconnected, stopping stream")
        raise
    except Exception as e:
        yield f"\n[ERROR: {e}]"
```

---

## 12. Buffer & Backpressure

Nếu network chậm, client không kịp consume:

```python
async def stream_with_buffer(question: str, max_buffer=10):
    buffer = asyncio.Queue(maxsize=max_buffer)
    
    async def producer():
        async for chunk in chain.astream({"question": question}):
            await buffer.put(chunk)
        await buffer.put(None)  # Sentinel
    
    asyncio.create_task(producer())
    
    while True:
        chunk = await buffer.get()
        if chunk is None:
            break
        yield chunk
```

---

## 13. Stream Tool Calls

Tool calling cũng có thể stream:

```python
async for event in agent.astream_events({"messages": [...]}, version="v2"):
    kind = event["event"]
    
    if kind == "on_tool_start":
        print(f"\n🔧 Calling {event['name']}...")
    elif kind == "on_tool_end":
        print(f"  ✓ Done")
    elif kind == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="")
```

---

## 14. Best Practices

✅ **Always stream for chat UX**

✅ **SSE for HTTP, WebSocket for bi-directional**

✅ **Show typing indicator** trước first chunk

✅ **Handle interruption**: cancel stream khi user navigate away

✅ **Buffer cho slow network**

✅ **Fallback non-stream** cho old browsers

---

## 15. Demo: Production Chat API

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
import json
import asyncio

app = FastAPI()
llm = ChatOpenAI(model="gpt-4o-mini")
prompt = ChatPromptTemplate.from_template("{message}")
chain = prompt | llm | StrOutputParser()

@app.post("/api/chat")
async def chat(request: Request):
    data = await request.json()
    message = data["message"]
    
    async def stream():
        try:
            # Initial event
            yield f"data: {json.dumps({'type': 'start'})}\n\n"
            
            # Stream chunks
            async for chunk in chain.astream({"message": message}):
                if await request.is_disconnected():
                    print("Client disconnected")
                    return
                
                event = {"type": "chunk", "content": chunk}
                yield f"data: {json.dumps(event)}\n\n"
            
            # Done
            yield f"data: {json.dumps({'type': 'done'})}\n\n"
        except Exception as e:
            yield f"data: {json.dumps({'type': 'error', 'message': str(e)})}\n\n"
    
    return StreamingResponse(
        stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        }
    )
```

---

## 16. Checklist

- [ ] Sync và async streaming
- [ ] Stream events với metadata
- [ ] FastAPI SSE endpoint
- [ ] WebSocket bi-directional
- [ ] Stream LangGraph agent
- [ ] Stream JSON output
- [ ] Streamlit/Gradio UI
- [ ] Handle disconnections

➡️ **Tiếp theo**: [LangServe & FastAPI](./04-LangServe-FastAPI.md)
