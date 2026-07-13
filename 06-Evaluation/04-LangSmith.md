# LangSmith Advanced

## 1. LangSmith Là Gì?

**LangSmith** = platform observability + evaluation cho LangChain/LangGraph apps.

**Features**:
- 🔍 **Tracing**: log mọi LLM call, tool call
- 📊 **Datasets**: golden set cho eval
- 🧪 **Experiments**: A/B test
- 📝 **Annotations**: human feedback
- 🛒 **Prompt Hub**: share prompts
- 📈 **Monitoring**: dashboards

URL: https://smith.langchain.com

---

## 2. Setup

```bash
pip install langsmith langchain
```

```env
LANGCHAIN_API_KEY=lsv2_...
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=my-project
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com  # Optional
```

→ Mọi LangChain call tự log lên.

---

## 3. Tracing

### 3.1 Automatic

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
response = llm.invoke("Hello")  # Auto traced
```

Mở dashboard để xem.

### 3.2 Manual Trace

```python
from langsmith import traceable

@traceable
def my_function(x, y):
    return x + y

@traceable(run_type="chain")
def my_pipeline(input):
    a = step_1(input)
    b = step_2(a)
    return b
```

### 3.3 Add Metadata

```python
result = chain.invoke(
    input,
    config={
        "run_name": "Customer Query #123",
        "tags": ["production", "customer-x"],
        "metadata": {
            "user_id": "u_456",
            "session_id": "s_789",
            "version": "v2",
        },
    },
)
```

---

## 4. Datasets

### 4.1 Tạo Dataset

```python
from langsmith import Client

client = Client()

dataset = client.create_dataset(
    dataset_name="my-eval-set",
    description="Golden set for RAG",
)

# Add examples
examples = [
    {
        "inputs": {"question": "Q1"},
        "outputs": {"answer": "A1"},
    },
    {
        "inputs": {"question": "Q2"},
        "outputs": {"answer": "A2"},
    },
]

for ex in examples:
    client.create_example(
        dataset_id=dataset.id,
        inputs=ex["inputs"],
        outputs=ex["outputs"],
    )
```

### 4.2 Import Từ Traces

```python
# Lấy runs gần đây
runs = client.list_runs(
    project_name="my-project",
    is_root=True,
    limit=100,
)

# Add to dataset
for run in runs:
    if run.feedback_stats.get("rating", 0) >= 4:  # High-rated runs
        client.create_example(
            dataset_id=dataset.id,
            inputs=run.inputs,
            outputs=run.outputs,
        )
```

### 4.3 Bulk Upload Từ CSV

```python
import csv

with open("examples.csv") as f:
    reader = csv.DictReader(f)
    for row in reader:
        client.create_example(
            dataset_id=dataset.id,
            inputs={"question": row["question"]},
            outputs={"answer": row["answer"]},
        )
```

---

## 5. Evaluations

### 5.1 Define Evaluator

```python
from langsmith.evaluation import RunEvaluator

def correctness_evaluator(run, example) -> dict:
    """LLM judge correctness"""
    predicted = run.outputs["answer"]
    expected = example.outputs["answer"]
    
    judge_response = llm.invoke(f"""
    Are these answers equivalent?
    Expected: {expected}
    Got: {predicted}
    
    Reply with JSON: {{"equivalent": true/false, "explanation": "..."}}
    """).content
    
    import json
    result = json.loads(judge_response)
    return {
        "key": "correctness",
        "score": 1.0 if result["equivalent"] else 0.0,
        "comment": result["explanation"],
    }

def latency_evaluator(run, example) -> dict:
    """Track latency"""
    return {
        "key": "latency_seconds",
        "score": (run.end_time - run.start_time).total_seconds(),
    }
```

### 5.2 Built-in Evaluators

```python
from langsmith.evaluation import LangChainStringEvaluator

# Exact match
exact_match = LangChainStringEvaluator("exact_match")

# Embedding similarity
embedding_sim = LangChainStringEvaluator("embedding_distance")

# LLM judge với criteria
criteria_eval = LangChainStringEvaluator(
    "criteria",
    config={"criteria": "Is the answer correct and helpful?"}
)
```

### 5.3 Run Experiment

```python
from langsmith.evaluation import evaluate

def my_chain(inputs):
    return {"answer": rag_pipeline.invoke(inputs["question"])}

results = evaluate(
    my_chain,
    data="my-eval-set",
    evaluators=[
        correctness_evaluator,
        latency_evaluator,
        # exact_match,
    ],
    experiment_prefix="rag-v2",
    metadata={"version": "2.0", "model": "gpt-4o-mini"},
)

print(results)
```

### 5.4 Compare Experiments

Trong UI, chọn 2 experiments → "Compare":
- Side-by-side outputs
- Metric deltas
- Drill into specific failures

---

## 6. Prompt Hub

### 6.1 Push Prompt

```python
from langchain import hub
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template("Answer {question}")

hub.push("my-username/my-rag-prompt", prompt)
```

### 6.2 Pull Prompt

```python
prompt = hub.pull("my-username/my-rag-prompt")

# Hoặc public prompts
prompt = hub.pull("rlm/rag-prompt")
```

### 6.3 Versioning

Mỗi lần push tạo version mới. Pull specific version:
```python
prompt = hub.pull("my-username/my-rag-prompt:v1")
```

---

## 7. Feedback & Annotations

### 7.1 Programmatic Feedback

```python
client.create_feedback(
    run_id=run.id,
    key="user_rating",
    score=5,
    comment="Great answer!",
)

client.create_feedback(
    run_id=run.id,
    key="correct",
    value=True,
)
```

### 7.2 Trong App (UI Feedback)

```python
# In Streamlit/Gradio
import streamlit as st

answer = rag.invoke(question)
run_id = ...  # Get from trace

col1, col2 = st.columns(2)
if col1.button("👍 Good"):
    client.create_feedback(run_id, key="thumbs", score=1)
if col2.button("👎 Bad"):
    client.create_feedback(run_id, key="thumbs", score=0)
```

### 7.3 Annotation Queue

UI cho human review runs:
- Filter runs by tag/score
- Add feedback
- Tag for retraining

---

## 8. Monitoring Dashboards

LangSmith UI cung cấp:
- 📊 Request count per day
- ⏱️ Latency p50/p95/p99
- 💰 Cost trends
- ❌ Error rates
- 🔝 Top tools used
- 👤 Per-user metrics

Custom dashboards: filter by tags, metadata.

---

## 9. Alerts

Setup alerts trong UI:
- Error rate > 5%
- p95 latency > 10s
- Daily cost > $X

Notifications: email, Slack, webhook.

---

## 10. CI/CD Integration

```yaml
# .github/workflows/eval.yml
name: LangSmith Eval

on:
  pull_request:

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: pip install -r requirements.txt
      - run: python run_eval.py
        env:
          LANGCHAIN_API_KEY: ${{ secrets.LANGCHAIN_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      - name: Check regression
        run: |
          python check_regression.py
```

`check_regression.py`:
```python
from langsmith import Client

client = Client()

# Get current PR experiment
current = client.read_project(name="pr-123-eval")
baseline = client.read_project(name="main-baseline")

# Compare
if current.feedback_stats["correctness"]["mean"] < baseline.feedback_stats["correctness"]["mean"] - 0.05:
    print("REGRESSION DETECTED")
    exit(1)
```

---

## 11. Self-Hosted LangSmith

Cho enterprise (data sovereignty):

```bash
# LangSmith on-prem
docker-compose up langsmith
```

Set:
```env
LANGCHAIN_ENDPOINT=https://langsmith.internal.company.com
```

---

## 12. Pricing (2026)

- **Developer**: free 5k traces/month
- **Pro**: $39/month + usage
- **Enterprise**: custom

---

## 13. Best Practices

✅ **Project per environment**: dev, staging, prod

✅ **Tag everything**: dễ filter sau

✅ **Save important runs to dataset**: build golden set

✅ **Daily/weekly experiments**: track improvements

✅ **Set alerts**: catch issues fast

✅ **Use Prompt Hub**: version control prompts

---

## 14. Demo: Full LangSmith Workflow

```python
import os
from langsmith import Client
from langsmith.evaluation import evaluate
from langchain_openai import ChatOpenAI

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_PROJECT"] = "demo-eval"

client = Client()
llm = ChatOpenAI(model="gpt-4o-mini")

# 1. Create dataset
dataset_name = "math-eval"
try:
    dataset = client.create_dataset(dataset_name)
except:
    dataset = client.read_dataset(dataset_name=dataset_name)

# 2. Add examples
test_data = [
    {"q": "2+2", "a": "4"},
    {"q": "10*3", "a": "30"},
    {"q": "100-50", "a": "50"},
]

for d in test_data:
    client.create_example(
        dataset_id=dataset.id,
        inputs={"question": d["q"]},
        outputs={"answer": d["a"]},
    )

# 3. Define chain
def math_chain(inputs):
    response = llm.invoke(f"Solve and return only number: {inputs['question']}")
    return {"answer": response.content.strip()}

# 4. Define evaluator
def correctness(run, example):
    predicted = run.outputs["answer"].strip()
    expected = example.outputs["answer"].strip()
    return {"key": "correct", "score": 1.0 if predicted == expected else 0.0}

# 5. Run eval
results = evaluate(
    math_chain,
    data=dataset_name,
    evaluators=[correctness],
    experiment_prefix="math-v1",
)

print(f"Results: {results}")
print(f"View at: https://smith.langchain.com")
```

---

## 15. Checklist

- [ ] Setup tracing
- [ ] Create datasets
- [ ] Define evaluators
- [ ] Run experiments
- [ ] Push/pull prompts từ Hub
- [ ] Add feedback từ app
- [ ] Setup CI/CD integration
- [ ] Configure alerts

➡️ **Tiếp theo**: [Unit Testing](./05-Unit-Testing.md)
