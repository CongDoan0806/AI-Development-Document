# MCP Architecture - Kiến Trúc Chi Tiết

## 1. Layers

```
┌─────────────────────────────────────┐
│  Application Layer                  │
│  (Claude Desktop, Cursor, Cline)    │
├─────────────────────────────────────┤
│  Client Layer (MCP Client)          │
│  - Manage connections                │
│  - Discover capabilities             │
│  - Send requests                     │
├─────────────────────────────────────┤
│  Protocol Layer (JSON-RPC 2.0)      │
│  - Request/Response                  │
│  - Notifications                     │
├─────────────────────────────────────┤
│  Transport Layer                    │
│  - stdio                             │
│  - HTTP+SSE                          │
│  - WebSocket                         │
├─────────────────────────────────────┤
│  Server Layer (MCP Server)          │
│  - Expose resources/tools/prompts    │
│  - Handle requests                   │
└─────────────────────────────────────┘
```

---

## 2. Lifecycle

### 2.1 Initialization

```
Client                    Server
  │                         │
  │── initialize ────────→  │
  │   (protocol_version,    │
  │    capabilities)         │
  │                         │
  │ ←── initialize_response ─│
  │     (server_info,       │
  │      capabilities)       │
  │                         │
  │── initialized ────────→ │
  │   (notification)        │
```

### 2.2 Discovery

```
  │── tools/list ──────────→ │
  │ ←── [tool1, tool2, ...] ─│

  │── resources/list ──────→ │
  │ ←── [res1, res2, ...] ───│

  │── prompts/list ───────→ │
  │ ←── [prompt1, ...] ──────│
```

### 2.3 Execution

```
  │── tools/call ──────────→ │
  │   (name, arguments)      │
  │ ←── result ──────────────│
```

---

## 3. Capabilities Negotiation

Client và server advertise capabilities:

```json
// Client → Server
{
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "roots": {"listChanged": true},
      "sampling": {}
    },
    "clientInfo": {
      "name": "claude-desktop",
      "version": "1.0.0"
    }
  }
}

// Server → Client
{
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {"listChanged": true},
      "resources": {"subscribe": true},
      "prompts": {}
    },
    "serverInfo": {
      "name": "my-server",
      "version": "1.0.0"
    }
  }
}
```

---

## 4. Resources Architecture

### 4.1 Resource Structure

```python
{
    "uri": "file:///path/to/document.md",
    "name": "Document",
    "description": "...",
    "mimeType": "text/markdown",
}
```

### 4.2 URI Schemes

| Scheme | Example |
|--------|---------|
| `file://` | Local files |
| `http(s)://` | Web resources |
| `git://` | Git resources |
| Custom | `mydb://table/123` |

### 4.3 Resource Templates (Dynamic)

```python
# URI template: file://documents/{path}
@mcp.resource("file://documents/{path}")
def get_document(path: str) -> str:
    return open(f"/data/{path}").read()
```

### 4.4 Subscribe to Changes

```
Client subscribes → resources/subscribe
Server notifies → notifications/resources/updated
```

---

## 5. Tools Architecture

### 5.1 Tool Structure

```json
{
  "name": "search",
  "description": "Search the database",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {"type": "string"},
      "limit": {"type": "integer", "default": 10}
    },
    "required": ["query"]
  }
}
```

### 5.2 Tool Result

```json
{
  "content": [
    {"type": "text", "text": "Result here"},
    {"type": "image", "data": "base64...", "mimeType": "image/png"}
  ],
  "isError": false
}
```

### 5.3 Content Types

- `text` - Text content
- `image` - Image (base64)
- `resource` - Reference to a resource

---

## 6. Prompts Architecture

```json
{
  "name": "summarize",
  "description": "Summarize text",
  "arguments": [
    {"name": "text", "description": "Text to summarize", "required": true},
    {"name": "max_words", "description": "Max words", "required": false}
  ]
}
```

Server returns rendered prompt:
```json
{
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "Please summarize the following in {max_words} words: {text}"
      }
    }
  ]
}
```

---

## 7. Sampling (Server → Client LLM)

Đặc biệt: server có thể request client gọi LLM.

```
Server: "Tôi cần gọi LLM. Client, gọi giúp tôi với prompt này"
Client: gọi LLM của user (using user's API key/quota) → trả về response
Server: dùng response để continue
```

Use case: server giúp generate sub-tasks mà không cần own API key.

---

## 8. Roots (Client → Server)

Client tell server: "đây là folder root mà bạn được phép access":

```json
{
  "method": "roots/list",
  "result": {
    "roots": [
      {"uri": "file:///Users/me/projects", "name": "Projects"}
    ]
  }
}
```

→ Security: server chỉ access trong roots.

---

## 9. Notifications

Server có thể notify client (không cần response):

```json
{
  "method": "notifications/tools/list_changed",
  "params": {}
}
```

Common notifications:
- `tools/list_changed` - Tools list thay đổi
- `resources/list_changed` - Resources thay đổi
- `resources/updated` - Resource specific updated
- `progress` - Long-running task progress

---

## 10. Authentication & Authorization

### 10.1 OAuth 2.1 (Standard)

```
1. Client → Server: request token
2. Server → User: authorize
3. User: confirms
4. Server → Client: access token
5. Client requests use token
```

### 10.2 API Key (Simple)

```json
{
  "env": {
    "API_KEY": "sk-..."
  }
}
```

---

## 11. Error Handling

### 11.1 Standard Errors

```json
{
  "error": {
    "code": -32600,
    "message": "Invalid Request",
    "data": {}
  }
}
```

Common codes:
- `-32700` Parse error
- `-32600` Invalid Request
- `-32601` Method not found
- `-32602` Invalid params
- `-32603` Internal error

### 11.2 Tool Errors

Tool có thể trả về error trong result:
```json
{
  "content": [{"type": "text", "text": "File not found"}],
  "isError": true
}
```

---

## 12. Pagination

Khi tools list lớn:

```json
{
  "method": "tools/list",
  "params": {"cursor": "abc123"}
}

// Response
{
  "result": {
    "tools": [...],
    "nextCursor": "def456"
  }
}
```

---

## 13. Progress Tracking

Long-running operations:

```python
@mcp.tool()
async def long_task(ctx: Context) -> str:
    for i in range(10):
        await ctx.report_progress(progress=i/10, total=1.0)
        # Do work
    return "Done"
```

Client nhận `notifications/progress`.

---

## 14. Best Practices

✅ **Versioning**: declare protocolVersion

✅ **Capabilities check**: chỉ dùng features cả 2 sides support

✅ **Graceful degradation**: nếu server không support feature X

✅ **Connection pooling**: cho HTTP transport

✅ **Timeout**: mỗi request có timeout

✅ **Reconnect**: handle disconnection

---

## 15. Checklist

- [ ] Hiểu các layers của MCP
- [ ] Lifecycle: init → discover → execute
- [ ] Resources với URI schemes
- [ ] Tools với inputSchema
- [ ] Prompts templates
- [ ] Sampling
- [ ] Roots cho security
- [ ] Notifications
- [ ] Auth & errors

➡️ **Tiếp theo**: [Build MCP Server](./03-MCP-Server.md)
