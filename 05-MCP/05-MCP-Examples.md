# MCP Examples & Integration

## 1. Real-World Use Cases

### 1.1 Code Assistant
- MCP servers: filesystem, git, GitHub, language servers
- Use case: AI có thể đọc code, run tests, create PRs

### 1.2 Data Analyst
- MCP servers: database, spreadsheet, BI tools
- Use case: AI query data, generate reports

### 1.3 DevOps Assistant
- MCP servers: AWS, k8s, monitoring, terraform
- Use case: AI deploy, monitor, troubleshoot

### 1.4 Personal Assistant
- MCP servers: calendar, email, contacts, notes
- Use case: AI quản lý lịch trình

---

## 2. Official MCP Servers

| Server | Function |
|--------|----------|
| `@modelcontextprotocol/server-filesystem` | File I/O |
| `@modelcontextprotocol/server-github` | GitHub API |
| `@modelcontextprotocol/server-gitlab` | GitLab API |
| `@modelcontextprotocol/server-google-maps` | Google Maps |
| `@modelcontextprotocol/server-postgres` | Postgres DB |
| `@modelcontextprotocol/server-sqlite` | SQLite DB |
| `@modelcontextprotocol/server-slack` | Slack |
| `@modelcontextprotocol/server-brave-search` | Web search |
| `@modelcontextprotocol/server-puppeteer` | Browser automation |
| `@modelcontextprotocol/server-memory` | Persistent memory |
| `@modelcontextprotocol/server-fetch` | HTTP requests |
| `@modelcontextprotocol/server-everart` | AI image gen |

### Install
```bash
# Filesystem (built-in)
npx @modelcontextprotocol/server-filesystem /path

# GitHub (needs token)
GITHUB_TOKEN=ghp_... npx @modelcontextprotocol/server-github
```

---

## 3. Claude Desktop Config

`~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/me/Documents",
        "/Users/me/Projects"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."
      }
    },
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://user:pass@localhost/db"
      ]
    },
    "my-custom": {
      "command": "python",
      "args": ["/Users/me/scripts/my_mcp_server.py"]
    }
  }
}
```

Restart Claude Desktop → tools appear.

---

## 4. Cursor IDE Config

`.cursor/mcp.json` trong project:

```json
{
  "mcpServers": {
    "project-tools": {
      "command": "node",
      "args": ["./scripts/mcp-server.js"]
    }
  }
}
```

---

## 5. Demo: Notion-like Notes Server

```python
# notes_server.py
from mcp.server.fastmcp import FastMCP
from datetime import datetime
import json
import os
from pathlib import Path

mcp = FastMCP("notes")

NOTES_DIR = Path.home() / ".mcp_notes"
NOTES_DIR.mkdir(exist_ok=True)

def get_note_path(note_id: str) -> Path:
    return NOTES_DIR / f"{note_id}.json"

@mcp.tool()
def create_note(title: str, content: str, tags: list[str] = None) -> dict:
    """Create a new note"""
    note_id = f"note_{datetime.now().strftime('%Y%m%d%H%M%S')}"
    note = {
        "id": note_id,
        "title": title,
        "content": content,
        "tags": tags or [],
        "created_at": datetime.now().isoformat(),
        "updated_at": datetime.now().isoformat(),
    }
    get_note_path(note_id).write_text(json.dumps(note, indent=2))
    return note

@mcp.tool()
def list_notes(tag: str = None) -> list[dict]:
    """List all notes, optionally filtered by tag"""
    notes = []
    for f in NOTES_DIR.glob("*.json"):
        note = json.loads(f.read_text())
        if tag is None or tag in note["tags"]:
            notes.append({
                "id": note["id"],
                "title": note["title"],
                "tags": note["tags"],
                "created_at": note["created_at"],
            })
    return sorted(notes, key=lambda n: n["created_at"], reverse=True)

@mcp.tool()
def get_note(note_id: str) -> dict:
    """Get full note content"""
    path = get_note_path(note_id)
    if not path.exists():
        return {"error": "Note not found"}
    return json.loads(path.read_text())

@mcp.tool()
def update_note(
    note_id: str, 
    title: str = None,
    content: str = None,
    tags: list[str] = None,
) -> dict:
    """Update existing note"""
    path = get_note_path(note_id)
    if not path.exists():
        return {"error": "Note not found"}
    note = json.loads(path.read_text())
    if title is not None:
        note["title"] = title
    if content is not None:
        note["content"] = content
    if tags is not None:
        note["tags"] = tags
    note["updated_at"] = datetime.now().isoformat()
    path.write_text(json.dumps(note, indent=2))
    return note

@mcp.tool()
def delete_note(note_id: str) -> dict:
    """Delete a note"""
    path = get_note_path(note_id)
    if path.exists():
        path.unlink()
        return {"deleted": note_id}
    return {"error": "Not found"}

@mcp.tool()
def search_notes(query: str) -> list[dict]:
    """Full-text search in notes"""
    results = []
    query_lower = query.lower()
    for f in NOTES_DIR.glob("*.json"):
        note = json.loads(f.read_text())
        if query_lower in note["title"].lower() or query_lower in note["content"].lower():
            results.append({
                "id": note["id"],
                "title": note["title"],
                "snippet": note["content"][:200],
            })
    return results

if __name__ == "__main__":
    mcp.run()
```

Add to Claude Desktop config:
```json
{
  "mcpServers": {
    "notes": {
      "command": "python",
      "args": ["/path/to/notes_server.py"]
    }
  }
}
```

---

## 6. Demo: Postgres Database Assistant

```python
# postgres_assistant.py
from mcp.server.fastmcp import FastMCP
import psycopg2
import os

mcp = FastMCP("postgres-assistant")

CONN_STR = os.environ.get("DATABASE_URL")

def get_conn():
    return psycopg2.connect(CONN_STR)

@mcp.tool()
def list_tables() -> list[str]:
    """List all tables in database"""
    with get_conn() as conn:
        cur = conn.cursor()
        cur.execute("""
            SELECT table_name 
            FROM information_schema.tables 
            WHERE table_schema='public'
            ORDER BY table_name
        """)
        return [r[0] for r in cur.fetchall()]

@mcp.tool()
def describe_table(table_name: str) -> list[dict]:
    """Get table schema"""
    with get_conn() as conn:
        cur = conn.cursor()
        cur.execute("""
            SELECT column_name, data_type, is_nullable
            FROM information_schema.columns
            WHERE table_name = %s
        """, (table_name,))
        return [
            {"name": r[0], "type": r[1], "nullable": r[2] == "YES"}
            for r in cur.fetchall()
        ]

@mcp.tool()
def query(sql: str, limit: int = 100) -> list[dict]:
    """Execute SELECT query. Only SELECT allowed."""
    if not sql.strip().lower().startswith("select"):
        return [{"error": "Only SELECT queries allowed"}]
    
    with get_conn() as conn:
        cur = conn.cursor()
        try:
            cur.execute(sql)
            if cur.description is None:
                return [{"error": "No results"}]
            columns = [d[0] for d in cur.description]
            rows = cur.fetchmany(limit)
            return [dict(zip(columns, row)) for row in rows]
        except Exception as e:
            return [{"error": str(e)}]

@mcp.resource("postgres://schema")
def get_schema() -> str:
    """Full database schema"""
    with get_conn() as conn:
        cur = conn.cursor()
        cur.execute("""
            SELECT table_name, column_name, data_type
            FROM information_schema.columns
            WHERE table_schema='public'
            ORDER BY table_name, ordinal_position
        """)
        rows = cur.fetchall()
    
    schema = {}
    for table, col, dtype in rows:
        schema.setdefault(table, []).append(f"  {col}: {dtype}")
    
    return "\n\n".join(
        f"Table: {t}\n" + "\n".join(cols)
        for t, cols in schema.items()
    )

if __name__ == "__main__":
    mcp.run()
```

---

## 7. Demo: Use MCP Trong LangGraph Agent

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

async def main():
    client = MultiServerMCPClient({
        "filesystem": {
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-filesystem",
                "/Users/me/Documents",
            ],
            "transport": "stdio",
        },
        "notes": {
            "command": "python",
            "args": ["/path/to/notes_server.py"],
            "transport": "stdio",
        },
    })
    
    # Lấy tools từ tất cả servers
    tools = await client.get_tools()
    print(f"Available tools: {[t.name for t in tools]}")
    
    # Create agent
    llm = ChatOpenAI(model="gpt-4o-mini")
    agent = create_react_agent(llm, tools)
    
    # Use
    result = await agent.ainvoke({
        "messages": [
            ("user", "Đọc file todo.md trong Documents và tạo notes cho từng todo item")
        ]
    })
    
    print(result["messages"][-1].content)
    
    await client.close()

asyncio.run(main())
```

---

## 8. MCP Server In Docker

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY server.py .

# Run as SSE server
CMD ["python", "server.py", "--http"]
```

```bash
docker build -t my-mcp-server .
docker run -p 8000:8000 -e API_KEY=... my-mcp-server
```

Config Claude Desktop:
```json
{
  "mcpServers": {
    "my-server": {
      "transport": "sse",
      "url": "http://localhost:8000/sse"
    }
  }
}
```

---

## 9. Best Practices Production

### 9.1 Security

✅ Whitelist paths/resources
✅ Auth/authz cho remote servers
✅ Audit log mọi tool calls
✅ Rate limit
✅ Sandbox execution

### 9.2 Reliability

✅ Health checks
✅ Graceful shutdown
✅ Connection retry
✅ Timeout per call

### 9.3 Performance

✅ Connection pooling
✅ Async I/O
✅ Cache static resources
✅ Batch where possible

### 9.4 Monitoring

✅ Log all calls với context
✅ Metrics (latency, error rate)
✅ Trace với OpenTelemetry

---

## 10. Debugging MCP Issues

### 10.1 Server không start

```bash
# Test trực tiếp
python server.py < /dev/null

# Check stderr
python server.py 2>&1
```

### 10.2 Claude không thấy tools

- Restart Claude Desktop hoàn toàn
- Check config JSON valid
- Check logs: `~/Library/Logs/Claude/`

### 10.3 Tool fail

- Dùng MCP Inspector: `npx @modelcontextprotocol/inspector python server.py`
- Add logging trong tool
- Check stderr output

---

## 11. Building MCP Ecosystem

### 11.1 Sharing Server

- Publish lên GitHub
- Add đến marketplace
- Document setup steps

### 11.2 Discoverability

- Add MCP servers list:
  - https://github.com/modelcontextprotocol/servers (official)
  - https://mcp.so/ (community)

---

## 12. Future Of MCP

- 🔮 **More transports**: WebRTC, gRPC
- 🔮 **Standard auth**: OAuth 2.1 universal
- 🔮 **Marketplace**: app store cho MCP servers
- 🔮 **Multi-modal**: video, audio resources
- 🔮 **Cross-vendor**: OpenAI, Google áp dụng MCP

---

## 13. Bài Tập

### Bài 1: Personal Productivity Server
Build MCP server combine: notes, todos, calendar (mock). Test với Claude Desktop.

### Bài 2: Wrap Existing API
Pick 1 public API (e.g., REST Countries, Pokemon API). Build MCP server wrapper.

### Bài 3: LangGraph + MCP App
Build app: agent dùng MCP filesystem + custom server cho task management. UI bằng Streamlit/Gradio.

---

## 14. Checklist Phase 5

- [ ] Hiểu MCP architecture
- [ ] Connect existing servers (filesystem, github)
- [ ] Build custom MCP server
- [ ] Build custom MCP client
- [ ] Integrate với LangChain/LangGraph
- [ ] Deploy MCP server (Docker)
- [ ] Debug với MCP Inspector

🎉 **Hoàn thành Phase 5 - MCP!**

➡️ **Tiếp theo**: [Phase 6 - Evaluation](../06-Evaluation/01-RAG-Evaluation.md)
