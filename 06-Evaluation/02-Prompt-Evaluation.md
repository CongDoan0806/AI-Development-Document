# Prompt Evaluation

## 1. Tại Sao Eval Prompts?

Khi tune prompt:
- Variation 1: 80% accuracy
- Variation 2: 85% accuracy
- Variation 3: 75% accuracy

→ Cần đo lường để chọn prompt tốt nhất, tránh dựa vào "feeling".

---

## 2. Setup Eval Framework

```python
class PromptEvalSuite:
    def __init__(self, test_cases, metrics):
        self.test_cases = test_cases  # List of (input, expected)
        self.metrics = metrics        # List of eval functions
    
    def evaluate(self, prompt_template, llm):
        results = []
        for input_data, expected in self.test_cases:
            prompt = prompt_template.format(**input_data)
            output = llm.invoke(prompt).content
            
            scores = {m.__name__: m(output, expected) for m in self.metrics}
            results.append({
                "input": input_data,
                "expected": expected,
                "output": output,
                "scores": scores,
            })
        return results

# Use
test_cases = [
    ({"text": "I love this"}, "positive"),
    ({"text": "This is terrible"}, "negative"),
    # ...
]

def exact_match(output, expected):
    return 1.0 if expected.lower() in output.lower() else 0.0

suite = PromptEvalSuite(test_cases, [exact_match])
```

---

## 3. A/B Testing Prompts

```python
prompt_v1 = "Classify sentiment: {text}\nAnswer:"
prompt_v2 = """Phân loại sentiment của text sau là 'positive', 'negative', hoặc 'neutral'.

Text: {text}

Sentiment:"""

prompt_v3 = """Examples:
"I love it" → positive
"Hate this" → negative
"It's okay" → neutral

Now classify: {text}
Answer:"""

results = {}
for name, prompt in [("v1", prompt_v1), ("v2", prompt_v2), ("v3", prompt_v3)]:
    suite_results = suite.evaluate(prompt, llm)
    avg = sum(r["scores"]["exact_match"] for r in suite_results) / len(suite_results)
    results[name] = avg

print(results)
# {"v1": 0.65, "v2": 0.80, "v3": 0.88}
```

---

## 4. Common Eval Metrics

### 4.1 Exact Match

```python
def exact_match(output: str, expected: str) -> float:
    return 1.0 if output.strip() == expected.strip() else 0.0
```

### 4.2 Contains

```python
def contains(output: str, expected: str) -> float:
    return 1.0 if expected.lower() in output.lower() else 0.0
```

### 4.3 Regex Match

```python
import re
def regex_match(output: str, pattern: str) -> float:
    return 1.0 if re.search(pattern, output) else 0.0
```

### 4.4 String Similarity

```python
from difflib import SequenceMatcher

def similarity(output: str, expected: str) -> float:
    return SequenceMatcher(None, output, expected).ratio()
```

### 4.5 Semantic Similarity

```python
from langchain_openai import OpenAIEmbeddings
import numpy as np

embeddings = OpenAIEmbeddings()

def semantic_similarity(output: str, expected: str) -> float:
    v1 = embeddings.embed_query(output)
    v2 = embeddings.embed_query(expected)
    return np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
```

### 4.6 ROUGE / BLEU (Tóm tắt, dịch)

```python
# pip install rouge-score
from rouge_score import rouge_scorer

def rouge_l(output: str, expected: str) -> float:
    scorer = rouge_scorer.RougeScorer(["rougeL"], use_stemmer=True)
    return scorer.score(expected, output)["rougeL"].fmeasure
```

### 4.7 LLM Judge

```python
def llm_judge(output, expected):
    judge_prompt = f"""
    Compare answers and return 0-10:
    Expected: {expected}
    Got: {output}
    
    Score:
    """
    score_str = llm.invoke(judge_prompt).content
    try:
        return float(score_str.strip()) / 10.0
    except:
        return 0.0
```

---

## 5. Structured Output Eval

```python
from pydantic import BaseModel

class Recipe(BaseModel):
    name: str
    ingredients: list[str]
    steps: list[str]

def eval_structured(output: dict, expected: Recipe) -> dict:
    scores = {}
    
    # Schema valid?
    try:
        parsed = Recipe(**output)
        scores["schema_valid"] = 1.0
    except:
        scores["schema_valid"] = 0.0
        return scores
    
    # Field-level
    scores["name_match"] = exact_match(parsed.name, expected.name)
    scores["ingredients_overlap"] = (
        len(set(parsed.ingredients) & set(expected.ingredients))
        / max(len(expected.ingredients), 1)
    )
    
    return scores
```

---

## 6. Cost & Latency Tracking

```python
import time
from langchain.callbacks import get_openai_callback

def benchmark_prompt(prompt_template, llm, test_cases):
    total_tokens = 0
    total_time = 0
    
    for input_data, expected in test_cases:
        prompt = prompt_template.format(**input_data)
        
        start = time.time()
        with get_openai_callback() as cb:
            output = llm.invoke(prompt).content
        elapsed = time.time() - start
        
        total_tokens += cb.total_tokens
        total_time += elapsed
    
    return {
        "avg_tokens": total_tokens / len(test_cases),
        "avg_latency": total_time / len(test_cases),
        "total_cost": cb.total_cost,
    }
```

---

## 7. LangSmith Prompt Eval

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

# Create dataset
client.create_dataset("sentiment-eval")
for input_data, expected in test_cases:
    client.create_example(
        inputs=input_data,
        outputs={"label": expected},
        dataset_name="sentiment-eval",
    )

# Define evaluator
def evaluate_match(run, example):
    predicted = run.outputs["text"].lower()
    expected = example.outputs["label"].lower()
    return {
        "score": 1.0 if expected in predicted else 0.0,
        "key": "match",
    }

# Run
def my_chain(inputs):
    return llm.invoke(prompt.format(**inputs))

results = evaluate(
    my_chain,
    data="sentiment-eval",
    evaluators=[evaluate_match],
    experiment_prefix="prompt-v2",
)
```

---

## 8. Demo: Prompt Optimization

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Test cases
test_cases = [
    ({"text": "Sản phẩm tuyệt vời, tôi sẽ mua lại!"}, "positive"),
    ({"text": "Chất lượng kém, hỏng sau 1 ngày"}, "negative"),
    ({"text": "Bình thường, không có gì đặc biệt"}, "neutral"),
    ({"text": "Giá tốt nhưng giao hàng chậm"}, "neutral"),  # Mixed
    ({"text": "Tuyệt!! Quá hài lòng!!! 5*"}, "positive"),
    # ... more cases
]

prompts = {
    "v1_basic": "Phân loại sentiment: {text}",
    
    "v2_options": "Phân loại sentiment thành 'positive', 'negative', 'neutral':\n\n{text}",
    
    "v3_examples": """Phân loại sentiment.

Examples:
"Tôi yêu sản phẩm này" → positive
"Quá tệ" → negative
"Cũng được" → neutral

Text: {text}
Sentiment:""",
    
    "v4_strict": """Bạn là sentiment classifier.

Rules:
1. Output CHỈ 1 trong: positive, negative, neutral
2. Không giải thích, không thêm gì khác
3. Nếu mixed → neutral
4. Nếu uncertain → neutral

Text: {text}
Output:""",
}

# Eval
def evaluate_prompt(prompt_template, test_cases):
    correct = 0
    for input_data, expected in test_cases:
        output = llm.invoke(prompt_template.format(**input_data)).content.lower().strip()
        if expected in output:
            correct += 1
    return correct / len(test_cases)

print(f"{'Prompt':<20} {'Accuracy':<10}")
print("-" * 30)
for name, p in prompts.items():
    acc = evaluate_prompt(p, test_cases)
    print(f"{name:<20} {acc:.1%}")
```

Output:
```
Prompt               Accuracy  
------------------------------
v1_basic             60.0%
v2_options           73.0%
v3_examples          87.0%
v4_strict            93.0%
```

---

## 9. Multi-Model Comparison

```python
models = {
    "gpt-4o-mini": ChatOpenAI(model="gpt-4o-mini"),
    "gpt-4o": ChatOpenAI(model="gpt-4o"),
    "claude-haiku": ChatAnthropic(model="claude-haiku-4-5"),
    "claude-sonnet": ChatAnthropic(model="claude-sonnet-4-6"),
}

best_prompt = prompts["v4_strict"]

print(f"{'Model':<20} {'Accuracy':<10} {'Avg Cost':<10}")
for name, model in models.items():
    acc = evaluate_prompt_with_model(best_prompt, test_cases, model)
    cost = benchmark_prompt(best_prompt, model, test_cases)
    print(f"{name:<20} {acc:.1%}      ${cost['total_cost']:.4f}")
```

---

## 10. Best Practices

✅ **Test set diverse**: cover edge cases

✅ **Multiple metrics**: accuracy + cost + latency

✅ **Versioning prompts**: LangSmith Hub

✅ **Test before deploy**: don't change prompt in production blindly

✅ **Real user queries**: include logs

✅ **Iterate fast**: small changes, test frequently

---

## 11. Bài Tập

### Bài 1: Sentiment Classifier Optimization
Build 5 prompt variations cho sentiment task. Test với 50 samples. Chọn best.

### Bài 2: Multi-Language Eval
Eval prompt với inputs tiếng Việt vs tiếng Anh. So sánh accuracy.

### Bài 3: LangSmith Experiment
Upload dataset lên LangSmith, run 3 prompts, compare trên dashboard.

---

## 12. Checklist

- [ ] Build test cases
- [ ] Multiple eval metrics
- [ ] A/B test prompts
- [ ] LLM-as-judge
- [ ] LangSmith eval setup
- [ ] Cost & latency tracking

➡️ **Tiếp theo**: [Hallucination Check](./03-Hallucination-Check.md)
