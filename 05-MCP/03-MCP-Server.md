# Build MCP Server

## 1. FastMCP - Easy Way

`FastMCP` là wrapper đơn giản giống FastAPI.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-server")

@mcp.tool()
def hello(name: str) -> str:
    """Greet someone"""
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run()
```

---

## 2. Tools

### 2.1 Simple Tool

```python
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

### 2.2 With Pydantic Schema

```python
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="Search query")
    limit: int = Field(default=10, ge=1, le=100)

@mcp.tool()
def search(input: SearchInput) -> list[str]:
    """Search database"""
    return db.search(input.query, limit=input.limit)
```

### 2.3 Async Tool

```python
@mcp.tool()
async def fetch_data(url: str) -> str:
    """Fetch data from URL"""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()
```

### 2.4 Tool With Context

```python
from mcp.server.fastmcp import Context

@mcp.tool()
async def long_task(input: str, ctx: Context) -> str:
    """Long-running task with progress"""
    for i in range(10):
        await ctx.info(f"Processing step {i}")
        await ctx.report_progress(i / 10, 1.0)
        # Work
    return "Done"
```

---

## 3. Resources

### 3.1 Static Resource

```python
@mcp.resource("config://app")
def get_config() -> str:
    """App configuration"""
    return json.dumps({"version": "1.0", "name": "My App"})
```

### 3.2 Dynamic Resource (Template)

```python
@mcp.resource("file://documents/{path}")
def get_doc(path: str) -> str:
    """Get document by path"""
    with open(f"/data/{path}") as f:
        return f.read()
```

### 3.3 Binary Resource

```python
from mcp.server.fastmcp import Image

@mcp.resource("image://logo")
def get_logo() -> Image:
    """Company logo"""
    return Image(path="/assets/logo.png")
```

---

## 4. Prompts

```python
@mcp.prompt()
def summarize(text: str, style: str = "concise") -> str:
    """Generate summary prompt"""
    return f"""Summarize this text in {style} style:

{text}

Summary:"""

@mcp.prompt()
def code_review(code: str, language: str) -> str:
    """Code review prompt"""
    return f"""Review this {language} code:

```{language}
{code}
```

Provide:
1. Issues found
2. Improvements
3. Refactored version"""
```

---

## 5. Running Server

### 5.1 stdio (Default)

```python
if __name__ == "__main__":
    mcp.run()  # stdio
```

Run:
```bash
python server.py
```

### 5.2 HTTP+SSE

```python
if __name__ == "__main__":
    mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

### 5.3 Both

```python
import sys

if __name__ == "__main__":
    if "--http" in sys.argv:
        mcp.run(transport="sse", port=8000)
    else:
        mcp.run()
```

---

## 6. Demo: SQLite Database Server

```python
from mcp.server.fastmcp import FastMCP
import sqlite3
from typing import Optional

mcp = FastMCP("sqlite-server")
DB_PATH = "data.db"

def get_conn():
    return sqlite3.connect(DB_PATH)

@mcp.tool()
def query(sql: str) -> list[dict]:
    """Execute SELECT query.
    
    Args:
        sql: SQL SELECT statement
    """
    if not sql.strip().lower().startswith("select"):
        return [{"error": "Only SELECT allowed"}]
    
    conn = get_conn()
    conn.row_factory = sqlite3.Row
    cursor = conn.execute(sql)
    return [dict(row) for row in cursor.fetchall()]

@mcp.tool()
def list_tables() -> list[str]:
    """List all tables"""
    conn = get_conn()
    cursor = conn.execute(
        "SELECT name FROM sqlite_master WHERE type='table'"
    )
    return [row[0] for row in cursor.fetchall()]

@mcp.tool()
def describe_table(table_name: str) -> list[dict]:
    """Describe table schema"""
    conn = get_conn()
    cursor = conn.execute(f"PRAGMA table_info({table_name})")
    return [
        {"name": r[1], "type": r[2], "notnull": bool(r[3])}
        for r in cursor.fetchall()
    ]

@mcp.resource("sqlite://schema")
def get_schema() -> str:
    """Database schema"""
    conn = get_conn()
    cursor = conn.execute(
        "SELECT sql FROM sqlite_master WHERE type='table'"
    )
    return "\n\n".join(row[0] for row in cursor.fetchall())

@mcp.prompt()
def query_helper(question: str) -> str:
    """Generate SQL query from question"""
    return f"""Generate SQL query for:

{question}

Available tables: use `list_tables` tool to discover.
Use `describe_table` to see columns."""

if __name__ == "__main__":
    mcp.run()
```

---

## 7. Demo: Filesystem Server (Safe)

```python
from mcp.server.fastmcp import FastMCP
from pathlib import Path
import os

mcp = FastMCP("filesystem-safe")

ALLOWED_ROOT = Path("/Users/me/Documents").resolve()

def safe_path(path: str) -> Path:
    """Validate path is under ALLOWED_ROOT"""
    p = (ALLOWED_ROOT / path).resolve()
    if not str(p).startswith(str(ALLOWED_ROOT)):
        raise ValueError(f"Path outside allowed: {p}")
    return p

@mcp.tool()
def read_file(path: str) -> str:
    """Read file content (relative to root)"""
    p = safe_path(path)
    if not p.exists():
        return f"File not found: {path}"
    return p.read_text()

@mcp.tool()
def write_file(path: str, content: str) -> str:
    """Write content to file"""
    p = safe_path(path)
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(content)
    return f"Wrote {len(content)} chars to {path}"

@mcp.tool()
def list_files(path: str = ".") -> list[dict]:
    """List files in directory"""
    p = safe_path(path)
    if not p.is_dir():
        return [{"error": "Not a directory"}]
    return [
        {
            "name": f.name,
            "type": "dir" if f.is_dir() else "file",
            "size": f.stat().st_size if f.is_file() else None,
        }
        for f in p.iterdir()
    ]

@mcp.tool()
def search_files(pattern: str, path: str = ".") -> list[str]:
    """Search files matching pattern"""
    p = safe_path(path)
    return [str(f.relative_to(ALLOWED_ROOT)) for f in p.rglob(pattern)]

if __name__ == "__main__":
    mcp.run()
```

---

## 8. Demo: REST API Server

```python
from mcp.server.fastmcp import FastMCP
import aiohttp

mcp = FastMCP("github-server")

GITHUB_TOKEN = os.environ.get("GITHUB_TOKEN")

async def github_request(method: str, path: str, json=None):
    headers = {
        "Authorization": f"Bearer {GITHUB_TOKEN}",
        "Accept": "application/vnd.github.v3+json",
    }
    url = f"https://api.github.com{path}"
    async with aiohttp.ClientSession() as session:
        async with session.request(method, url, headers=headers, json=json) as resp:
            return await resp.json()

@mcp.tool()
async def get_repo(owner: str, repo: str) -> dict:
    """Get repository info"""
    return await github_request("GET", f"/repos/{owner}/{repo}")

@mcp.tool()
async def list_issues(owner: str, repo: str, state: str = "open") -> list[dict]:
    """List issues"""
    result = await github_request("GET", f"/repos/{owner}/{repo}/issues?state={state}")
    return [{"number": i["number"], "title": i["title"], "state": i["state"]} for i in result]

@mcp.tool()
async def create_issue(
    owner: str, 
    repo: str, 
    title: str, 
    body: str = ""
) -> dict:
    """Create new issue"""
    return await github_request(
        "POST", 
        f"/repos/{owner}/{repo}/issues",
        json={"title": title, "body": body}
    )

if __name__ == "__main__":
    mcp.run()
```

---

## 9. Server Với State

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("stateful")

# State trong memory
counter = {"value": 0}
todos = []

@mcp.tool()
def increment() -> int:
    """Increment counter"""
    counter["value"] += 1
    return counter["value"]

@mcp.tool()
def get_counter() -> int:
    """Get counter value"""
    return counter["value"]

@mcp.tool()
def add_todo(text: str, priority: str = "medium") -> dict:
    """Add todo item"""
    todo = {"id": len(todos), "text": text, "priority": priority, "done": False}
    todos.append(todo)
    return todo

@mcp.tool()
def list_todos(done: bool = False) -> list[dict]:
    """List todos"""
    return [t for t in todos if t["done"] == done]

@mcp.tool()
def complete_todo(todo_id: int) -> dict:
    """Mark todo as done"""
    if 0 <= todo_id < len(todos):
        todos[todo_id]["done"] = True
        return todos[todo_id]
    return {"error": "Not found"}
```

---

## 10. Logging & Debugging

```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger("mcp-server")

@mcp.tool()
def my_tool(arg: str) -> str:
    logger.info(f"Called with: {arg}")
    result = process(arg)
    logger.info(f"Result: {result}")
    return result
```

### MCP Inspector

Tool để test server:
```bash
npx @modelcontextprotocol/inspector python server.py
```

Mở UI để:
- List tools/resources/prompts
- Call tools manually
- See requests/responses

---

## 11. Testing

```python
# test_server.py
import pytest
from mcp.client import Client
from mcp.client.stdio import stdio_client, StdioServerParameters

@pytest.mark.asyncio
async def test_add():
    params = StdioServerParameters(
        command="python",
        args=["server.py"],
    )
    
    async with stdio_client(params) as (read, write):
        async with Client(read, write) as client:
            await client.initialize()
            
            result = await client.call_tool("add", {"a": 5, "b": 3})
            assert "8" in result.content[0].text
```

---

## 12. Best Practices

✅ **Type hints**: bắt buộc cho mọi tool

✅ **Docstring đầy đủ**: LLM dùng để hiểu tool

✅ **Validate inputs**: dùng Pydantic

✅ **Limit scope**: server chỉ làm 1 việc

✅ **Error messages rõ ràng**: cho LLM debug

✅ **Read-only mặc định**: separate write tools

✅ **Async cho I/O**: file, network

✅ **Logging**: debug dễ hơn

---

## 13. Bài Tập

### Bài 1: Calculator Server
Build server với tools: add, subtract, multiply, divide, power, sqrt. Test với MCP Inspector.

### Bài 2: Notes Server
Server quản lý notes:
- create_note(title, content)
- list_notes()
- get_note(id)
- update_note(id, content)
- delete_note(id)
- search_notes(query)

### Bài 3: Weather Server
Wrap OpenWeather API:
- get_current(city)
- get_forecast(city, days)
- compare_cities(cities: list)

---

## 14. Checklist

- [ ] FastMCP setup
- [ ] Tools với type hints
- [ ] Pydantic schemas
- [ ] Async tools
- [ ] Resources (static + dynamic)
- [ ] Prompts
- [ ] Multiple transports
- [ ] State management
- [ ] Logging & MCP Inspector
- [ ] Testing

➡️ **Tiếp theo**: [Build MCP Client](./04-MCP-Client.md)
