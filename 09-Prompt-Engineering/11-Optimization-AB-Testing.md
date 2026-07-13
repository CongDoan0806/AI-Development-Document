# Optimization & A/B Testing

## 1. Iterative Improvement

Prompt v1 rarely perfect. Cần process iterate:

```
Write → Test → Analyze → Fix → Test again → ...
```

### Workflow
```
1. Define metrics (accuracy, format, cost, latency)
2. Build test set (10-50 examples)
3. Write baseline prompt
4. Measure baseline metrics
5. Hypothesize improvements
6. A/B test versions
7. Pick winner
8. Deploy với canary
9. Monitor production
10. Loop
```

---

## 2. Define Metrics

### 2.1 Quality Metrics

```python
metrics = {
    # Accuracy
    "exact_match": 0.0,           # Exact string match
    "semantic_match": 0.0,        # Semantic similarity
    "factual_accuracy": 0.0,      # Facts correct
    
    # Format
    "format_compliance": 0.0,     # JSON valid
    "schema_match": 0.0,          # Pydantic validate
    
    # Style
    "tone_match": 0.0,            # Brand voice
    "length_appropriate": 0.0,
    
    # Safety
    "no_hallucination": 0.0,
    "no_pii_leak": 0.0,
}
```

### 2.2 Cost Metrics

```python
cost_metrics = {
    "input_tokens": 0,
    "output_tokens": 0,
    "cost_usd": 0.0,
    "tokens_per_request": 0,
}
```

### 2.3 Performance Metrics

```python
perf_metrics = {
    "latency_p50": 0.0,    # ms
    "latency_p95": 0.0,
    "latency_p99": 0.0,
    "throughput": 0,        # req/sec
}
```

---

## 3. Build Test Set

### 3.1 Golden Examples

```python
test_set = [
    {
        "id": "test_001",
        "input": {"text": "Yêu sản phẩm này!"},
        "expected": {"sentiment": "positive", "confidence": 0.95},
        "category": "easy",
    },
    {
        "id": "test_002",
        "input": {"text": "Tốt nhưng đắt"},
        "expected": {"sentiment": "neutral"},
        "category": "medium",  # Mixed
    },
    # ... 30-50 examples
]
```

### 3.2 Diversity

Cover:
- ✅ **Happy path**: input chuẩn
- ✅ **Edge cases**: empty, very long, malformed
- ✅ **Adversarial**: tricky, ambiguous
- ✅ **Domain coverage**: tất cả categories

### 3.3 Stratified Sample

```python
test_set = (
    sample(easy_examples, 10) +      # 30% easy
    sample(medium_examples, 15) +     # 45% medium
    sample(hard_examples, 8) +         # 25% hard
)
```

---

## 4. Baseline & Iteration

### Step 1: Baseline

```python
def evaluate(prompt_template, test_set, llm):
    results = []
    for item in test_set:
        prompt = prompt_template.format(**item["input"])
        output = llm.invoke(prompt).content
        score = compute_score(output, item["expected"])
        results.append({"id": item["id"], "score": score, "output": output})
    
    return {
        "mean_score": sum(r["score"] for r in results) / len(results),
        "results": results,
    }

baseline = evaluate(v1_prompt, test_set, llm)
print(f"V1 baseline: {baseline['mean_score']:.2%}")
```

### Step 2: Analyze Failures

```python
def analyze_failures(results):
    failures = [r for r in results if r["score"] < 0.5]
    
    print(f"Failed: {len(failures)}/{len(results)}")
    print(f"\nFailure samples:")
    for f in failures[:5]:
        print(f"\nID: {f['id']}")
        print(f"Output: {f['output'][:200]}")

analyze_failures(baseline["results"])
```

### Step 3: Hypothesize

```
Quan sát: Failures chủ yếu ở:
- Sentiment mixed (good thing + bad thing trong cùng câu)
- Sarcasm
- Multi-sentence với conflicting sentiment

Hypothesis: prompt cần:
1. Examples về mixed cases
2. Explicit rule cho sarcasm
3. Aggregate strategy cho multi-sentence
```

### Step 4: Improve

```python
v2_prompt = """
Phân loại sentiment thành: positive, negative, neutral

Quy tắc đặc biệt:
1. Mixed (good + bad) → neutral
2. Sarcasm → đánh giá ý thật, không lời nói
3. Multi-sentence → majority wins

Examples:
"Tốt nhưng đắt" → neutral (mixed)
"Yeah, super great, broke after 1 day..." → negative (sarcasm)
"Sản phẩm tuyệt vời. Giao chậm. Phục vụ tốt." → positive (2/3 positive)

Text: {text}
Output:
"""

v2_results = evaluate(v2_prompt, test_set, llm)
print(f"V1: {baseline['mean_score']:.2%}")
print(f"V2: {v2_results['mean_score']:.2%}")
```

### Step 5: Iterate

If V2 > V1 → keep improving.
If V2 < V1 → revert, try different hypothesis.

---

## 5. A/B Testing Framework

```python
class PromptABTest:
    def __init__(self, variants, test_set, llm):
        self.variants = variants  # {name: prompt_template}
        self.test_set = test_set
        self.llm = llm
        self.results = {}
    
    def run(self):
        for name, prompt in self.variants.items():
            print(f"\nTesting {name}...")
            self.results[name] = evaluate(prompt, self.test_set, self.llm)
    
    def report(self):
        print("\n=== A/B Test Results ===")
        print(f"{'Variant':<20} {'Score':<10} {'Tokens':<10} {'Cost':<10}")
        for name, result in self.results.items():
            print(f"{name:<20} {result['mean_score']:.2%}  ...")
    
    def winner(self):
        return max(self.results.items(), key=lambda x: x[1]["mean_score"])

# Use
test = PromptABTest(
    variants={
        "v1_basic": v1_prompt,
        "v2_examples": v2_prompt,
        "v3_cot": v3_prompt,
        "v4_role": v4_prompt,
    },
    test_set=test_set,
    llm=llm,
)
test.run()
test.report()
print(f"\nWinner: {test.winner()[0]}")
```

---

## 6. Statistical Significance

Score khác nhau 1% có thực sự có ý nghĩa?

```python
from scipy import stats

def is_significantly_better(scores_a, scores_b, alpha=0.05):
    """Paired t-test"""
    t_stat, p_value = stats.ttest_rel(scores_a, scores_b)
    
    if p_value < alpha:
        if t_stat > 0:
            return "A significantly better"
        else:
            return "B significantly better"
    return "No significant difference"

# Use
scores_v1 = [r["score"] for r in v1_results["results"]]
scores_v2 = [r["score"] for r in v2_results["results"]]
verdict = is_significantly_better(scores_v1, scores_v2)
print(verdict)
```

---

## 7. Multi-Dimensional Eval

Đo nhiều dimensions, weighted:

```python
def composite_score(result, weights):
    return (
        result["accuracy"] * weights["accuracy"] +
        result["format"] * weights["format"] +
        result["cost"] * weights["cost"]    # Lower better
        # ...
    )

weights = {
    "accuracy": 0.5,
    "format_compliance": 0.2,
    "cost_efficiency": 0.2,  # 1/cost normalized
    "latency": 0.1,           # 1/latency normalized
}

scores = {
    name: composite_score(result, weights)
    for name, result in test.results.items()
}
```

---

## 8. Auto-Optimization

### 8.1 Meta-Prompt Optimization

LLM tự cải thiện prompt:

```python
def auto_optimize(base_prompt, test_set, llm, n_iterations=5):
    current = base_prompt
    best_score = evaluate(current, test_set, llm)["mean_score"]
    
    for i in range(n_iterations):
        # Analyze failures
        failures = analyze_failures(current, test_set, llm)
        
        # Ask LLM to improve
        improve_prompt = f"""
        Current prompt:
        {current}
        
        Failures:
        {failures[:3]}
        
        Improve the prompt to fix these failures. 
        Output ONLY the new prompt, no explanation.
        """
        new_prompt = llm.invoke(improve_prompt).content
        
        # Test
        new_score = evaluate(new_prompt, test_set, llm)["mean_score"]
        
        if new_score > best_score:
            best_score = new_score
            current = new_prompt
            print(f"Iteration {i+1}: improved to {new_score:.2%}")
        else:
            print(f"Iteration {i+1}: no improvement")
    
    return current, best_score
```

### 8.2 DSPy - Programmatic Optimization

```python
# pip install dspy-ai
import dspy

# Define module
class SentimentClassifier(dspy.Module):
    def __init__(self):
        super().__init__()
        self.classify = dspy.Predict("text -> sentiment")
    
    def forward(self, text):
        return self.classify(text=text)

# Compile with optimizer
optimizer = dspy.BootstrapFewShot(
    metric=accuracy_metric,
    max_bootstrapped_demos=3,
)

compiled = optimizer.compile(
    SentimentClassifier(),
    trainset=train_examples,
)

# Use - DSPy tự tune prompt
result = compiled("Sản phẩm tốt!")
```

---

## 9. LangSmith For A/B

→ Đã giới thiệu ở [06-Evaluation/04-LangSmith](../06-Evaluation/04-LangSmith.md)

```python
from langsmith.evaluation import evaluate

# Run v1
results_v1 = evaluate(
    lambda x: {"output": chain_v1.invoke(x)},
    data="test-dataset",
    evaluators=[accuracy_evaluator],
    experiment_prefix="prompt-v1",
)

# Run v2
results_v2 = evaluate(
    lambda x: {"output": chain_v2.invoke(x)},
    data="test-dataset",
    evaluators=[accuracy_evaluator],
    experiment_prefix="prompt-v2",
)

# Compare in LangSmith UI
```

---

## 10. Production Deployment

### 10.1 Canary Release

```python
import random

def production_call(user_input):
    # 90% v1, 10% v2 (canary)
    if random.random() < 0.1:
        prompt = v2_prompt
        version = "v2_canary"
    else:
        prompt = v1_prompt
        version = "v1_stable"
    
    response = llm.invoke(prompt.format(input=user_input))
    
    # Log metrics
    log_metric(version, response, ...)
    
    return response
```

### 10.2 Gradual Rollout

```python
# Week 1: 10% v2
# Week 2: 30% v2
# Week 3: 50% v2 if metrics good
# Week 4: 100% v2

ROLLOUT_PERCENTAGE = {
    "v2": 0.1,  # Adjust over time
}
```

### 10.3 Rollback Plan

```python
# Feature flag
USE_V2 = config.get("use_prompt_v2", False)

prompt = v2_prompt if USE_V2 else v1_prompt

# If issues → toggle flag → instant rollback
```

---

## 11. Production Monitoring

### Track Metrics Per Version

```python
class PromptMetricsTracker:
    def __init__(self):
        self.metrics = defaultdict(list)
    
    def record(self, version, latency, tokens, success, user_feedback=None):
        self.metrics[version].append({
            "timestamp": datetime.now(),
            "latency": latency,
            "tokens": tokens,
            "success": success,
            "user_feedback": user_feedback,
        })
    
    def report(self, version):
        data = self.metrics[version]
        return {
            "n": len(data),
            "avg_latency": mean([d["latency"] for d in data]),
            "success_rate": sum(d["success"] for d in data) / len(data),
            "avg_feedback": mean(d["user_feedback"] for d in data if d["user_feedback"]),
        }
```

---

## 12. Common Optimization Patterns

### 12.1 Make Examples More Specific

```
Before:
"Examples:
'good' → positive
'bad' → negative"

After (more discriminative):
"Examples:
'good price but bad delivery' → neutral (mixed)
'sản phẩm tốt' → positive
'tệ' → negative"
```

### 12.2 Add Edge Cases

```
"Special cases:
- Empty string → neutral
- All caps (shouting) → +urgency, sentiment likely strong
- Multiple ! or ? → emphasis
- Emoji-only → infer from emoji"
```

### 12.3 Tighten Format

```
Before: "Output sentiment"
After: "Output ONLY one word: 'positive', 'negative', or 'neutral'.
        No punctuation, no quotes, no explanation."
```

### 12.4 Compress Instructions

```
Before (500 tokens of rules)
   →
After (200 tokens, prioritized)
```

→ Cheaper without quality loss.

---

## 13. Cost-Quality Pareto Frontier

```python
configs = [
    {"model": "gpt-4o-mini", "prompt": "basic", "k": 3},
    {"model": "gpt-4o-mini", "prompt": "detailed", "k": 5},
    {"model": "gpt-4o", "prompt": "basic", "k": 3},
    {"model": "gpt-4o", "prompt": "detailed", "k": 5},
]

results = []
for c in configs:
    metrics = evaluate(...)
    results.append({
        "config": c,
        "quality": metrics["accuracy"],
        "cost": metrics["cost_per_query"],
    })

# Plot quality vs cost
# Pick point on Pareto frontier (best quality for each cost level)
```

---

## 14. Demo: Full Optimization

```python
import pandas as pd
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Test set
test_set = [
    {"text": "Yêu sản phẩm này!", "expected": "positive"},
    {"text": "Tệ quá", "expected": "negative"},
    {"text": "Tốt nhưng đắt", "expected": "neutral"},
    {"text": "Bình thường", "expected": "neutral"},
    {"text": "Yeah, 'great'... broke after 1 day", "expected": "negative"},  # Sarcasm
    # ...
]

prompts = {
    "v1_basic": "Sentiment: {text}",
    
    "v2_options": """Classify sentiment as: positive, negative, or neutral.

Text: {text}
Sentiment:""",
    
    "v3_examples": """Classify sentiment.

Examples:
- "Yêu lắm" → positive
- "Quá tệ" → negative
- "Bình thường" → neutral

Text: {text}
Sentiment:""",
    
    "v4_full": """Bạn là expert sentiment analyzer.

Categories: positive, negative, neutral

Quy tắc:
- Mixed feelings → neutral
- Sarcasm → đánh giá ý thật
- Uncertain → neutral

Examples:
"Yêu lắm" → positive
"Tốt nhưng đắt" → neutral
"Yeah great... broke after 1 day" → negative

Text: {text}
Output (1 word only):""",
}

# Run A/B
results = {}
for name, p in prompts.items():
    correct = 0
    for tc in test_set:
        output = llm.invoke(p.format(text=tc["text"])).content.lower().strip()
        if tc["expected"] in output:
            correct += 1
    accuracy = correct / len(test_set)
    
    # Also measure tokens
    sample = llm.invoke(p.format(text="test"))
    tokens = sample.response_metadata.get("token_usage", {}).get("total_tokens", 0)
    
    results[name] = {
        "accuracy": accuracy,
        "avg_tokens": tokens,
    }

df = pd.DataFrame(results).T
print(df)
print(f"\nBest: {df['accuracy'].idxmax()}")
```

Output:
```
            accuracy  avg_tokens
v1_basic         0.6          12
v2_options       0.7          20
v3_examples      0.85         50
v4_full          0.92         85

Best: v4_full
```

---

## 15. Best Practices

✅ **Define metrics FIRST**, then write prompts

✅ **Test set diverse** & representative

✅ **A/B test, don't guess**

✅ **Iterate fast, small changes**

✅ **Track production metrics** continuously

✅ **Statistical significance** trước khi conclude

✅ **Document changes**: what worked, what didn't

✅ **Version control prompts** (Git, LangSmith Hub)

---

## 16. Bài Tập

### Bài 1: Full Optimization Cycle
- Chọn 1 task (e.g., extract entities từ news)
- Build test set 30 examples
- Write 4 prompt variants
- Run A/B
- Pick winner
- Document findings

### Bài 2: Auto-Optimize
Implement meta-prompt optimization. Test trên 20 failures. Đo improvement.

### Bài 3: Production Monitoring
Setup metrics tracking cho 1 prompt deployed. Track tokens, latency, success rate qua 1 tuần.

---

## 17. Checklist

- [ ] Define quality + cost + perf metrics
- [ ] Build diverse test set
- [ ] Baseline measurement
- [ ] Analyze failures systematically
- [ ] A/B test variants
- [ ] Statistical significance check
- [ ] LangSmith experiments
- [ ] Canary deployment
- [ ] Production monitoring
- [ ] Version control prompts

➡️ **Tiếp theo**: [Templates Library](./12-Templates-Library.md)
