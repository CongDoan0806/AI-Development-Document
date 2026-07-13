# Retry & Fallback Flow

## 1. Retry Pattern Với Counter

```python
from typing import TypedDict
from langgraph.graph import StateGraph, END

class State(TypedDict):
    input: str
    output: str
    attempts: int
    max_attempts: int
    error: str

def try_action(state):
    try:
        result = risky_operation(state["input"])
        return {"output": result, "error": ""}
    except Exception as e:
        return {
            "attempts": state["attempts"] + 1,
            "error": str(e),
        }

def should_retry(state):
    if state.get("output"):
        return END  # Success
    if state["attempts"] >= state["max_attempts"]:
        return "give_up"
    return "try_action"  # Retry

def give_up(state):
    return {"output": f"Failed after {state['attempts']} attempts: {state['error']}"}

graph = StateGraph(State)
graph.add_node("try_action", try_action)
graph.add_node("give_up", give_up)

graph.set_entry_point("try_action")
graph.add_conditional_edges("try_action", should_retry, {
    "try_action": "try_action",
    "give_up": "give_up",
    END: END,
})
graph.add_edge("give_up", END)

app = graph.compile()
```

---

## 2. Error Node Pattern

Riêng 1 node để handle errors:

```python
class State(TypedDict):
    input: str
    output: str
    error: str
    retry_count: int

def main_action(state):
    try:
        result = do_something(state["input"])
        return {"output": result}
    except Exception as e:
        return {"error": str(e)}

def error_handler(state):
    error = state["error"]
    if "rate_limit" in error.lower():
        import time
        time.sleep(5)
        return {"error": "", "retry_count": state["retry_count"] + 1}
    elif "timeout" in error.lower():
        return {"error": "", "retry_count": state["retry_count"] + 1}
    else:
        return {"output": f"Unrecoverable: {error}"}

def route(state):
    if state.get("output"):
        return END
    if state.get("error"):
        return "error_handler"
    return END

def route_after_error(state):
    if state["retry_count"] >= 3:
        return END
    if state.get("error"):
        return END  # Unrecoverable
    return "main_action"

graph.add_node("main_action", main_action)
graph.add_node("error_handler", error_handler)

graph.add_conditional_edges("main_action", route, {...})
graph.add_conditional_edges("error_handler", route_after_error, {...})
```

---

## 3. Fallback Chain

Try primary → secondary → tertiary:

```python
class State(TypedDict):
    input: str
    output: str
    tried: list

def try_premium_model(state):
    try:
        result = premium_llm.invoke(state["input"])
        return {"output": result.content}
    except:
        return {"tried": state["tried"] + ["premium"]}

def try_standard_model(state):
    try:
        result = standard_llm.invoke(state["input"])
        return {"output": result.content}
    except:
        return {"tried": state["tried"] + ["standard"]}

def try_cheap_model(state):
    try:
        result = cheap_llm.invoke(state["input"])
        return {"output": result.content}
    except:
        return {"tried": state["tried"] + ["cheap"], "output": "All models failed"}

def route(state):
    if state.get("output"):
        return END
    last_tried = state["tried"][-1] if state["tried"] else None
    if last_tried == "premium":
        return "standard"
    elif last_tried == "standard":
        return "cheap"
    return END

graph.add_node("premium", try_premium_model)
graph.add_node("standard", try_standard_model)
graph.add_node("cheap", try_cheap_model)

graph.set_entry_point("premium")
graph.add_conditional_edges("premium", route, {
    "standard": "standard", END: END
})
graph.add_conditional_edges("standard", route, {
    "cheap": "cheap", END: END
})
graph.add_edge("cheap", END)
```

---

## 4. Self-Correction Loop

Validate output, retry với feedback:

```python
class State(TypedDict):
    question: str
    answer: str
    valid: bool
    feedback: str
    iterations: int

def generate(state):
    prompt = state["question"]
    if state.get("feedback"):
        prompt += f"\n\nPrevious attempt feedback: {state['feedback']}"
    
    answer = llm.invoke(prompt).content
    return {"answer": answer, "iterations": state["iterations"] + 1}

def validate(state):
    # LLM judge
    verdict = llm.invoke(f"""
    Đánh giá answer: 
    Q: {state["question"]}
    A: {state["answer"]}
    
    Trả về JSON: {{"valid": bool, "feedback": "..."}}
    """).content
    
    import json
    data = json.loads(verdict)
    return {"valid": data["valid"], "feedback": data["feedback"]}

def route(state):
    if state["valid"] or state["iterations"] >= 3:
        return END
    return "generate"

graph.add_node("generate", generate)
graph.add_node("validate", validate)

graph.set_entry_point("generate")
graph.add_edge("generate", "validate")
graph.add_conditional_edges("validate", route, {
    "generate": "generate",
    END: END,
})
```

---

## 5. CRAG - Corrective RAG

Khi retrieval kém → tự correct:

```python
class CRAGState(TypedDict):
    question: str
    documents: list
    relevance: str  # "yes", "no", "partial"
    web_search_results: list
    answer: str

def retrieve(state):
    docs = vectorstore.similarity_search(state["question"], k=5)
    return {"documents": docs}

def grade_documents(state):
    # Đánh giá độ liên quan
    relevant = 0
    for doc in state["documents"]:
        score = grade_doc(state["question"], doc)
        if score > 0.7:
            relevant += 1
    
    if relevant >= 3:
        return {"relevance": "yes"}
    elif relevant >= 1:
        return {"relevance": "partial"}
    return {"relevance": "no"}

def web_search(state):
    # Fallback: search web
    results = tavily_search.invoke(state["question"])
    return {"web_search_results": results}

def generate(state):
    docs = state["documents"]
    if state.get("web_search_results"):
        docs = docs + state["web_search_results"]
    
    answer = llm.invoke(f"Context: {docs}\nQ: {state['question']}").content
    return {"answer": answer}

def route_after_grade(state):
    if state["relevance"] == "yes":
        return "generate"
    return "web_search"  # Fallback web

graph = StateGraph(CRAGState)
graph.add_node("retrieve", retrieve)
graph.add_node("grade", grade_documents)
graph.add_node("web_search", web_search)
graph.add_node("generate", generate)

graph.set_entry_point("retrieve")
graph.add_edge("retrieve", "grade")
graph.add_conditional_edges("grade", route_after_grade, {
    "generate": "generate",
    "web_search": "web_search",
})
graph.add_edge("web_search", "generate")
graph.add_edge("generate", END)
```

---

## 6. Built-in Retry

```python
from langgraph.pregel.retry import RetryPolicy

def flaky_node(state):
    # Có thể fail
    ...

graph.add_node(
    "flaky",
    flaky_node,
    retry=RetryPolicy(
        max_attempts=3,
        backoff_factor=2.0,  # Exponential backoff
        retry_on=Exception,
    ),
)
```

---

## 7. Demo: Self-Correcting Code Agent

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

llm = ChatOpenAI(model="gpt-4o-mini")

class CodeState(TypedDict):
    requirements: str
    code: str
    test_result: str
    iterations: int
    final_code: str

def generate_code(state):
    prompt = f"""
    Viết Python code theo yêu cầu:
    {state["requirements"]}
    
    {'Previous attempt feedback: ' + state.get('test_result', '') if state.get('test_result') else ''}
    
    Chỉ trả về code, không giải thích.
    """
    code = llm.invoke(prompt).content
    # Strip markdown
    code = code.replace("```python", "").replace("```", "").strip()
    return {"code": code, "iterations": state.get("iterations", 0) + 1}

def test_code(state):
    try:
        # Thực thi code trong sandbox
        exec_globals = {}
        exec(state["code"], exec_globals)
        return {"test_result": "PASS"}
    except Exception as e:
        return {"test_result": f"FAIL: {str(e)}"}

def route(state):
    if state["test_result"] == "PASS":
        return "finalize"
    if state["iterations"] >= 3:
        return "give_up"
    return "generate"

def finalize(state):
    return {"final_code": state["code"]}

def give_up(state):
    return {"final_code": f"Could not write working code after 3 attempts. Last error: {state['test_result']}"}

graph = StateGraph(CodeState)
graph.add_node("generate", generate_code)
graph.add_node("test", test_code)
graph.add_node("finalize", finalize)
graph.add_node("give_up", give_up)

graph.set_entry_point("generate")
graph.add_edge("generate", "test")
graph.add_conditional_edges("test", route, {
    "generate": "generate",
    "finalize": "finalize",
    "give_up": "give_up",
})
graph.add_edge("finalize", END)
graph.add_edge("give_up", END)

app = graph.compile()

result = app.invoke({
    "requirements": "Function fibonacci(n) trả về số fibonacci thứ n",
    "iterations": 0,
})
print(result["final_code"])
```

---

## 8. Checklist

- [ ] Retry pattern với counter
- [ ] Error handler node
- [ ] Fallback chain
- [ ] Self-correction loop
- [ ] CRAG pattern
- [ ] Built-in RetryPolicy

➡️ **Tiếp theo**: [Human-in-the-Loop](./05-Human-in-the-Loop.md)
