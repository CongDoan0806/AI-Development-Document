# Project 6: Code Review Agent

## 1. Mục Tiêu

Build agent tự động code review:
- Đọc git diff
- Phân tích code quality
- Detect bugs, security issues
- Suggest improvements
- Post comments on GitHub PR

**Tech**: LangGraph, GitHub API, AST parsing, LangChain.

---

## 2. Architecture

```
GitHub PR → [Fetch diff] → [Analyze files]
                                 ↓
                    [Parallel Reviewers]
                    ├── Security Reviewer
                    ├── Performance Reviewer
                    ├── Style Reviewer
                    └── Bug Detector
                                 ↓
                    [Aggregate findings]
                                 ↓
                    [Post to GitHub]
```

---

## 3. Setup

```bash
pip install langchain langchain-openai langgraph
pip install PyGithub
pip install tree-sitter tree-sitter-languages  # AST
```

```env
GITHUB_TOKEN=ghp_...
OPENAI_API_KEY=sk-...
```

---

## 4. GitHub Integration

```python
from github import Github
import os

g = Github(os.environ["GITHUB_TOKEN"])

def get_pr_files(owner: str, repo: str, pr_number: int):
    """Get files changed in PR"""
    repo_obj = g.get_repo(f"{owner}/{repo}")
    pr = repo_obj.get_pull(pr_number)
    
    files = []
    for f in pr.get_files():
        files.append({
            "filename": f.filename,
            "status": f.status,  # added, modified, removed
            "patch": f.patch,    # The diff
            "additions": f.additions,
            "deletions": f.deletions,
        })
    return files, pr

def post_comment(pr, body, file_path=None, line=None):
    """Post comment on PR"""
    if file_path and line:
        # Inline comment
        pr.create_review_comment(
            body=body,
            commit_id=pr.head.sha,
            path=file_path,
            line=line,
        )
    else:
        # General comment
        pr.create_issue_comment(body)
```

---

## 5. State Schema

```python
from typing import TypedDict, List, Annotated
from operator import add

class CodeReviewState(TypedDict):
    pr_url: str
    files: List[dict]
    findings: Annotated[List[dict], add]  # Append from multiple reviewers
    summary: str
```

---

## 6. Reviewers (Specialists)

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel
from typing import List, Literal

llm = ChatOpenAI(model="gpt-4o", temperature=0)

class Finding(BaseModel):
    file: str
    line: int = 0
    severity: Literal["critical", "high", "medium", "low", "info"]
    category: str
    issue: str
    suggestion: str

class ReviewResult(BaseModel):
    findings: List[Finding]

structured_llm = llm.with_structured_output(ReviewResult)

# Security Reviewer
security_prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là security expert. Review code tìm:
- SQL injection
- XSS
- Hardcoded secrets
- Insecure auth
- Path traversal
- Command injection
- Sensitive data leakage"""),
    ("user", "File: {file}\n\nDiff:\n{diff}\n\nFindings (JSON):"),
])

def security_reviewer(state):
    findings = []
    for file in state["files"]:
        if file["status"] == "removed":
            continue
        result = structured_llm.invoke(security_prompt.format(
            file=file["filename"],
            diff=file["patch"],
        ))
        for f in result.findings:
            f.file = file["filename"]
            findings.append(f.dict())
    return {"findings": findings}

# Performance Reviewer
perf_prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là performance expert. Tìm:
- N+1 queries
- Unnecessary loops
- Memory leaks
- Inefficient algorithms
- Blocking I/O in async code
- Missing indexes (SQL)"""),
    ("user", "File: {file}\n\nDiff:\n{diff}"),
])

def performance_reviewer(state):
    findings = []
    for file in state["files"]:
        # Skip non-code files
        if not file["filename"].endswith((".py", ".js", ".ts", ".go", ".java")):
            continue
        result = structured_llm.invoke(perf_prompt.format(
            file=file["filename"],
            diff=file["patch"],
        ))
        for f in result.findings:
            f.file = file["filename"]
            findings.append(f.dict())
    return {"findings": findings}

# Style Reviewer
style_prompt = ChatPromptTemplate.from_messages([
    ("system", """Code style reviewer. Check:
- Naming conventions
- Code organization
- Documentation
- DRY violations
- Magic numbers
- Code complexity"""),
    ("user", "File: {file}\n\nDiff:\n{diff}"),
])

def style_reviewer(state):
    findings = []
    for file in state["files"]:
        result = structured_llm.invoke(style_prompt.format(
            file=file["filename"],
            diff=file["patch"],
        ))
        for f in result.findings:
            f.file = file["filename"]
            findings.append(f.dict())
    return {"findings": findings}

# Bug Detector
bug_prompt = ChatPromptTemplate.from_messages([
    ("system", """Find bugs:
- Off-by-one errors
- Null pointer / None handling
- Type errors
- Logic errors
- Race conditions
- Resource leaks
- Edge cases not handled"""),
    ("user", "File: {file}\n\nDiff:\n{diff}"),
])

def bug_detector(state):
    findings = []
    for file in state["files"]:
        result = structured_llm.invoke(bug_prompt.format(
            file=file["filename"],
            diff=file["patch"],
        ))
        for f in result.findings:
            f.file = file["filename"]
            findings.append(f.dict())
    return {"findings": findings}
```

---

## 7. Aggregator

```python
def summarizer(state):
    findings = state["findings"]
    
    if not findings:
        summary = "✅ No issues found! LGTM."
    else:
        # Group by severity
        by_severity = {}
        for f in findings:
            by_severity.setdefault(f["severity"], []).append(f)
        
        # Build summary
        sections = []
        for severity in ["critical", "high", "medium", "low", "info"]:
            items = by_severity.get(severity, [])
            if not items:
                continue
            
            emoji = {"critical": "🔴", "high": "🟠", "medium": "🟡", "low": "🔵", "info": "ℹ️"}[severity]
            sections.append(f"### {emoji} {severity.upper()} ({len(items)})")
            
            for item in items:
                sections.append(f"**{item['file']}{':L'+str(item['line']) if item['line'] else ''}** - {item['category']}")
                sections.append(f"- Issue: {item['issue']}")
                sections.append(f"- Suggestion: {item['suggestion']}")
        
        summary = "## 🤖 AI Code Review\n\n" + "\n".join(sections)
        summary += f"\n\n**Total: {len(findings)} findings**"
    
    return {"summary": summary}
```

---

## 8. Build Graph

```python
from langgraph.graph import StateGraph, START, END

graph = StateGraph(CodeReviewState)

# Parallel reviewers
graph.add_node("security", security_reviewer)
graph.add_node("performance", performance_reviewer)
graph.add_node("style", style_reviewer)
graph.add_node("bugs", bug_detector)
graph.add_node("summarize", summarizer)

# All reviewers run in parallel from START
graph.add_edge(START, "security")
graph.add_edge(START, "performance")
graph.add_edge(START, "style")
graph.add_edge(START, "bugs")

# All converge to summarize
graph.add_edge("security", "summarize")
graph.add_edge("performance", "summarize")
graph.add_edge("style", "summarize")
graph.add_edge("bugs", "summarize")

graph.add_edge("summarize", END)

reviewer_graph = graph.compile()
```

---

## 9. Main Review Function

```python
def review_pr(owner: str, repo: str, pr_number: int, post_to_github=True):
    # 1. Fetch PR
    files, pr = get_pr_files(owner, repo, pr_number)
    
    # 2. Run reviewers
    result = reviewer_graph.invoke({
        "pr_url": pr.html_url,
        "files": files,
        "findings": [],
    })
    
    # 3. Post comments
    if post_to_github:
        # General summary
        pr.create_issue_comment(result["summary"])
        
        # Inline comments cho critical/high
        for f in result["findings"]:
            if f["severity"] in ["critical", "high"] and f["line"]:
                try:
                    pr.create_review_comment(
                        body=f"**{f['category']}**: {f['issue']}\n\n💡 {f['suggestion']}",
                        commit_id=pr.head.sha,
                        path=f["file"],
                        line=f["line"],
                    )
                except Exception as e:
                    print(f"Cannot post inline: {e}")
    
    return result

# Use
result = review_pr("owner", "repo", 123)
print(result["summary"])
```

---

## 10. GitHub Actions Integration

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-python@v4
        with:
          python-version: "3.11"
      
      - name: Install
        run: pip install -r requirements.txt
      
      - name: Run AI Review
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python review.py \
            --owner ${{ github.repository_owner }} \
            --repo ${{ github.event.repository.name }} \
            --pr ${{ github.event.pull_request.number }}
```

`review.py`:
```python
import argparse
from reviewer import review_pr

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--owner", required=True)
    parser.add_argument("--repo", required=True)
    parser.add_argument("--pr", type=int, required=True)
    args = parser.parse_args()
    
    review_pr(args.owner, args.repo, args.pr)
```

---

## 11. AST-Based Analysis (Advanced)

Cho Python:

```python
import ast

def analyze_python_complexity(code: str) -> dict:
    """Analyze Python code complexity"""
    try:
        tree = ast.parse(code)
    except SyntaxError as e:
        return {"error": str(e)}
    
    metrics = {
        "functions": 0,
        "classes": 0,
        "max_function_lines": 0,
        "has_global_state": False,
    }
    
    for node in ast.walk(tree):
        if isinstance(node, ast.FunctionDef):
            metrics["functions"] += 1
            lines = node.end_lineno - node.lineno
            metrics["max_function_lines"] = max(metrics["max_function_lines"], lines)
        elif isinstance(node, ast.ClassDef):
            metrics["classes"] += 1
        elif isinstance(node, ast.Global):
            metrics["has_global_state"] = True
    
    return metrics

@tool
def python_metrics(code: str) -> dict:
    """Get Python code metrics"""
    return analyze_python_complexity(code)
```

---

## 12. Custom Rules

```python
import re

class CustomRule:
    def check(self, code: str, filename: str) -> List[Finding]:
        raise NotImplementedError

class NoTodoRule(CustomRule):
    def check(self, code, filename):
        findings = []
        for i, line in enumerate(code.split("\n"), 1):
            if re.search(r'\b(TODO|FIXME|XXX)\b', line):
                findings.append(Finding(
                    file=filename,
                    line=i,
                    severity="low",
                    category="todo",
                    issue=f"TODO comment: {line.strip()}",
                    suggestion="Create issue and reference it",
                ))
        return findings

class NoConsoleLogRule(CustomRule):
    """For JS/TS files"""
    def check(self, code, filename):
        if not filename.endswith((".js", ".ts")):
            return []
        findings = []
        for i, line in enumerate(code.split("\n"), 1):
            if "console.log" in line:
                findings.append(Finding(
                    file=filename,
                    line=i,
                    severity="medium",
                    category="logging",
                    issue="console.log left in code",
                    suggestion="Remove or use proper logger",
                ))
        return findings

custom_rules = [NoTodoRule(), NoConsoleLogRule()]

def custom_rules_reviewer(state):
    findings = []
    for file in state["files"]:
        if file["status"] == "removed":
            continue
        for rule in custom_rules:
            findings.extend([f.dict() for f in rule.check(file["patch"], file["filename"])])
    return {"findings": findings}
```

---

## 13. Cost Optimization

```python
# Skip files khi không cần review
def filter_files(files):
    skip_patterns = [
        r'\.md$',           # Docs
        r'\.json$',         # Config
        r'\.lock$',         # Lock files
        r'\.min\.',         # Minified
        r'node_modules/',
        r'__pycache__/',
        r'\.test\.',        # Test files (lighter review)
    ]
    return [
        f for f in files
        if not any(re.search(p, f["filename"]) for p in skip_patterns)
        and f["additions"] + f["deletions"] < 500  # Skip huge diffs
    ]

# Use cheaper model cho large PRs
def get_llm_for_pr(num_files):
    if num_files > 20:
        return ChatOpenAI(model="gpt-4o-mini")  # Cheap
    return ChatOpenAI(model="gpt-4o")  # Smart
```

---

## 14. Demo Output

```markdown
## 🤖 AI Code Review

### 🔴 CRITICAL (1)
**src/auth.py:45** - Security
- Issue: SQL injection vulnerability - user input concatenated directly
- Suggestion: Use parameterized queries: `cursor.execute("... WHERE id = ?", [user_id])`

### 🟠 HIGH (2)
**src/api.py:120** - Bug
- Issue: NoneType error possible if user not found
- Suggestion: Add null check before accessing user.email

**src/db.py:88** - Performance
- Issue: N+1 query in loop
- Suggestion: Use `JOIN` or batch fetch

### 🟡 MEDIUM (3)
...

### ℹ️ INFO (5)
...

**Total: 11 findings**
```

---

## 15. Bài Tập

### Bài 1: Multi-Language Support
Add reviewers cho Go, Rust, Java.

### Bài 2: Auto-Fix
Generate code fix suggestions, post as PR suggestions.

### Bài 3: Trend Analysis
Track findings over time per repo, identify common issues.

### Bài 4: Slack Notifications
Notify channel khi review xong, mention author.

---

## 16. Checklist

- [ ] GitHub PR integration
- [ ] Multi-specialist reviewers
- [ ] Parallel execution với LangGraph
- [ ] Structured findings (Pydantic)
- [ ] Post inline comments
- [ ] GitHub Actions integration
- [ ] AST analysis
- [ ] Custom rules
- [ ] Cost optimization

🎉 **Hoàn thành Phase 8 - Real Projects!**

🎉🎉🎉 **HOÀN THÀNH TOÀN BỘ LỘ TRÌNH!** 🎉🎉🎉

---

## Tổng Kết Lộ Trình

Bạn đã học:
- ✅ Phase 0: Python nâng cao, LLM concepts, Embeddings
- ✅ Phase 1: LangChain core (LCEL, Models, Prompts, Parsers)
- ✅ Phase 2: RAG end-to-end (10 chủ đề)
- ✅ Phase 3: Agents (Tool calling, Planning, Memory, Error handling)
- ✅ Phase 4: LangGraph (State machines, Multi-agent, Persistence)
- ✅ Phase 5: MCP (Model Context Protocol)
- ✅ Phase 6: Evaluation (RAG, Prompts, Hallucination)
- ✅ Phase 7: Production (Caching, Cost, Streaming, Deployment)
- ✅ Phase 8: 6 Real Projects

## Bước Tiếp Theo

1. **Build project lớn** áp dụng tất cả
2. **Contribute** to LangChain/LangGraph open source
3. **Stay updated**: theo dõi LangChain blog, Anthropic, OpenAI
4. **Community**: join Discord LangChain, MCP

**Chúc bạn thành công với AI! 🚀**
