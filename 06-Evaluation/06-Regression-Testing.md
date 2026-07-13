# Regression Testing - Tránh "Tệ Đi"

## 1. Regression Là Gì?

**Regression** = thay đổi mới làm AI app **tệ hơn** version cũ (dù không có lỗi visible).

```
Version 1: 85% accuracy, 2s latency, $0.01/query
Version 2 (sau update prompt): 78% accuracy, 2.5s, $0.012/query
                              ↑ Regression!
```

→ Cần test **mỗi khi thay đổi** để catch regressions.

---

## 2. Golden Set (Baseline)

Dataset cố định để so sánh qua các versions:

```python
# golden_set.json
[
    {
        "id": "q001",
        "question": "What is LangChain?",
        "expected_answer": "LangChain is a framework for developing AI applications",
        "category": "factual",
        "difficulty": "easy",
    },
    {
        "id": "q002",
        "question": "Explain RAG",
        "expected_answer": "RAG combines retrieval and generation...",
        "category": "explanation",
        "difficulty": "medium",
    },
    # ... 100-200 items
]
```

---

## 3. Run Baseline

```python
import json
from datetime import datetime

def run_baseline(pipeline, golden_set, version: str):
    results = []
    for item in golden_set:
        output = pipeline.invoke(item["question"])
        score = evaluate(output, item["expected_answer"])
        
        results.append({
            "id": item["id"],
            "question": item["question"],
            "output": output,
            "score": score,
        })
    
    # Save
    baseline = {
        "version": version,
        "timestamp": datetime.now().isoformat(),
        "results": results,
        "summary": {
            "mean_score": sum(r["score"] for r in results) / len(results),
            "total": len(results),
        }
    }
    
    with open(f"baselines/{version}.json", "w") as f:
        json.dump(baseline, f, indent=2)
    
    return baseline

# Setup baseline
baseline_v1 = run_baseline(pipeline_v1, golden_set, "v1.0")
print(f"v1 mean: {baseline_v1['summary']['mean_score']}")
```

---

## 4. Compare Versions

```python
def compare(baseline_a, baseline_b, threshold=0.05):
    """Detect regression"""
    
    summary = {
        "version_a": baseline_a["version"],
        "version_b": baseline_b["version"],
        "score_a": baseline_a["summary"]["mean_score"],
        "score_b": baseline_b["summary"]["mean_score"],
        "delta": baseline_b["summary"]["mean_score"] - baseline_a["summary"]["mean_score"],
        "regressed": [],
        "improved": [],
    }
    
    # Per-question
    results_a = {r["id"]: r for r in baseline_a["results"]}
    results_b = {r["id"]: r for r in baseline_b["results"]}
    
    for qid in results_a:
        if qid not in results_b:
            continue
        score_a = results_a[qid]["score"]
        score_b = results_b[qid]["score"]
        
        if score_a > score_b + threshold:
            summary["regressed"].append({
                "id": qid,
                "question": results_a[qid]["question"],
                "from": score_a,
                "to": score_b,
            })
        elif score_b > score_a + threshold:
            summary["improved"].append({"id": qid, "from": score_a, "to": score_b})
    
    return summary

# Use
comparison = compare(baseline_v1, baseline_v2)
print(f"Overall delta: {comparison['delta']:+.2%}")
print(f"Regressed: {len(comparison['regressed'])}")
print(f"Improved: {len(comparison['improved'])}")

if comparison['delta'] < -0.05:
    print("⚠️ REGRESSION DETECTED")
```

---

## 5. CI/CD Integration

### 5.1 GitHub Actions

```yaml
# .github/workflows/regression-check.yml
name: Regression Test

on:
  pull_request:
    paths:
      - 'src/**'
      - 'prompts/**'

jobs:
  regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: "3.11"
      
      - name: Install
        run: pip install -r requirements.txt
      
      - name: Run regression test
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          LANGCHAIN_API_KEY: ${{ secrets.LANGCHAIN_API_KEY }}
        run: python tests/regression.py --baseline=main --current=pr
      
      - name: Comment on PR
        uses: actions/github-script@v6
        if: always()
        with:
          script: |
            const results = require('./regression_results.json')
            const body = `## Regression Results
            
            **Score**: ${results.score_b.toFixed(2)} (delta: ${results.delta.toFixed(2)})
            **Regressed**: ${results.regressed.length}
            **Improved**: ${results.improved.length}
            
            ${results.delta < -0.05 ? '⚠️ Regression detected' : '✓ No regression'}
            `
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            })
```

### 5.2 Fail If Regression

```python
# tests/regression.py
import sys

baseline = load_baseline("main")
current = run_baseline(pipeline, golden_set, "pr")
comparison = compare(baseline, current)

if comparison["delta"] < -0.05:
    print(f"❌ Regression: {comparison['delta']:.2%}")
    print("Failed examples:")
    for r in comparison["regressed"][:5]:
        print(f"  - {r['question']}: {r['from']:.2f} → {r['to']:.2f}")
    sys.exit(1)

print(f"✓ OK: delta {comparison['delta']:+.2%}")
```

---

## 6. LangSmith Regression Workflow

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

# Định nghĩa evaluators
def evaluator(run, example):
    score = compute_score(run.outputs, example.outputs)
    return {"key": "accuracy", "score": score}

# Run experiment
results = evaluate(
    my_chain,
    data="golden-set",
    evaluators=[evaluator],
    experiment_prefix=f"regression-{datetime.now():%Y%m%d}",
)

# Get baseline experiment
baseline = client.read_project(project_name="main-baseline")

# Compare
if results.stats["accuracy"]["mean"] < baseline.stats["accuracy"]["mean"] - 0.05:
    raise Exception("Regression detected")
```

---

## 7. Test Different Aspects

### 7.1 Accuracy Regression

```python
def test_accuracy():
    correct = sum(1 for r in results if r["correct"])
    accuracy = correct / len(results)
    assert accuracy >= 0.85, f"Accuracy {accuracy} below baseline 0.85"
```

### 7.2 Latency Regression

```python
import time

def test_latency():
    durations = []
    for q in test_queries:
        start = time.time()
        pipeline.invoke(q)
        durations.append(time.time() - start)
    
    p95 = sorted(durations)[int(len(durations) * 0.95)]
    assert p95 < 5.0, f"p95 latency {p95}s exceeds 5s baseline"
```

### 7.3 Cost Regression

```python
from langchain.callbacks import get_openai_callback

def test_cost():
    with get_openai_callback() as cb:
        for q in test_queries:
            pipeline.invoke(q)
    
    avg_cost = cb.total_cost / len(test_queries)
    assert avg_cost < 0.015, f"Avg cost ${avg_cost} exceeds $0.015 baseline"
```

### 7.4 Format Compliance

```python
import json

def test_json_output():
    """Ensure outputs always valid JSON"""
    for q in test_queries:
        output = chain.invoke(q)
        try:
            data = json.loads(output)
            assert "answer" in data
        except:
            pytest.fail(f"Invalid JSON for: {q}")
```

---

## 8. Stratified Sampling

Track regression across categories:

```python
def regression_by_category(baseline, current):
    by_category = {}
    
    results_a = {r["id"]: r for r in baseline["results"]}
    results_b = {r["id"]: r for r in current["results"]}
    
    for qid, item in results_a.items():
        cat = item.get("category", "other")
        if cat not in by_category:
            by_category[cat] = {"count": 0, "delta_sum": 0}
        
        delta = results_b[qid]["score"] - item["score"]
        by_category[cat]["count"] += 1
        by_category[cat]["delta_sum"] += delta
    
    # Report
    for cat, data in by_category.items():
        avg_delta = data["delta_sum"] / data["count"]
        print(f"{cat}: {avg_delta:+.2%} (n={data['count']})")
```

Output:
```
factual: +0.02% (n=50)
explanation: -0.08% (n=30)  ← Regression here
edge_case: +0.05% (n=20)
```

→ Drill into "explanation" category.

---

## 9. Manual Annotation Of Regressions

```python
def review_regressions(comparison):
    print(f"Reviewing {len(comparison['regressed'])} regressed examples\n")
    
    for r in comparison["regressed"]:
        print(f"Q: {r['question']}")
        print(f"V1 ({r['from']:.2f}): {results_v1[r['id']]['output']}")
        print(f"V2 ({r['to']:.2f}): {results_v2[r['id']]['output']}")
        
        verdict = input("V2 actually better? (y/n/u=unsure): ")
        # Save annotation
```

→ Có thể golden set ground truth bị wrong, V2 thực sự tốt hơn.

---

## 10. Track Over Time

```python
# Plot trends
import matplotlib.pyplot as plt
import json
from pathlib import Path

baselines = [json.loads(p.read_text()) for p in Path("baselines").glob("*.json")]
baselines.sort(key=lambda b: b["timestamp"])

versions = [b["version"] for b in baselines]
scores = [b["summary"]["mean_score"] for b in baselines]

plt.plot(versions, scores, marker="o")
plt.xlabel("Version")
plt.ylabel("Mean Score")
plt.title("Regression Trends")
plt.xticks(rotation=45)
plt.show()
```

---

## 11. Best Practices

✅ **Golden set diverse**: cover all important use cases

✅ **Update golden set carefully**: ground truth must be correct

✅ **Multiple metrics**: accuracy + cost + latency + format

✅ **Auto-run on PR**: catch trước merge

✅ **Per-category breakdown**: detect localized regressions

✅ **Manual review borderline**: LLM judge có thể sai

✅ **Version everything**: prompt, model, config

---

## 12. Demo: Complete Regression Setup

```python
# regression_test.py
import json
import time
import sys
from datetime import datetime
from pathlib import Path

class RegressionRunner:
    def __init__(self, golden_path, baseline_dir):
        self.golden = json.loads(Path(golden_path).read_text())
        self.baseline_dir = Path(baseline_dir)
    
    def run(self, pipeline, version):
        results = []
        total_cost = 0
        durations = []
        
        for item in self.golden:
            start = time.time()
            try:
                from langchain.callbacks import get_openai_callback
                with get_openai_callback() as cb:
                    output = pipeline.invoke(item["question"])
                total_cost += cb.total_cost
            except Exception as e:
                output = f"ERROR: {e}"
            
            duration = time.time() - start
            durations.append(duration)
            
            # Score
            score = self._score(output, item["expected_answer"])
            
            results.append({
                "id": item["id"],
                "question": item["question"],
                "output": output[:500],
                "expected": item["expected_answer"][:500],
                "category": item.get("category"),
                "score": score,
                "duration": duration,
            })
        
        baseline = {
            "version": version,
            "timestamp": datetime.now().isoformat(),
            "results": results,
            "summary": {
                "mean_score": sum(r["score"] for r in results) / len(results),
                "p95_latency": sorted(durations)[int(len(durations)*0.95)],
                "total_cost": total_cost,
                "n": len(results),
            },
        }
        
        # Save
        (self.baseline_dir / f"{version}.json").write_text(
            json.dumps(baseline, indent=2)
        )
        return baseline
    
    def _score(self, actual, expected):
        # Simple containment + semantic
        actual_lower = actual.lower()
        expected_lower = expected.lower()
        
        # Containment
        keywords = [w for w in expected_lower.split() if len(w) > 4][:5]
        contained = sum(1 for w in keywords if w in actual_lower)
        return contained / len(keywords) if keywords else 0
    
    def compare(self, version_a, version_b, fail_threshold=-0.05):
        a = json.loads((self.baseline_dir / f"{version_a}.json").read_text())
        b = json.loads((self.baseline_dir / f"{version_b}.json").read_text())
        
        delta = b["summary"]["mean_score"] - a["summary"]["mean_score"]
        
        print(f"\n=== {version_a} → {version_b} ===")
        print(f"Score: {a['summary']['mean_score']:.3f} → {b['summary']['mean_score']:.3f} ({delta:+.3f})")
        print(f"p95 latency: {a['summary']['p95_latency']:.2f}s → {b['summary']['p95_latency']:.2f}s")
        print(f"Cost: ${a['summary']['total_cost']:.3f} → ${b['summary']['total_cost']:.3f}")
        
        if delta < fail_threshold:
            print(f"\n⚠️ REGRESSION (delta {delta:.3f} < {fail_threshold})")
            return False
        return True

# Use
runner = RegressionRunner("golden.json", "baselines/")

# Run versions
runner.run(pipeline_v1, "v1.0")
runner.run(pipeline_v2, "v2.0")

# Compare
ok = runner.compare("v1.0", "v2.0")
sys.exit(0 if ok else 1)
```

---

## 13. Checklist Phase 6

- [ ] Build golden set
- [ ] Run baseline
- [ ] Compare versions
- [ ] Multiple regression metrics
- [ ] CI/CD integration
- [ ] LangSmith experiments
- [ ] Stratified analysis
- [ ] Track trends over time

🎉 **Hoàn thành Phase 6 - Evaluation!**

➡️ **Tiếp theo**: [Phase 7 - Production](../07-Production/01-Caching.md)
