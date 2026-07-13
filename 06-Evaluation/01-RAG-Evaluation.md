# RAG Evaluation - Đánh Giá Hệ Thống RAG

## 1. Tại Sao Cần Eval?

RAG có nhiều "moving parts" (chunking, retrieval, reranking, generation). Không eval → không biết:
- Cấu hình nào tốt nhất
- Update có làm tệ đi không
- Issue nằm ở step nào

→ **Eval = đo lường khoa học**.

---

## 2. RAG Metrics Chính

### Retrieval Metrics
- **Recall@K**: bao nhiêu doc đúng trong top K
- **Precision@K**: trong top K, bao nhiêu là đúng
- **MRR (Mean Reciprocal Rank)**: vị trí của doc đúng đầu tiên
- **NDCG**: chất lượng ranking
- **Hit Rate**: ít nhất 1 doc đúng trong top K

### Generation Metrics
- **Faithfulness**: answer có dựa trên context không (anti-hallucination)
- **Answer Relevancy**: answer có liên quan câu hỏi không
- **Context Precision**: bao nhiêu context được dùng
- **Context Recall**: context có đủ để answer không
- **Answer Correctness**: so với ground truth

---

## 3. Golden Dataset

Tạo dataset chuẩn để eval:

```python
golden = [
    {
        "question": "LangChain là gì?",
        "ground_truth": "LangChain là framework AI...",
        "context": ["doc về LangChain"],  # Optional
    },
    {
        "question": "RAG hoạt động thế nào?",
        "ground_truth": "RAG kết hợp retrieval và generation...",
    },
    # ... 50-200 items
]
```

### Cách Build Golden Dataset

1. **Manual**: domain expert viết
2. **LLM-generated**: gen từ documents, sau đó review
3. **From logs**: chọn real queries từ production

```python
# Auto-generate Q&A từ docs
gen_prompt = """
Cho đoạn text sau, tạo 3 câu hỏi mà text trả lời được:

Text: {text}

Output JSON: [{{"question": "...", "answer": "..."}}, ...]
"""

# Generate và review
for doc in docs:
    questions = llm.invoke(gen_prompt.format(text=doc))
    # Manual review trước khi add vào golden set
```

---

## 4. Retrieval Eval

### 4.1 Manual Implementation

```python
def evaluate_retriever(retriever, golden_set, k=5):
    metrics = {"recall": 0, "mrr": 0, "hit": 0}
    
    for item in golden_set:
        question = item["question"]
        expected_ids = set(item["expected_doc_ids"])
        
        # Retrieve
        docs = retriever.invoke(question)
        retrieved_ids = [d.metadata["id"] for d in docs[:k]]
        
        # Hit rate
        if any(rid in expected_ids for rid in retrieved_ids):
            metrics["hit"] += 1
        
        # Recall@K
        found = sum(1 for rid in retrieved_ids if rid in expected_ids)
        metrics["recall"] += found / len(expected_ids)
        
        # MRR
        for i, rid in enumerate(retrieved_ids):
            if rid in expected_ids:
                metrics["mrr"] += 1 / (i + 1)
                break
    
    n = len(golden_set)
    return {k: v / n for k, v in metrics.items()}

# Use
metrics = evaluate_retriever(retriever, golden_set)
print(metrics)  # {"recall": 0.78, "mrr": 0.65, "hit": 0.92}
```

---

## 5. RAGAS - Framework Chuyên Dụng

```bash
pip install ragas
```

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# Prepare data
data = {
    "question": ["Q1", "Q2"],
    "answer": ["A1", "A2"],         # RAG output
    "contexts": [["ctx1"], ["ctx2"]],  # Retrieved docs
    "ground_truth": ["GT1", "GT2"],   # Expected
}

dataset = Dataset.from_dict(data)

result = evaluate(
    dataset,
    metrics=[
        faithfulness,
        answer_relevancy,
        context_precision,
        context_recall,
    ],
)

print(result)
# {"faithfulness": 0.85, "answer_relevancy": 0.92, ...}
```

### 5.1 Faithfulness

Đo: claims trong answer có support bởi context không.

```
Answer: "LangChain được tạo năm 2022 bởi Harrison Chase tại Google"
Context: "LangChain được tạo năm 2022 bởi Harrison Chase"

Claims trong answer:
1. "tạo năm 2022" → ✓ in context
2. "Harrison Chase" → ✓ in context
3. "tại Google" → ✗ NOT in context → hallucination

Faithfulness = 2/3 = 0.67
```

### 5.2 Answer Relevancy

Đo: answer có address câu hỏi không.

```python
# RAGAS dùng LLM để generate câu hỏi từ answer
# Nếu generated Q gần với original Q → relevant
```

### 5.3 Context Precision

Đo: top K retrieved docs có relevant không?

### 5.4 Context Recall

Đo: ground truth có "be supported" bởi context không?

---

## 6. LLM-as-Judge

Dùng LLM mạnh đánh giá output:

```python
judge_prompt = """
Đánh giá answer theo 1-10:

Question: {question}
Answer: {answer}
Ground Truth: {gt}

Tiêu chí:
- Correctness (đúng/sai)
- Completeness (đủ thông tin)
- Conciseness (ngắn gọn)

Output JSON: {{"correctness": int, "completeness": int, "conciseness": int, "reasoning": "..."}}
"""

judge = ChatOpenAI(model="gpt-4o")  # Dùng model tốt hơn để judge

def llm_judge(question, answer, gt):
    response = judge.invoke(judge_prompt.format(
        question=question, answer=answer, gt=gt
    ))
    return json.loads(response.content)
```

### 6.1 Pairwise Comparison

So sánh 2 systems:

```python
pairwise_prompt = """
Question: {q}
Answer A: {a}
Answer B: {b}
Ground truth: {gt}

Cái nào tốt hơn? Trả lời: A, B, hoặc TIE.
Reasoning: ...
"""
```

---

## 7. End-to-End RAG Eval

```python
from ragas import evaluate
from datasets import Dataset

def run_rag_eval(rag_pipeline, golden_set):
    questions = []
    answers = []
    contexts = []
    ground_truths = []
    
    for item in golden_set:
        result = rag_pipeline.invoke({"question": item["question"]})
        questions.append(item["question"])
        answers.append(result["answer"])
        contexts.append([d.page_content for d in result["docs"]])
        ground_truths.append(item["ground_truth"])
    
    dataset = Dataset.from_dict({
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths,
    })
    
    result = evaluate(dataset, metrics=[
        faithfulness, answer_relevancy,
        context_precision, context_recall,
    ])
    
    return result

# Run
metrics = run_rag_eval(my_rag_pipeline, golden_set)
print(metrics)
```

---

## 8. A/B Testing Configurations

```python
configs = [
    {"chunk_size": 500, "k": 3, "rerank": False},
    {"chunk_size": 1000, "k": 5, "rerank": False},
    {"chunk_size": 1000, "k": 10, "rerank": True},
    {"chunk_size": 2000, "k": 5, "rerank": True},
]

results = []
for config in configs:
    pipeline = build_pipeline(**config)
    metrics = run_rag_eval(pipeline, golden_set)
    results.append({"config": config, "metrics": metrics})

# Sort by score
results.sort(key=lambda r: r["metrics"]["faithfulness"], reverse=True)
for r in results:
    print(f"{r['config']}: {r['metrics']}")
```

---

## 9. LangSmith Eval

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

# 1. Upload dataset
dataset = client.create_dataset("rag-golden-set")
for item in golden_set:
    client.create_example(
        inputs={"question": item["question"]},
        outputs={"answer": item["ground_truth"]},
        dataset_id=dataset.id,
    )

# 2. Define evaluator
def correctness_evaluator(run, example):
    predicted = run.outputs["answer"]
    expected = example.outputs["answer"]
    
    # LLM judge
    score = llm_judge(
        example.inputs["question"],
        predicted,
        expected,
    )
    return {"score": score, "key": "correctness"}

# 3. Run eval
results = evaluate(
    lambda inputs: {"answer": rag_pipeline.invoke(inputs)},
    data="rag-golden-set",
    evaluators=[correctness_evaluator],
    experiment_prefix="rag-v2",
)

# 4. View in LangSmith UI
```

---

## 10. Continuous Eval In CI/CD

```yaml
# .github/workflows/eval.yml
name: RAG Eval

on:
  pull_request:
    paths:
      - 'rag/**'

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup
        run: pip install -r requirements.txt
      - name: Run eval
        run: |
          python eval.py
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      - name: Compare with baseline
        run: |
          python check_regression.py
          # Fail nếu metrics tệ hơn baseline 5%
```

---

## 11. Reporting

```python
import pandas as pd

def report(results):
    df = pd.DataFrame([
        {**r["config"], **r["metrics"]}
        for r in results
    ])
    print(df.to_markdown())
    
    # Save
    df.to_csv("eval_results.csv", index=False)
```

---

## 12. Best Practices

✅ **Build golden set sớm** - trước khi tối ưu

✅ **Eval multiple metrics** - không chỉ 1

✅ **Track over time** - regression check

✅ **Human review** - LLM judge có bias

✅ **Diverse questions** - factual, complex, edge cases

✅ **Automate** - tích hợp CI/CD

---

## 13. Common Issues & Fixes

| Issue | Symptom | Fix |
|-------|---------|-----|
| Low recall | Top K không có doc đúng | Tăng K, hybrid search, better chunking |
| Low faithfulness | Answer hallucinate | Better prompt, lower temp, verify |
| Low precision | Nhiều noise trong context | Rerank, compression |
| Slow | Latency cao | Cache, reduce K, faster model |

---

## 14. Checklist

- [ ] Hiểu RAG metrics chính
- [ ] Build golden dataset
- [ ] Implement basic retrieval metrics
- [ ] RAGAS eval
- [ ] LLM-as-judge
- [ ] A/B test configs
- [ ] LangSmith eval
- [ ] CI/CD integration

➡️ **Tiếp theo**: [Prompt Evaluation](./02-Prompt-Evaluation.md)
