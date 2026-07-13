# Hallucination Check

## 1. Hallucination Là Gì?

**Hallucination** = LLM "bịa" thông tin không có trong context hoặc training data.

```
Context: "LangChain được tạo năm 2022"
Question: "Ai sáng lập LangChain?"
Answer: "Harrison Chase và John Doe vào năm 2020 tại MIT"
       ↑ Hallucination (chỉ "Harrison Chase 2022" có trong context)
```

---

## 2. Các Loại Hallucination

### 2.1 Intrinsic Hallucination
Trả lời sai dù có context đúng:
```
Context: "Paris is capital of France"
Answer: "London is capital of France"
```

### 2.2 Extrinsic Hallucination
Thêm thông tin không có trong context:
```
Context: "Paris is capital of France"
Answer: "Paris is capital of France with 12 million people"  # Bịa số
```

### 2.3 Factual Hallucination
Sai facts về thế giới thật:
```
Q: "When did WWII end?"
A: "1955"  # Sai - thực tế 1945
```

---

## 3. Detect Hallucination

### 3.1 NLI (Natural Language Inference)

Check answer có "entailed" by context:

```python
# pip install transformers
from transformers import pipeline

nli = pipeline(
    "text-classification",
    model="microsoft/deberta-large-mnli",
)

def check_entailment(context: str, claim: str) -> dict:
    result = nli(f"{context} </s> {claim}")
    # entailment, neutral, contradiction
    return result

# Use
ctx = "Paris is capital of France."
claim_good = "France's capital is Paris."
claim_bad = "London is France's capital."

print(check_entailment(ctx, claim_good))  # entailment ~0.95
print(check_entailment(ctx, claim_bad))   # contradiction ~0.90
```

### 3.2 LLM-Based Check

```python
faithfulness_prompt = """
Đánh giá answer có dựa hoàn toàn vào context không.

Context:
{context}

Question: {question}
Answer: {answer}

Hãy check từng claim trong answer:
1. List các claims (fact, number, name)
2. Check mỗi claim: SUPPORTED (có trong context), CONTRADICTED, NOT_FOUND

Output JSON:
{{
  "claims": [
    {{"claim": "...", "status": "SUPPORTED|CONTRADICTED|NOT_FOUND"}}
  ],
  "is_faithful": bool,
  "score": 0-1.0
}}
"""

def check_faithfulness(context, question, answer, llm):
    result = llm.invoke(faithfulness_prompt.format(
        context=context, question=question, answer=answer
    ))
    import json
    return json.loads(result.content)

# Use
result = check_faithfulness(
    context="LangChain được tạo năm 2022 bởi Harrison Chase",
    question="Ai tạo ra LangChain?",
    answer="Harrison Chase tại Google năm 2022",
    llm=ChatOpenAI(model="gpt-4o"),
)

print(result)
# {
#   "claims": [
#     {"claim": "Harrison Chase tạo LangChain", "status": "SUPPORTED"},
#     {"claim": "Tại Google", "status": "NOT_FOUND"},
#     {"claim": "Năm 2022", "status": "SUPPORTED"},
#   ],
#   "is_faithful": false,
#   "score": 0.67
# }
```

### 3.3 RAGAS Faithfulness

```python
from ragas.metrics import faithfulness
# Đã có sẵn implementation
```

---

## 4. Self-Consistency Check

Gọi LLM nhiều lần với prompt khác nhau, check consistency:

```python
def consistency_check(question, contexts, n=3):
    answers = []
    for _ in range(n):
        prompt = f"Context: {contexts}\nQ: {question}\nA:"
        answers.append(llm.invoke(prompt).content)
    
    # Check semantic similarity
    embeddings = OpenAIEmbeddings()
    vecs = embeddings.embed_documents(answers)
    
    similarities = []
    for i in range(len(vecs)):
        for j in range(i+1, len(vecs)):
            sim = cosine_sim(vecs[i], vecs[j])
            similarities.append(sim)
    
    avg_sim = sum(similarities) / len(similarities)
    return avg_sim  # < 0.7 = inconsistent = possibly hallucinating
```

---

## 5. Citation-Based Check

Yêu cầu LLM cite sources, sau đó verify:

```python
prompt_with_citation = """
Trả lời câu hỏi. Cite source [n] cho mỗi claim. Nếu không có info, nói 'I don't know'.

Sources:
[1] {source_1}
[2] {source_2}
[3] {source_3}

Question: {question}

Answer (với citations):
"""

def verify_citations(answer, sources):
    import re
    # Extract [n] references
    cited_nums = set(int(m) for m in re.findall(r'\[(\d+)\]', answer))
    
    # Split answer into sentences
    sentences = answer.split(".")
    
    for s in sentences:
        if "[" in s:
            # Has citation
            num = int(re.search(r'\[(\d+)\]', s).group(1))
            source = sources[num - 1]
            # Check claim against source
            if not is_supported_by(s, source):
                return False, f"Claim not supported: {s}"
    
    return True, "OK"
```

---

## 6. Grounding Score

```python
def grounding_score(answer: str, context: str) -> float:
    """
    Tỷ lệ tokens trong answer xuất hiện trong context.
    """
    answer_tokens = set(answer.lower().split())
    context_tokens = set(context.lower().split())
    
    common = answer_tokens & context_tokens
    return len(common) / len(answer_tokens) if answer_tokens else 0
```

⚠️ Lưu ý: metric đơn giản, có thể false positive (n-grams khác nhau).

---

## 7. Question-Answer Consistency

Sinh câu hỏi từ answer, check có match question gốc không:

```python
def qa_consistency(question, answer):
    # Generate question từ answer
    gen_prompt = f"Generate question that this answer responds to:\nAnswer: {answer}\nQuestion:"
    gen_q = llm.invoke(gen_prompt).content
    
    # Compare with original
    sim = semantic_similarity(question, gen_q)
    return sim  # > 0.8 = consistent
```

---

## 8. Anti-Hallucination Techniques

### 8.1 Better Prompts

❌ **Bad**:
```
Trả lời câu hỏi: {question}
Context: {context}
```

✅ **Good**:
```
RULES:
1. CHỈ trả lời dựa trên CONTEXT.
2. Nếu không có info → "Tôi không có thông tin về điều này."
3. KHÔNG bịa, suy đoán, hoặc thêm thông tin từ kiến thức chung.
4. Cite [Source X] sau mỗi claim.

Context:
{context}

Question: {question}

Answer (tuân thủ rules):
```

### 8.2 Lower Temperature

```python
llm = ChatOpenAI(temperature=0)  # Deterministic
```

### 8.3 Structured Output

Force output format → less freedom to hallucinate:

```python
class Answer(BaseModel):
    answer: str
    confidence: float
    cited_sources: list[str]
    is_uncertain: bool

llm.with_structured_output(Answer).invoke(...)
```

### 8.4 Self-Verification

```python
def generate_with_verification(question, context):
    # Step 1: Generate
    answer = llm.invoke(f"Context: {context}\nQ: {question}").content
    
    # Step 2: Verify
    verify_prompt = f"""
    Verify the answer is fully grounded in context:
    Context: {context}
    Answer: {answer}
    
    Issues found (or 'NONE'):
    """
    issues = llm.invoke(verify_prompt).content
    
    if issues.strip() != "NONE":
        # Step 3: Regenerate
        retry_prompt = f"""
        Previous answer had issues: {issues}
        Regenerate with only info from context.
        
        Context: {context}
        Q: {question}
        Better A:
        """
        answer = llm.invoke(retry_prompt).content
    
    return answer
```

### 8.5 Multi-Pass Generation

```python
# Pass 1: Draft
draft = llm.invoke("Answer Q with context...")

# Pass 2: Critique
critique = llm.invoke(f"Find issues in: {draft}")

# Pass 3: Revise
final = llm.invoke(f"Revise considering critique: {critique}\nOriginal: {draft}")
```

---

## 9. Production Anti-Hallucination Pipeline

```python
def safe_rag_answer(question, retriever, llm, judge_llm=None):
    judge = judge_llm or llm
    
    # 1. Retrieve
    docs = retriever.invoke(question)
    context = "\n".join(d.page_content for d in docs)
    
    # 2. Generate với strict prompt
    answer = llm.invoke(STRICT_RAG_PROMPT.format(
        context=context, question=question
    )).content
    
    # 3. Faithfulness check
    check = check_faithfulness(context, question, answer, judge)
    
    if not check["is_faithful"]:
        # Có hallucination
        # Option A: Regenerate
        answer = retry_with_feedback(question, context, check, llm)
        
        # Option B: Add disclaimer
        unsupported = [c["claim"] for c in check["claims"] if c["status"] != "SUPPORTED"]
        if unsupported:
            answer += f"\n\n⚠️ Note: Following may not be fully supported: {unsupported}"
    
    return {
        "answer": answer,
        "faithfulness_score": check["score"],
        "sources": [d.metadata for d in docs],
    }
```

---

## 10. Monitoring In Production

```python
import logging

class HallucinationMonitor:
    def __init__(self, threshold=0.7):
        self.threshold = threshold
        self.alerts = []
    
    def check_response(self, question, context, answer):
        score = check_faithfulness(context, question, answer)["score"]
        
        logging.info(f"Faithfulness: {score}")
        
        if score < self.threshold:
            alert = {
                "question": question,
                "answer": answer,
                "score": score,
                "timestamp": datetime.now(),
            }
            self.alerts.append(alert)
            
            # Send to monitoring
            send_to_slack(f"Low faithfulness: {score}\nQ: {question}")
        
        return score
```

---

## 11. Best Practices

✅ **Strict prompt + low temp**: foundation

✅ **Structured output**: less freedom for hallucination

✅ **Multi-pass với verification**: catch + fix

✅ **Citations required**: enforce grounding

✅ **Monitor in production**: detect drift

✅ **Allow "I don't know"**: better than wrong answer

---

## 12. Checklist

- [ ] Hiểu các loại hallucination
- [ ] Detect bằng NLI/LLM judge
- [ ] Self-consistency check
- [ ] Citation verification
- [ ] Strict prompting
- [ ] Production monitoring

➡️ **Tiếp theo**: [LangSmith Advanced](./04-LangSmith.md)
