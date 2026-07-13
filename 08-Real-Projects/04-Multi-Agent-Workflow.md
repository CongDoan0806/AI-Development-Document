# Project 4: Multi-Agent Research Workflow

## 1. Mục Tiêu

Build hệ thống multi-agent:
- 🔍 **Researcher**: search web, gather info
- ✍️ **Writer**: draft content
- 📝 **Editor**: review, suggest improvements
- 🎯 **Supervisor**: điều phối

Workflow: research → write → edit → publish

---

## 2. Architecture (LangGraph)

```
              ┌─────────────┐
              │ Supervisor  │
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │Researcher│ │  Writer  │ │  Editor  │
  └──────────┘ └──────────┘ └──────────┘
        ↓            ↓            ↓
        └────── Shared State ─────┘
```

---

## 3. State

```python
from typing import TypedDict, Annotated, List, Literal
from langgraph.graph.message import add_messages

class WorkflowState(TypedDict):
    messages: Annotated[list, add_messages]
    topic: str
    research_notes: str
    draft: str
    final_article: str
    edit_iterations: int
    next_agent: str
```

---

## 4. Agents

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_community.tools import TavilySearchResults

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.3)

# Researcher tools
search = TavilySearchResults(max_results=5)

def researcher_node(state):
    topic = state["topic"]
    
    # Search
    results = search.invoke(f"{topic} latest 2024")
    
    # Synthesize notes
    notes_prompt = f"""Tổng hợp research notes từ các nguồn sau về chủ đề: {topic}

Sources:
{results}

Output: notes có cấu trúc với:
- Key facts
- Quotes (cite source URL)
- Statistics
- Trends
"""
    notes = llm.invoke(notes_prompt).content
    
    return {
        "research_notes": notes,
        "messages": [("ai", f"📚 Research completed for: {topic}")],
        "next_agent": "writer",
    }

def writer_node(state):
    write_prompt = f"""Viết bài blog ~500 từ về:
Topic: {state["topic"]}

Research notes:
{state["research_notes"]}

Yêu cầu:
- Engaging intro
- 3 main points
- Conclusion với takeaway
- Tone: professional, accessible
"""
    draft = llm.invoke(write_prompt).content
    
    return {
        "draft": draft,
        "messages": [("ai", f"✍️ Draft completed ({len(draft)} chars)")],
        "next_agent": "editor",
    }

def editor_node(state):
    iterations = state.get("edit_iterations", 0)
    
    edit_prompt = f"""Edit bài viết sau:
{state["draft"]}

Check:
1. Grammar, spelling
2. Flow, structure
3. Clarity
4. Engaging

Nếu OK → trả về CHỈ JSON: {{"status": "approved", "final": "<final text>"}}
Nếu cần fix → JSON: {{"status": "needs_revision", "feedback": "<specific issues>"}}
"""
    response = llm.invoke(edit_prompt).content
    
    import json
    try:
        result = json.loads(response.strip())
        if result["status"] == "approved":
            return {
                "final_article": result["final"],
                "messages": [("ai", "✅ Edit approved")],
                "next_agent": "FINISH",
            }
        else:
            return {
                "messages": [("ai", f"🔄 Needs revision: {result['feedback']}")],
                "next_agent": "writer",
                "edit_iterations": iterations + 1,
            }
    except:
        return {"final_article": state["draft"], "next_agent": "FINISH"}

def supervisor_node(state):
    """Decide next agent based on state"""
    iterations = state.get("edit_iterations", 0)
    
    if not state.get("research_notes"):
        return {"next_agent": "researcher"}
    elif not state.get("draft"):
        return {"next_agent": "writer"}
    elif not state.get("final_article") and iterations < 3:
        return {"next_agent": "editor"}
    return {"next_agent": "FINISH"}
```

---

## 5. Build Graph

```python
from langgraph.graph import StateGraph, END

def route(state):
    return state["next_agent"]

graph = StateGraph(WorkflowState)

graph.add_node("supervisor", supervisor_node)
graph.add_node("researcher", researcher_node)
graph.add_node("writer", writer_node)
graph.add_node("editor", editor_node)

graph.set_entry_point("supervisor")

# Conditional routing
graph.add_conditional_edges(
    "supervisor",
    route,
    {
        "researcher": "researcher",
        "writer": "writer",
        "editor": "editor",
        "FINISH": END,
    }
)

# After each agent → back to supervisor
graph.add_edge("researcher", "supervisor")
graph.add_edge("writer", "supervisor")
graph.add_edge("editor", "supervisor")

app = graph.compile()
```

---

## 6. Run

```python
result = app.invoke({
    "topic": "Tác động của AI lên giáo dục",
    "messages": [],
    "edit_iterations": 0,
})

print("=" * 50)
print("FINAL ARTICLE")
print("=" * 50)
print(result["final_article"])

print("\n--- Workflow Steps ---")
for m in result["messages"]:
    print(m)
```

---

## 7. Streaming Progress

```python
async def run_with_progress(topic):
    async for event in app.astream(
        {"topic": topic, "messages": [], "edit_iterations": 0},
        stream_mode="updates",
    ):
        for node, output in event.items():
            print(f"\n[{node}]")
            if "messages" in output:
                for m in output["messages"]:
                    print(f"  {m}")
```

---

## 8. Add More Agents

### 8.1 Fact-Checker

```python
def fact_checker_node(state):
    """Verify claims trong draft"""
    check_prompt = f"""Kiểm tra các facts trong draft này:
{state['draft']}

Cho mỗi claim, verify với research_notes:
{state['research_notes']}

Output JSON: {{
  "verified_claims": [...],
  "unsupported_claims": [...],
  "verdict": "passed" | "needs_revision"
}}"""
    
    result = json.loads(llm.invoke(check_prompt).content)
    
    if result["verdict"] == "needs_revision":
        return {
            "next_agent": "writer",
            "messages": [("ai", f"⚠️ Unsupported: {result['unsupported_claims']}")],
        }
    return {"next_agent": "editor"}
```

### 8.2 SEO Optimizer

```python
def seo_node(state):
    seo_prompt = f"""Optimize cho SEO:
{state['draft']}

Add:
- Meta title (max 60 chars)
- Meta description (max 160 chars)
- Keywords list
- H2/H3 structure
- Internal/external links suggestions"""
    
    seo_data = llm.invoke(seo_prompt).content
    return {"seo_data": seo_data, "next_agent": "publisher"}
```

### 8.3 Publisher

```python
def publisher_node(state):
    """Publish to CMS"""
    # Call WordPress/Medium/Notion API
    response = requests.post(
        "https://api.wordpress.com/wp/v2/posts",
        json={
            "title": state["topic"],
            "content": state["final_article"],
            "status": "draft",
        },
        headers={"Authorization": f"Bearer {WP_TOKEN}"},
    )
    return {"published_url": response.json()["link"], "next_agent": "FINISH"}
```

---

## 9. Human-In-The-Loop

Pause để user review trước publish:

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(
    checkpointer=memory,
    interrupt_before=["publisher"],  # Pause trước publish
)

# Run
config = {"configurable": {"thread_id": "session1"}}
result = app.invoke({"topic": "..."}, config=config)

# Show draft to user
print(result["final_article"])
input("Press Enter to publish (or Ctrl-C to cancel)...")

# Continue
final = app.invoke(None, config=config)
print(f"Published at: {final['published_url']}")
```

---

## 10. Parallel Research

Multiple researchers parallel cho speed:

```python
from langgraph.constants import Send

def fan_out_research(state):
    """Tạo 3 sub-research tasks"""
    topic = state["topic"]
    sub_topics = [
        f"{topic} - historical",
        f"{topic} - current state",
        f"{topic} - future predictions",
    ]
    return [Send("researcher", {"sub_topic": t}) for t in sub_topics]

def sub_researcher(state):
    notes = llm.invoke(f"Research: {state['sub_topic']}").content
    return {"sub_notes": [notes]}

def merge_research(state):
    """Combine sub-notes"""
    combined = "\n\n".join(state["sub_notes"])
    return {"research_notes": combined}

# Update graph
graph.add_node("merge_research", merge_research)
graph.add_conditional_edges("supervisor", fan_out_research, ["researcher"])
graph.add_edge("researcher", "merge_research")
graph.add_edge("merge_research", "writer")
```

---

## 11. Demo: Hierarchical Teams

```python
# Sub-graph: Research team
research_graph = StateGraph(...)
research_graph.add_node("web_searcher", web_searcher)
research_graph.add_node("academic_searcher", academic_searcher)
research_graph.add_node("news_searcher", news_searcher)
research_graph.add_node("synthesizer", synthesizer)
research_subgraph = research_graph.compile()

# Sub-graph: Writing team
writing_graph = StateGraph(...)
writing_graph.add_node("outliner", outliner)
writing_graph.add_node("drafter", drafter)
writing_graph.add_node("polisher", polisher)
writing_subgraph = writing_graph.compile()

# Main graph
main_graph = StateGraph(...)
main_graph.add_node("research_team", research_subgraph)
main_graph.add_node("writing_team", writing_subgraph)
main_graph.add_edge("research_team", "writing_team")
```

---

## 12. Cost & Time Tracking

```python
import time
from langchain.callbacks import get_openai_callback

class CostTracker:
    def __init__(self):
        self.costs = {}
        self.times = {}
    
    def track(self, node_name):
        from contextlib import contextmanager
        
        @contextmanager
        def context():
            start = time.time()
            with get_openai_callback() as cb:
                yield cb
            self.costs[node_name] = cb.total_cost
            self.times[node_name] = time.time() - start
        
        return context()
    
    def report(self):
        total_cost = sum(self.costs.values())
        total_time = sum(self.times.values())
        print(f"\n=== Cost & Time Report ===")
        for node in self.costs:
            print(f"{node}: ${self.costs[node]:.4f} ({self.times[node]:.1f}s)")
        print(f"TOTAL: ${total_cost:.4f} ({total_time:.1f}s)")
```

---

## 13. UI Với Streamlit

```python
import streamlit as st

st.title("📝 AI Content Workflow")

topic = st.text_input("Topic")

if st.button("Generate Article"):
    progress = st.empty()
    article = st.empty()
    
    final = None
    for event in app.stream({"topic": topic, "messages": [], "edit_iterations": 0}):
        for node, output in event.items():
            progress.info(f"Running: {node}")
            if output.get("research_notes"):
                with st.expander("📚 Research"):
                    st.write(output["research_notes"])
            if output.get("draft"):
                with st.expander("✍️ Draft"):
                    st.write(output["draft"])
            if output.get("final_article"):
                final = output["final_article"]
    
    if final:
        article.success("✅ Done!")
        st.markdown("---")
        st.markdown("### Final Article")
        st.write(final)
```

---

## 14. Bài Tập Mở Rộng

### Bài 1: Code Generation Workflow
Agents: Analyst → Designer → Coder → Tester → Documenter.

### Bài 2: Market Research
Agents: Industry analyst, Competitor researcher, Customer insights, Report writer.

### Bài 3: Customer Support Escalation
Triage → Specialist → Manager → Resolution.

---

## 15. Checklist

- [ ] Multi-agent với LangGraph
- [ ] Supervisor pattern
- [ ] Conditional routing
- [ ] Human-in-the-loop
- [ ] Parallel agents
- [ ] Subgraph composition
- [ ] Cost tracking
- [ ] UI integration

➡️ **Tiếp theo**: [SQL Agent](./05-SQL-Agent.md)
