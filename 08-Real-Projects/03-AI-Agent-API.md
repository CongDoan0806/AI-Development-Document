# Project 3: AI Agent Gọi REST API

## 1. Mục Tiêu

Build agent có thể:
- Hiểu OpenAPI spec
- Tự generate tools từ API endpoints
- Authenticate với API
- Chain multiple API calls
- Error handling + retry

**Use case**: GitHub manager, calendar bot, e-commerce assistant, ...

---

## 2. Architecture

```
[OpenAPI Spec] → [Auto-generate Tools]
                       ↓
User Query → [LangGraph Agent] → [Tools (HTTP)] → [REST API]
                       ↓
              [Format Response]
```

---

## 3. Approach 1: Manual Tools

```python
from langchain_core.tools import tool
import requests
import os

BASE_URL = "https://api.github.com"
TOKEN = os.environ["GITHUB_TOKEN"]

def gh_request(method, path, json=None, params=None):
    response = requests.request(
        method,
        f"{BASE_URL}{path}",
        headers={"Authorization": f"Bearer {TOKEN}"},
        json=json,
        params=params,
        timeout=10,
    )
    response.raise_for_status()
    return response.json()

@tool
def list_my_repos(per_page: int = 10) -> list:
    """List my GitHub repositories"""
    return gh_request("GET", "/user/repos", params={"per_page": per_page})

@tool
def get_repo(owner: str, repo: str) -> dict:
    """Get repository info"""
    return gh_request("GET", f"/repos/{owner}/{repo}")

@tool
def list_issues(owner: str, repo: str, state: str = "open") -> list:
    """List repository issues"""
    return gh_request("GET", f"/repos/{owner}/{repo}/issues", params={"state": state})

@tool
def create_issue(owner: str, repo: str, title: str, body: str = "") -> dict:
    """Create new issue"""
    return gh_request("POST", f"/repos/{owner}/{repo}/issues", 
                      json={"title": title, "body": body})

@tool
def add_comment(owner: str, repo: str, issue_number: int, comment: str) -> dict:
    """Add comment to issue"""
    return gh_request("POST", f"/repos/{owner}/{repo}/issues/{issue_number}/comments",
                      json={"body": comment})

tools = [list_my_repos, get_repo, list_issues, create_issue, add_comment]
```

---

## 4. Approach 2: Auto From OpenAPI

```python
from langchain_community.agent_toolkits.openapi import planner
from langchain_community.tools.openapi.utils.openapi_utils import OpenAPISpec
from langchain_community.agent_toolkits.openapi.spec import reduce_openapi_spec

# Load spec
import yaml
with open("github_openapi.yaml") as f:
    raw_spec = yaml.safe_load(f)

spec = reduce_openapi_spec(raw_spec)

# Build agent
agent = planner.create_openapi_agent(
    api_spec=spec,
    requests_wrapper=...,
    llm=llm,
    allow_dangerous_requests=True,
)

agent.invoke({"input": "List my repos and create an issue about updating README"})
```

---

## 5. Approach 3: Generic HTTP Tool

```python
@tool
def http_request(
    url: str,
    method: str = "GET",
    headers: dict = None,
    json_body: dict = None,
    params: dict = None,
) -> dict:
    """Make HTTP request to API.
    
    Args:
        url: Full URL
        method: GET, POST, PUT, DELETE, PATCH
        headers: Request headers (dict)
        json_body: JSON body for POST/PUT
        params: Query parameters
    """
    response = requests.request(
        method.upper(),
        url,
        headers=headers,
        json=json_body,
        params=params,
        timeout=10,
    )
    return {
        "status": response.status_code,
        "body": response.json() if response.text else None,
    }
```

⚠️ **Cẩn thận**: cho LLM access HTTP arbitrary rất rủi ro (SSRF). Cần whitelist domain.

---

## 6. Build LangGraph Agent

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

agent = create_react_agent(
    model=llm,
    tools=tools,
    state_modifier="""Bạn là GitHub Assistant.

Quy tắc:
- Dùng tools để query/modify GitHub
- Trước khi create issue, check existing issues
- Tạo title rõ ràng, body có context
- Confirm với user trước actions destructive
- Trả lời tiếng Việt"""
)

# Use
result = agent.invoke({
    "messages": [
        ("user", "Liệt kê 5 issues mới nhất của owner/repo và tạo 1 comment trên issue đầu tiên")
    ]
})

for m in result["messages"]:
    m.pretty_print()
```

---

## 7. Error Handling

```python
from langchain_core.tools import ToolException
import requests

@tool
def safe_api_call(endpoint: str) -> dict:
    """Call API with error handling"""
    try:
        response = requests.get(
            f"{BASE_URL}{endpoint}",
            headers={"Authorization": f"Bearer {TOKEN}"},
            timeout=10,
        )
        
        if response.status_code == 404:
            raise ToolException(f"Resource not found: {endpoint}")
        if response.status_code == 401:
            raise ToolException("Authentication failed")
        if response.status_code == 429:
            raise ToolException("Rate limited - try again later")
        if response.status_code >= 500:
            raise ToolException(f"Server error: {response.status_code}")
        
        response.raise_for_status()
        return response.json()
    
    except requests.Timeout:
        raise ToolException("Request timed out")
    except requests.ConnectionError:
        raise ToolException("Cannot connect to API")
```

---

## 8. Pagination Helper

```python
@tool
def list_all_issues(owner: str, repo: str, state: str = "open") -> list:
    """List ALL issues (auto-paginate)"""
    all_issues = []
    page = 1
    
    while True:
        result = gh_request(
            "GET",
            f"/repos/{owner}/{repo}/issues",
            params={"state": state, "page": page, "per_page": 100},
        )
        if not result:
            break
        all_issues.extend(result)
        page += 1
        if len(result) < 100:
            break
    
    return all_issues
```

---

## 9. Confirmation For Destructive Actions

```python
from langgraph.types import interrupt

def make_confirmable(tool_func, action_description):
    """Wrap tool to require user confirmation"""
    @tool
    def confirmable(*args, **kwargs):
        # Pause for confirmation
        response = interrupt({
            "action": action_description,
            "args": kwargs,
            "tool": tool_func.name,
        })
        if not response.get("approved"):
            return f"Cancelled by user: {action_description}"
        return tool_func.invoke(kwargs)
    return confirmable

safe_create_issue = make_confirmable(create_issue, "Create GitHub issue")
safe_delete_repo = make_confirmable(delete_repo, "DELETE GitHub repo")
```

---

## 10. Demo: Multi-API Agent

```python
import os
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
import requests

# GitHub tools
@tool
def gh_list_repos() -> list:
    """List my GitHub repos"""
    return requests.get(
        "https://api.github.com/user/repos",
        headers={"Authorization": f"Bearer {os.environ['GITHUB_TOKEN']}"}
    ).json()[:5]

@tool
def gh_create_issue(owner: str, repo: str, title: str, body: str) -> dict:
    """Create GitHub issue"""
    return requests.post(
        f"https://api.github.com/repos/{owner}/{repo}/issues",
        headers={"Authorization": f"Bearer {os.environ['GITHUB_TOKEN']}"},
        json={"title": title, "body": body},
    ).json()

# Linear tools
@tool
def linear_list_issues() -> list:
    """List Linear issues"""
    response = requests.post(
        "https://api.linear.app/graphql",
        headers={"Authorization": os.environ["LINEAR_TOKEN"]},
        json={"query": "{ issues(first: 10) { nodes { id title state { name } } } }"}
    )
    return response.json()["data"]["issues"]["nodes"]

@tool
def linear_create_issue(title: str, description: str = "") -> dict:
    """Create Linear issue"""
    response = requests.post(
        "https://api.linear.app/graphql",
        headers={"Authorization": os.environ["LINEAR_TOKEN"]},
        json={
            "query": """
            mutation CreateIssue($title: String!, $desc: String) {
                issueCreate(input: {title: $title, description: $desc, teamId: "..."}) {
                    issue { id title }
                }
            }
            """,
            "variables": {"title": title, "desc": description}
        }
    )
    return response.json()

# Slack tools
@tool
def slack_post_message(channel: str, message: str) -> dict:
    """Send Slack message"""
    return requests.post(
        "https://slack.com/api/chat.postMessage",
        headers={"Authorization": f"Bearer {os.environ['SLACK_TOKEN']}"},
        json={"channel": channel, "text": message},
    ).json()

# Build agent
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

agent = create_react_agent(
    model=llm,
    tools=[
        gh_list_repos,
        gh_create_issue,
        linear_list_issues,
        linear_create_issue,
        slack_post_message,
    ],
    state_modifier="""Bạn là DevOps assistant. Coordinate giữa GitHub, Linear, Slack.

Workflows:
- Khi user báo bug → tạo GitHub issue + Linear ticket + notify Slack
- Daily standup → fetch issues từ Linear, post summary to Slack
- PR review → check GitHub, update Linear status"""
)

# Test
result = agent.invoke({"messages": [
    ("user", "Có bug: login API trả 500 error. Tạo GitHub issue ở owner/api repo, Linear ticket, và notify #engineering")
]})

for m in result["messages"]:
    m.pretty_print()
```

---

## 11. Caching API Responses

```python
from functools import lru_cache
import time

@lru_cache(maxsize=100)
def cached_gh_request(method, path, cache_ts):
    """Cache for 60s (cache_ts is rounded to minute)"""
    return gh_request(method, path)

@tool
def get_repo_cached(owner: str, repo: str) -> dict:
    """Get repo info (cached 60s)"""
    cache_ts = int(time.time() // 60)
    return cached_gh_request("GET", f"/repos/{owner}/{repo}", cache_ts)
```

---

## 12. OAuth Flow

For user-specific access:

```python
from fastapi import FastAPI, Request
from authlib.integrations.starlette_client import OAuth

oauth = OAuth()
oauth.register(
    name="github",
    client_id="...",
    client_secret="...",
    access_token_url="https://github.com/login/oauth/access_token",
    authorize_url="https://github.com/login/oauth/authorize",
    api_base_url="https://api.github.com/",
)

@app.get("/login")
async def login(request: Request):
    return await oauth.github.authorize_redirect(request, "http://localhost/callback")

@app.get("/callback")
async def callback(request: Request):
    token = await oauth.github.authorize_access_token(request)
    # Store token per user
    user_tokens[user_id] = token["access_token"]
```

---

## 13. Monitoring API Calls

```python
import logging
from datetime import datetime

api_log = logging.getLogger("api-calls")

def log_api_call(method, url, status, latency_ms):
    api_log.info("API call", extra={
        "method": method,
        "url": url,
        "status": status,
        "latency_ms": latency_ms,
        "timestamp": datetime.now().isoformat(),
    })

@tool
def tracked_api_call(method: str, url: str) -> dict:
    start = time.time()
    response = requests.request(method, url, headers=HEADERS)
    elapsed = (time.time() - start) * 1000
    log_api_call(method, url, response.status_code, elapsed)
    return response.json()
```

---

## 14. Bài Tập Mở Rộng

### Bài 1: E-Commerce Assistant
Tools: search products, check stock, create order, track shipment.

### Bài 2: Calendar Manager
Tools: list events, create event, find free time, send invite.

### Bài 3: Multi-Cloud Manager
Tools cho AWS, GCP, Azure: list resources, check costs.

---

## 15. Checklist

- [ ] Manual API tools
- [ ] Auto từ OpenAPI spec
- [ ] Error handling robust
- [ ] Pagination
- [ ] Confirmation cho destructive
- [ ] Multi-API coordination
- [ ] OAuth flow
- [ ] Monitoring

➡️ **Tiếp theo**: [Multi-Agent Workflow](./04-Multi-Agent-Workflow.md)
