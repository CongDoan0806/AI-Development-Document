# Giới Thiệu MCP (Model Context Protocol)

## 1. MCP Là Gì?

**Model Context Protocol (MCP)** là **open standard** do **Anthropic** giới thiệu cuối 2024, cho phép kết nối **LLM/AI applications** với **mọi nguồn dữ liệu và tools** theo cách chuẩn hóa.

```
Trước MCP:
[Claude Desktop] → custom integration → [GitHub]
[Cursor] → another integration → [Slack]
[Your App] → yet another → [Database]
Mỗi tool x mỗi app = N×M integrations

Với MCP:
[Any AI App] ↔ MCP Protocol ↔ [Any Server]
Chuẩn 1 protocol → N + M integrations
```

**MCP = USB-C của AI**: 1 chuẩn cho mọi connection.

---

## 2. Tại Sao Cần MCP?

### Vấn Đề Trước MCP

Để Claude/ChatGPT đọc file/database/API của bạn:
- Mỗi app có cách integration khác nhau
- Tool definitions không reusable
- Khó share tool giữa các AI apps
- Phải code tool calling từng app

### MCP Solve

- **Chuẩn hóa**: 1 protocol cho mọi AI app
- **Reusable**: viết server 1 lần, dùng cho Claude Desktop, Cursor, Cline, ...
- **Marketplace**: cộng đồng share MCP servers
- **Security**: granular permissions

---

## 3. Kiến Trúc MCP

```
┌──────────────────┐       ┌──────────────────┐
│  Host            │       │  MCP Server      │
│  (AI Application)│ ──→   │  (Resources/Tools)│
│                  │ ←──   │                  │
│  e.g.:           │       │  e.g.:           │
│  • Claude Desktop│       │  • Filesystem    │
│  • Cursor        │       │  • GitHub        │
│  • Cline         │       │  • Slack         │
│  • Your custom   │       │  • Database      │
└──────────────────┘       │  • Custom        │
                            └──────────────────┘
                                    ↑
                            Communication via:
                            • stdio (local)
                            • HTTP+SSE (remote)
```

### Components

- **Host**: AI app (Claude Desktop, IDE, ...)
- **Client**: connection logic trong host
- **Server**: cung cấp resources/tools
- **Protocol**: JSON-RPC qua stdio hoặc HTTP

---

## 4. MCP Cung Cấp 3 Loại Capabilities

### 4.1 Resources
Read-only data sources mà LLM có thể đọc:
- Files
- Database rows
- API responses
- Documents

### 4.2 Tools
Functions LLM có thể call (action):
- Run SQL query
- Send Slack message
- Create GitHub issue
- Execute shell command

### 4.3 Prompts
Reusable prompt templates server provide:
- "Summarize this document"
- "Generate commit message"

---

## 5. Cài Đặt

### Python SDK

```bash
pip install mcp
```

### TypeScript/Node

```bash
npm install @modelcontextprotocol/sdk
```

---

## 6. Hello World MCP Server (Python)

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("hello-server")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Cộng 2 số"""
    return a + b

@mcp.tool()
def get_greeting(name: str) -> str:
    """Tạo lời chào"""
    return f"Xin chào, {name}!"

@mcp.resource("hello://welcome")
def welcome_message() -> str:
    """Welcome message resource"""
    return "Welcome to my MCP server!"

if __name__ == "__main__":
    mcp.run()
```

Run:
```bash
python server.py
```

---

## 7. Connect Claude Desktop Tới Server

### macOS

Sửa file `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "hello-server": {
      "command": "python",
      "args": ["/path/to/server.py"]
    }
  }
}
```

### Windows

Sửa `%APPDATA%\Claude\claude_desktop_config.json` tương tự.

Restart Claude Desktop → tools sẽ xuất hiện.

---

## 8. Public MCP Servers Có Sẵn

| Server | Function |
|--------|----------|
| `filesystem` | Read/write files |
| `github` | GitHub API |
| `gitlab` | GitLab API |
| `slack` | Slack workspace |
| `postgres` | Postgres database |
| `sqlite` | SQLite database |
| `google-drive` | Google Drive |
| `brave-search` | Web search |
| `puppeteer` | Browser automation |
| `memory` | Persistent memory |

### Install Filesystem Server

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/me/Documents"
      ]
    }
  }
}
```

---

## 9. MCP Server Phổ Biến: Filesystem Demo

Sau khi connect:
```
User (in Claude): "List files in my Documents folder"

Claude (using MCP):
→ Call tool: list_directory("/Users/me/Documents")
→ Get result
→ "Here are the files: ..."
```

---

## 10. MCP vs LangChain Tools

| | LangChain Tool | MCP |
|---|----------------|-----|
| **Scope** | LangChain ecosystem | Universal (any AI app) |
| **Discovery** | Code import | Runtime discovery |
| **Sharing** | Python package | MCP server marketplace |
| **Auth** | Per tool | Standard |
| **Multi-app** | Khó | Sẵn sàng |

**Khi nào dùng**:
- **LangChain tool**: dùng trong LangChain app, custom logic
- **MCP**: muốn share tools cross-apps, hoặc dùng tools có sẵn

→ Có thể **combine**: dùng MCP server làm LangChain tool (sẽ học sau).

---

## 11. Protocol Details

### Message Format (JSON-RPC 2.0)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "add",
    "arguments": {"a": 5, "b": 3}
  }
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {"type": "text", "text": "8"}
    ]
  }
}
```

### Available Methods

| Method | Purpose |
|--------|---------|
| `initialize` | Handshake |
| `tools/list` | List available tools |
| `tools/call` | Execute tool |
| `resources/list` | List resources |
| `resources/read` | Read resource |
| `prompts/list` | List prompts |
| `prompts/get` | Get prompt |

---

## 12. Transports

### 12.1 stdio (Local)

Server chạy như subprocess, communicate qua stdin/stdout.

```bash
python server.py
```

Dùng cho: local tools, development.

### 12.2 HTTP + SSE (Remote)

Server chạy như HTTP service.

```python
mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

Dùng cho: remote servers, production.

### 12.3 Streamable HTTP (Mới Nhất 2025)

Bi-directional streaming over HTTP. Replace SSE.

---

## 13. Security Considerations

⚠️ **MCP có thể nguy hiểm**:
- Tool có thể xóa file
- Server có thể access sensitive data
- Malicious server lừa LLM

### Best Practices

✅ **Whitelist servers**: chỉ dùng từ nguồn tin cậy

✅ **Sandbox**: chạy server trong container

✅ **Read-only mặc định**: yêu cầu confirm cho write actions

✅ **Audit log**: log mọi tool calls

✅ **Limit scope**: cho server access path/data cụ thể

---

## 14. Roadmap Phase 5

1. ✅ Giới thiệu MCP (file này)
2. ⏭️ [MCP Architecture](./02-MCP-Architecture.md)
3. ⏭️ [Build MCP Server](./03-MCP-Server.md)
4. ⏭️ [Build MCP Client](./04-MCP-Client.md)
5. ⏭️ [MCP Examples & Integration](./05-MCP-Examples.md)

---

## 15. Tài Nguyên

- 📚 Docs: https://modelcontextprotocol.io/
- 🐙 GitHub: https://github.com/modelcontextprotocol
- 🛒 Server marketplace: https://github.com/modelcontextprotocol/servers
- 🐍 Python SDK: https://github.com/modelcontextprotocol/python-sdk
- 📺 Anthropic announcement: https://www.anthropic.com/news/model-context-protocol

---

## 16. Checklist

- [ ] Hiểu MCP là gì và tại sao cần
- [ ] Phân biệt Host/Client/Server
- [ ] Cài đặt MCP Python SDK
- [ ] Build được Hello World server
- [ ] Connect với Claude Desktop
- [ ] Hiểu Resources/Tools/Prompts
- [ ] Hiểu transports (stdio, SSE, HTTP)

➡️ **Tiếp theo**: [MCP Architecture](./02-MCP-Architecture.md)
