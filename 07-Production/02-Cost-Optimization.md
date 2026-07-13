# Cost Optimization

## 1. Cost Breakdown

```
Mỗi request RAG = 
  Embedding (query) + 
  Vector DB read + 
  LLM input tokens + 
  LLM output tokens

= $0.001 - $0.10 (tùy model & complexity)

1M requests/month → $1,000 - $100,000
```

→ Cần tối ưu mạnh.

---

## 2. Pricing Models (2026, USD/1M tokens)

| Model | Input | Output | Use case |
|-------|-------|--------|----------|
| Claude Opus 4.7 | $15 | $75 | Complex reasoning |
| Claude Sonnet 4.6 | $3 | $15 | General |
| Claude Haiku 4.5 | $1 | $5 | Fast/cheap |
| GPT-4o | $2.50 | $10 | Multimodal |
| GPT-4o-mini | $0.15 | $0.60 | Default |
| Gemini 2.0 Flash | $0.075 | $0.30 | Cheapest |
| DeepSeek V3 | $0.27 | $1.10 | Open weights |

---

## 3. Model Routing

Dùng model phù hợp cho từng task:

```python
def smart_route(question: str) -> str:
    # Classify complexity
    if is_simple(question):
        return "gpt-4o-mini"  # Cheap
    elif needs_reasoning(question):
        return "claude-opus-4-7"  # Smart but expensive
    return "claude-sonnet-4-6"  # Default

def get_llm(question):
    model = smart_route(question)
    return ChatOpenAI(model=model) if "gpt" in model else ChatAnthropic(model=model)
```

### Tiered Approach

```python
def answer_with_fallback(question):
    # Try cheap model first
    cheap_response = cheap_llm.invoke(question)
    
    # Check quality
    if check_quality(cheap_response) > 0.8:
        return cheap_response
    
    # Escalate to better model
    return expensive_llm.invoke(question)
```

---

## 4. Prompt Optimization

### 4.1 Shorter Prompts

```python
# Bad: 500 tokens
prompt = """Bạn là một trợ lý AI hữu ích, thông minh và đa năng.
Bạn được tạo ra để giúp đỡ người dùng trong nhiều vấn đề khác nhau,
bao gồm trả lời câu hỏi, viết code, tóm tắt, dịch thuật, v.v...
[continues for 400 more tokens]
"""

# Good: 50 tokens
prompt = "Bạn là trợ lý AI. Trả lời câu hỏi ngắn gọn, chính xác."
```

### 4.2 Compress Context

```python
# Quá nhiều docs
docs = retriever.invoke(q, k=20)  # 20 * 500 = 10,000 tokens

# Tối ưu
docs = retriever.invoke(q, k=5)  # 5 * 500 = 2,500 tokens
# + rerank để chất lượng vẫn cao
```

### 4.3 Few-Shot Selectively

```python
# Bad: luôn dùng 10 examples
examples = ALL_EXAMPLES[:10]  # 10 * 200 = 2,000 tokens

# Good: chọn examples liên quan
selector = SemanticSimilarityExampleSelector(...)
examples = selector.select(question, k=3)  # 3 * 200 = 600 tokens
```

---

## 5. Reduce Output Tokens

### 5.1 max_tokens

```python
llm = ChatOpenAI(max_tokens=200)  # Limit output
```

### 5.2 Strict Format

```python
prompt = """Trả lời CHỈ 1 từ: yes hoặc no.
Q: {question}
A:"""
# Output ~1 token
```

### 5.3 Structured Output

```python
class Answer(BaseModel):
    label: Literal["yes", "no"]
    confidence: float

# Compact JSON ~20 tokens
```

---

## 6. Batch Processing

Batch API rẻ hơn 50%:

### OpenAI Batch

```python
import json

requests = [
    {"custom_id": f"req-{i}", "method": "POST", "url": "/v1/chat/completions", 
     "body": {"model": "gpt-4o-mini", "messages": [{"role": "user", "content": q}]}}
    for i, q in enumerate(questions)
]

with open("batch.jsonl", "w") as f:
    for r in requests:
        f.write(json.dumps(r) + "\n")

# Upload và submit batch
# Wait up to 24h cho results
```

### Anthropic Batch (50% off)

```python
from anthropic import Anthropic

client = Anthropic()

batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"req-{i}",
            "params": {
                "model": "claude-sonnet-4-6",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": q}],
            }
        }
        for i, q in enumerate(questions)
    ]
)
```

---

## 7. Caching Strategy

→ Đã học ở [Caching](./01-Caching.md). Recap:

- Cache exact match: 100% saving cho repeat queries
- Cache semantic: 50% saving cho similar queries
- Prompt caching: 90% saving cho cached prefix tokens

---

## 8. Streaming + Early Exit

Stream output, stop khi có đủ info:

```python
def generate_until_complete(prompt):
    text = ""
    for chunk in llm.stream(prompt):
        text += chunk.content
        if "ANSWER:" in text and len(text.split("ANSWER:")[1]) > 50:
            break  # Got enough
    return text
```

---

## 9. Token Counting

```python
import tiktoken

def count_tokens(text: str, model="gpt-4o-mini") -> int:
    encoding = tiktoken.encoding_for_model(model)
    return len(encoding.encode(text))

# Pre-check cost trước khi gọi
prompt = build_prompt(question, context)
tokens = count_tokens(prompt)
cost = tokens / 1_000_000 * 0.15  # GPT-4o-mini input

if cost > 0.01:  # Threshold
    # Compress prompt
    prompt = compress(prompt)
```

---

## 10. Open Source Models

Self-host để 0 cost per call (chỉ infra):

### Ollama (Local)

```python
from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.1:8b")
# Chạy local, không tốn API
```

### vLLM / TGI (Server)

```bash
# vLLM server
vllm serve meta-llama/Llama-3.1-8B-Instruct
```

```python
llm = ChatOpenAI(
    base_url="http://localhost:8000/v1",
    api_key="dummy",
)
```

**Trade-off**:
- ✅ No per-call cost
- ❌ Infrastructure cost ($500-5000/month)
- ❌ Lower quality than GPT-4o

**Khi nào dùng**: high volume, latency-sensitive, data privacy.

---

## 11. RAG Cost Optimization

### 11.1 Cheaper Embedding Model

```python
# text-embedding-3-large: $0.13/1M tokens
# text-embedding-3-small: $0.02/1M tokens
# voyage-3-lite: $0.02/1M tokens

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
```

### 11.2 Compress Context Before LLM

```python
from langchain.retrievers.document_compressors import EmbeddingsFilter

compressor = EmbeddingsFilter(
    embeddings=embeddings,
    similarity_threshold=0.76,
)
# Loại docs không relevant → giảm input tokens
```

### 11.3 Smaller K

```python
# k=20 → 20 docs × 500 tokens = 10K input tokens
# k=5 (với rerank) → 5 docs × 500 = 2.5K input tokens
# Quality tương đương, chi phí 1/4
```

---

## 12. Monitor Cost

### 12.1 Track Per Request

```python
from langchain.callbacks import get_openai_callback

with get_openai_callback() as cb:
    result = chain.invoke(input)
    
    log = {
        "user_id": user_id,
        "tokens": cb.total_tokens,
        "cost": cb.total_cost,
        "timestamp": datetime.now(),
    }
    save_to_db(log)
```

### 12.2 Daily Report

```python
def daily_cost_report():
    today_costs = db.query(
        "SELECT SUM(cost), COUNT(*) FROM requests WHERE DATE(timestamp) = CURDATE()"
    )
    total_cost, count = today_costs[0]
    
    print(f"Today: ${total_cost:.2f} ({count} requests)")
    print(f"Avg per request: ${total_cost/count:.4f}")
    
    # Alert
    if total_cost > 100:
        send_alert(f"Daily cost ${total_cost} exceeds budget")
```

### 12.3 Per-User Limits

```python
def check_user_budget(user_id):
    monthly = db.query(
        "SELECT SUM(cost) FROM requests WHERE user_id = %s AND MONTH(timestamp) = MONTH(NOW())",
        [user_id]
    )[0][0] or 0
    
    if monthly > 10:  # $10/month limit
        raise Exception("Monthly budget exceeded")
```

---

## 13. Cost-Quality Trade-off

```python
configs = {
    "premium": {
        "model": "gpt-4o",
        "rerank": True,
        "k": 10,
        "expected_cost": 0.05,
        "expected_quality": 0.95,
    },
    "balanced": {
        "model": "gpt-4o-mini",
        "rerank": True,
        "k": 5,
        "expected_cost": 0.01,
        "expected_quality": 0.85,
    },
    "cheap": {
        "model": "gpt-4o-mini",
        "rerank": False,
        "k": 3,
        "expected_cost": 0.003,
        "expected_quality": 0.75,
    },
}

# User chọn theo budget
config = configs[user_tier]
```

---

## 14. Demo: Smart Cost Optimization

```python
import tiktoken
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

def estimate_cost(input_tokens, output_tokens, model):
    rates = {
        "gpt-4o": (2.50, 10),
        "gpt-4o-mini": (0.15, 0.60),
        "claude-sonnet-4-6": (3, 15),
        "claude-haiku-4-5": (1, 5),
    }
    in_rate, out_rate = rates[model]
    return (input_tokens / 1_000_000 * in_rate + 
            output_tokens / 1_000_000 * out_rate)

class CostOptimizedRouter:
    def __init__(self):
        self.models = {
            "cheap": ChatOpenAI(model="gpt-4o-mini"),
            "medium": ChatAnthropic(model="claude-sonnet-4-6"),
            "expensive": ChatAnthropic(model="claude-opus-4-7"),
        }
        self.budget_per_request = 0.05  # $0.05
    
    def classify(self, prompt):
        # Length-based
        tokens = len(tiktoken.encoding_for_model("gpt-4o").encode(prompt))
        if tokens < 500:
            return "cheap"
        elif tokens < 2000:
            return "medium"
        return "expensive"
    
    def invoke(self, prompt):
        tier = self.classify(prompt)
        return self.models[tier].invoke(prompt)

router = CostOptimizedRouter()
result = router.invoke("Simple question")  # Uses cheap
result = router.invoke("Complex multi-paragraph reasoning...")  # Uses expensive
```

---

## 15. Best Practices

✅ **Default to cheapest viable model**

✅ **Aggressive caching**: aim 30%+ hit rate

✅ **Compress prompts ruthlessly**: every token counts

✅ **Batch when possible**: 50% off

✅ **Native prompt caching**: free 90% off cached tokens

✅ **Monitor per user**: alert outliers

✅ **Set budgets**: per request, per user, per day

---

## 16. Checklist

- [ ] Understand pricing tiers
- [ ] Model routing strategy
- [ ] Prompt compression
- [ ] Output token limits
- [ ] Batch API for offline
- [ ] Open source for high volume
- [ ] Cost tracking dashboard
- [ ] User-level budgets

➡️ **Tiếp theo**: [Streaming](./03-Streaming.md)
