# Build MCP Client

## 1. Client Là Gì?

**Client** = component connect và sử dụng MCP server. Có 2 loại:
- **Built-in clients**: Claude Desktop, Cursor, Cline (chỉ cần config)
- **Custom client**: tự code (Python, TypeScript) để dùng MCP trong app riêng

---

## 2. Python Client - Basics

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
import asyncio

async def main():
    # Server parameters
    params = StdioServerParameters(
        command="python",
        args=["server.py"],
        env=None,
    )
    
    # Connect
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize
            await session.initialize()
            
            # List tools
            tools = await session.list_tools()
            print(f"Tools: {[t.name for t in tools.tools]}")
            
            # Call tool
            result = await session.call_tool("add", {"a": 5, "b": 3})
            print(f"Result: {result.content[0].text}")

asyncio.run(main())
```

---

## 3. Discover Capabilities

### 3.1 List Tools

```python
tools_response = await session.list_tools()
for tool in tools_response.tools:
    print(f"Tool: {tool.name}")
    print(f"  Description: {tool.description}")
    print(f"  Schema: {tool.inputSchema}")
```

### 3.2 List Resources

```python
resources_response = await session.list_resources()
for r in resources_response.resources:
    print(f"Resource: {r.uri} - {r.name}")
```

### 3.3 List Prompts

```python
prompts_response = await session.list_prompts()
for p in prompts_response.prompts:
    print(f"Prompt: {p.name}")
    print(f"  Args: {p.arguments}")
```

---

## 4. Call Tools

```python
result = await session.call_tool("search", {"query": "AI", "limit": 5})

# Parse content
for content in result.content:
    if content.type == "text":
        print(content.text)
    elif content.type == "image":
        # Save image
        with open("output.png", "wb") as f:
            f.write(base64.b64decode(content.data))

# Check if error
if result.isError:
    print("Tool returned error")
```

---

## 5. Read Resources

```python
result = await session.read_resource("file://documents/readme.md")
for content in result.contents:
    print(content.text)
```

---

## 6. Get Prompts

```python
result = await session.get_prompt("summarize", {"text": "Long text...", "style": "bullet"})
for msg in result.messages:
    print(f"{msg.role}: {msg.content.text}")
```

---

## 7. Connect Multiple Servers

```python
async def main():
    servers = {
        "filesystem": StdioServerParameters(
            command="npx",
            args=["@modelcontextprotocol/server-filesystem", "/data"],
        ),
        "github": StdioServerParameters(
            command="npx",
            args=["@modelcontextprotocol/server-github"],
            env={"GITHUB_TOKEN": "..."},
        ),
    }
    
    sessions = {}
    
    for name, params in servers.items():
        # Open each connection
        read, write = await stdio_client(params).__aenter__()
        session = await ClientSession(read, write).__aenter__()
        await session.initialize()
        sessions[name] = session
    
    # Use
    fs_tools = await sessions["filesystem"].list_tools()
    gh_tools = await sessions["github"].list_tools()
```

---

## 8. HTTP/SSE Client

```python
from mcp.client.sse import sse_client

async def main():
    async with sse_client("http://localhost:8000/sse") as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            # ...
```

---

## 9. Integrate MCP Vào LangChain

Convert MCP tools → LangChain tools:

```python
# pip install langchain-mcp-adapters
from langchain_mcp_adapters.client import MultiServerMCPClient

async def main():
    async with MultiServerMCPClient({
        "filesystem": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-filesystem", "/data"],
        },
        "github": {
            "command": "npx",
            "args": ["@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": "..."},
        },
    }) as client:
        # Get all tools
        tools = client.get_tools()
        
        # Use trong LangGraph agent
        from langgraph.prebuilt import create_react_agent
        from langchain_openai import ChatOpenAI
        
        llm = ChatOpenAI(model="gpt-4o-mini")
        agent = create_react_agent(llm, tools)
        
        result = await agent.ainvoke({
            "messages": [("user", "List my files and create a GitHub issue")]
        })
```

---

## 10. Custom Client Class

```python
class MCPClient:
    def __init__(self, server_params):
        self.params = server_params
        self.session = None
    
    async def __aenter__(self):
        self._stdio_ctx = stdio_client(self.params)
        read, write = await self._stdio_ctx.__aenter__()
        self._session_ctx = ClientSession(read, write)
        self.session = await self._session_ctx.__aenter__()
        await self.session.initialize()
        return self
    
    async def __aexit__(self, *args):
        await self._session_ctx.__aexit__(*args)
        await self._stdio_ctx.__aexit__(*args)
    
    async def list_tools(self):
        return (await self.session.list_tools()).tools
    
    async def call_tool(self, name, args):
        result = await self.session.call_tool(name, args)
        return result.content[0].text if result.content else None

# Use
async def main():
    params = StdioServerParameters(command="python", args=["server.py"])
    async with MCPClient(params) as client:
        tools = await client.list_tools()
        print(tools)
        result = await client.call_tool("add", {"a": 1, "b": 2})
        print(result)
```

---

## 11. Demo: Build Chatbot Với MCP

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage

llm = ChatAnthropic(model="claude-sonnet-4-6")

async def mcp_chatbot():
    params = StdioServerParameters(
        command="python",
        args=["sqlite_server.py"],
    )
    
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # Get tools as LangChain format
            tools_response = await session.list_tools()
            
            # Convert to LangChain tool format
            langchain_tools = []
            for tool in tools_response.tools:
                langchain_tools.append({
                    "name": tool.name,
                    "description": tool.description,
                    "input_schema": tool.inputSchema,
                })
            
            # Bind to LLM
            llm_with_tools = llm.bind_tools(langchain_tools)
            
            messages = []
            print("Chatbot ready. Type 'exit' to quit.\n")
            
            while True:
                user_input = input("You: ")
                if user_input == "exit":
                    break
                
                messages.append(HumanMessage(content=user_input))
                
                # Agent loop
                while True:
                    response = llm_with_tools.invoke(messages)
                    messages.append(response)
                    
                    if not response.tool_calls:
                        print(f"AI: {response.content}\n")
                        break
                    
                    # Execute tools
                    for tc in response.tool_calls:
                        result = await session.call_tool(tc["name"], tc["args"])
                        content = result.content[0].text if result.content else ""
                        messages.append(ToolMessage(
                            content=content,
                            tool_call_id=tc["id"],
                        ))

asyncio.run(mcp_chatbot())
```

---

## 12. Reconnection & Error Handling

```python
async def robust_client(params, max_retries=3):
    for attempt in range(max_retries):
        try:
            async with stdio_client(params) as (read, write):
                async with ClientSession(read, write) as session:
                    await session.initialize()
                    return session
        except Exception as e:
            print(f"Attempt {attempt+1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(2 ** attempt)
```

---

## 13. Sampling - Server Asks Client For LLM

```python
async def sampling_callback(request):
    """Server requests LLM completion"""
    messages = request.messages
    # Convert to your LLM format
    response = llm.invoke([
        (m.role, m.content.text) for m in messages
    ])
    return CreateMessageResult(
        role="assistant",
        content=TextContent(text=response.content),
        model=response.model,
    )

async with ClientSession(
    read, write,
    sampling_callback=sampling_callback,
) as session:
    # Server có thể request LLM thông qua client
    ...
```

---

## 14. Best Practices

✅ **Connection lifetime**: keep alive throughout app

✅ **Error handling**: retry on disconnect

✅ **Type conversion**: MCP types ↔ LangChain types

✅ **Tool filtering**: chọn subset tools nếu quá nhiều

✅ **Caching**: tool list, schemas ít thay đổi

✅ **Concurrent calls**: async/await tools in parallel

---

## 15. Bài Tập

### Bài 1: CLI Client
Build CLI tool có thể connect đến bất kỳ MCP server, list tools, gọi tool interactively.

### Bài 2: Multi-Server Agent
Connect 3 MCP servers (filesystem, github, weather) → agent dùng được tools từ cả 3.

### Bài 3: MCP-LangChain Wrapper
Build adapter convert MCP server → LangChain tools tự động.

---

## 16. Checklist

- [ ] Connect stdio server
- [ ] List tools/resources/prompts
- [ ] Call tools
- [ ] Read resources
- [ ] HTTP/SSE client
- [ ] Multi-server connection
- [ ] LangChain integration
- [ ] Custom client class

➡️ **Tiếp theo**: [MCP Examples](./05-MCP-Examples.md)
