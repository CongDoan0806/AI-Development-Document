# Anti-Hallucination Prompting

## 1. Hallucination Là Gì?

LLM **bịa** thông tin không có trong context hoặc training, nhưng nói rất tự tin.

```
Q: "Cuốn sách 'Tuyết Trắng Núi Đỏ' của Nguyễn Văn X xuất bản năm nào?"
A: "Cuốn sách được xuất bản năm 2018 bởi NXB Văn học, đoạt giải thưởng A năm 2019..."
   ↑ Hoàn toàn bịa - sách không tồn tại
```

→ **Hallucination = enemy #1 của LLM apps**.

→ Xem thêm: [06-Evaluation/03-Hallucination-Check.md](../06-Evaluation/03-Hallucination-Check.md) cho detection.

---

## 2. Prevention Strategies

### Strategy 1: Strict Grounding Prompt

```
RULES (CRITICAL):
1. CHỈ dùng information trong CONTEXT
2. Nếu CONTEXT không có info → trả lời "Tôi không có thông tin này"
3. KHÔNG dùng general knowledge
4. KHÔNG suy đoán
5. KHÔNG combine information từ nhiều sources nếu không explicit liên quan
6. Cite [Source N] sau mỗi claim

Context:
{context}

Question: {question}

Answer (tuân thủ rules):
```

### Strategy 2: Encourage "I Don't Know"

```
Trả lời câu hỏi sau. 

Nếu bạn không chắc chắn về câu trả lời, HÃY NÓI:
- "Tôi không có thông tin về điều này"
- "Tôi không chắc, nhưng có thể là..."
- "Để xác nhận, hãy hỏi expert/check source X"

KHÔNG bao giờ bịa thông tin. Better to be honest than wrong.

Question: {question}
```

### Strategy 3: Confidence Scoring

```
Trả lời + confidence:

Question: {question}

Output JSON:
{
  "answer": "...",
  "confidence": 0.0 to 1.0,
  "reasoning": "...",
  "uncertainty_areas": ["where I'm unsure"]
}

Confidence guide:
- 1.0: Certain, from well-known facts
- 0.7-0.9: Likely correct
- 0.4-0.6: Possibly correct, but unsure
- 0.0-0.3: Don't know / guessing
```

### Strategy 4: Require Citations

```
Answer using ONLY the context. Format answer with citations [n].

Example:
"LangChain được tạo bởi Harrison Chase [1] vào năm 2022 [1]."

Context:
[1] LangChain was created by Harrison Chase in 2022.
[2] ...

Question: {question}

Answer (mọi claim phải có citation):
```

---

## 3. Self-Verification

LLM tự verify output.

### Pattern

```python
def verified_answer(question, llm):
    # Step 1: Generate
    answer = llm.invoke(f"Q: {question}\nA:").content
    
    # Step 2: Verify
    verify_prompt = f"""
    Question: {question}
    Proposed answer: {answer}
    
    Verify:
    1. Are facts in this answer correct?
    2. Are sources cited (or implied)?
    3. Are there unsupported claims?
    4. Should anything be marked as uncertain?
    
    Output JSON:
    {{
      "verified": bool,
      "issues": ["..."],
      "corrected_answer": "..."  // if needed
    }}
    """
    
    verification = llm.invoke(verify_prompt).content
    # Parse và use corrected version nếu cần
    return verification
```

---

## 4. Verify Against Context

```
Context:
{context}

Proposed answer: {draft_answer}

Task: Check from CONTEXT whether each claim in proposed answer is:
1. SUPPORTED - explicitly in context
2. NOT_FOUND - not in context (might be hallucination)
3. CONTRADICTED - context says opposite

Output:
[
  {"claim": "...", "verdict": "SUPPORTED|NOT_FOUND|CONTRADICTED"}
]

Then: produce corrected answer with only SUPPORTED claims.
```

---

## 5. Chain-of-Verification (CoVe)

Technique từ Meta AI để giảm hallucination:

### 4 Steps

```
Step 1: Generate baseline response
Step 2: Plan verification questions
Step 3: Answer each verification question independently
Step 4: Generate final response based on verifications
```

### Implementation

```python
def chain_of_verification(query):
    # Step 1: Baseline
    baseline = llm.invoke(query).content
    
    # Step 2: Plan questions
    plan_prompt = f"""
    Original response: {baseline}
    
    Generate 5 verification questions to fact-check this response.
    Each question should target 1 specific claim.
    """
    questions = llm.invoke(plan_prompt).content.split("\n")
    
    # Step 3: Answer independently
    verifications = []
    for q in questions:
        # Important: answer độc lập, không reference baseline
        ans = llm.invoke(f"Answer accurately: {q}").content
        verifications.append((q, ans))
    
    # Step 4: Final answer
    final_prompt = f"""
    Original query: {query}
    Initial response: {baseline}
    
    Verifications:
    {verifications}
    
    Generate FINAL response, correcting any inconsistencies.
    """
    return llm.invoke(final_prompt).content
```

---

## 6. Step-Back + Verification

Combine techniques:

```
Step 1: Step-back question (general context)
Step 2: Original question with context
Step 3: Verify answer against context
Step 4: Final answer
```

---

## 7. Temperature & Sampling

### Lower Temperature

```python
# Higher hallucination risk
llm = ChatOpenAI(temperature=1.5)

# Lower hallucination
llm = ChatOpenAI(temperature=0)
```

→ Temperature 0 nhưng vẫn có thể hallucinate (model uncertainty).

### Top_p

```python
llm = ChatOpenAI(temperature=0, top_p=0.1)  # Very strict
```

---

## 8. Structured Output Reduces Hallucination

Schema forces consistency:

```python
class FactClaim(BaseModel):
    claim: str
    source: str = Field(description="WHERE this info comes from")
    confidence: float = Field(ge=0, le=1)

class Answer(BaseModel):
    answer: str
    claims: List[FactClaim]
    unknown_aspects: List[str] = Field(description="What I don't know")
```

→ Each claim **REQUIRES** source field → harder to bịa.

---

## 9. Negative Prompting

Tell LLM EXACTLY what not to do:

```
DO NOT:
- Make up names, dates, numbers
- Claim authorship without source
- Provide statistics without citation
- Reference non-existent papers/books/laws
- Give specific URLs unless they're in context

IF UNCERTAIN:
- Say "I'm not certain, but..."
- Suggest where to verify
- Mark estimates as estimates
```

---

## 10. Test-Time Prompts

### Adversarial Self-Check

```
Question: {question}

First, ANSWER as if you know everything.

Then, RECONSIDER:
- Am I making this up?
- Do I actually know this, or am I guessing?
- What sources would I cite?

Then, FINAL ANSWER with appropriate hedging.
```

### Devil's Advocate

```
You answered: {answer}

Now, play devil's advocate:
- Find 3 reasons this answer could be wrong
- Identify what facts you assumed
- Rate your actual certainty 0-100%

Then revise if needed.
```

---

## 11. Domain-Specific Anti-Hallucination

### 11.1 Medical

```
RULES:
1. CHỈ provide general health information
2. KHÔNG diagnose conditions
3. KHÔNG recommend specific treatments
4. KHÔNG give drug dosages
5. ALWAYS recommend consulting healthcare provider
6. ALWAYS cite source nếu mention specific stat/study

If uncertain: "Please consult a doctor for personalized medical advice."
```

### 11.2 Legal

```
RULES:
1. KHÔNG give specific legal advice
2. KHÔNG reference specific laws/cases without source
3. ALWAYS clarify "this is general info, not legal advice"
4. ALWAYS recommend consulting lawyer
5. Note jurisdiction (laws vary by country)
```

### 11.3 Financial

```
RULES:
1. KHÔNG recommend specific stocks/investments
2. KHÔNG promise returns
3. CITE source for any number/stat
4. Disclaimer: "not financial advice, consult advisor"
5. Acknowledge market volatility
```

### 11.4 Historical/Factual

```
RULES:
1. CITE source for specific dates, names, places
2. Use "approximately" cho rough dates
3. Acknowledge conflicting accounts when relevant
4. Distinguish well-established facts vs controversial
5. NEVER invent quotes or attributions
```

---

## 12. Hedging Language

Train LLM dùng hedging khi uncertain:

```
Use hedge words appropriately:

CERTAIN (don't hedge):
- "The Earth orbits the Sun"
- "Python was created in 1991"

LIKELY (mild hedge):
- "X is generally considered..."
- "Most experts agree..."

UNCERTAIN (strong hedge):
- "I'm not entirely sure, but..."
- "This may be incorrect, but..."
- "I cannot verify, however..."

UNKNOWN (acknowledge):
- "I don't have information on this"
- "I'm uncertain about this"
```

---

## 13. Demo: Anti-Hallucination Pipeline

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field
from typing import List

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class GroundedAnswer(BaseModel):
    answer: str = Field(description="The answer text")
    claims: List[dict] = Field(description="[{claim, source, confidence}]")
    uncertainties: List[str] = Field(description="What I'm unsure about")
    needs_verification: List[str] = Field(description="Should be verified")
    confidence_overall: float

def safe_qa(question, context=None):
    if context:
        system = """You are a helpful assistant. ONLY use information from 
        the provided context. Do not use general knowledge. If context 
        doesn't have info, say so."""
        
        user = f"Context:\n{context}\n\nQuestion: {question}"
    else:
        system = """You are a helpful assistant. Be honest about what 
        you know vs don't know. Cite sources mentally. If uncertain, 
        say so explicitly."""
        
        user = question
    
    # Structured output forces grounding
    grounded_llm = llm.with_structured_output(GroundedAnswer)
    result = grounded_llm.invoke([
        ("system", system),
        ("user", user),
    ])
    
    # Display warnings if low confidence
    if result.confidence_overall < 0.7:
        print(f"⚠️ Low confidence ({result.confidence_overall})")
        print(f"   Uncertainties: {result.uncertainties}")
    
    return result

# Test
result = safe_qa("Ai viết cuốn 'Tuyết Trắng Núi Đỏ'?")
print(result.answer)
# Expected: "Tôi không có thông tin về cuốn sách này" 
#           hoặc acknowledge uncertainty
```

---

## 14. Monitoring In Production

```python
class HallucinationMonitor:
    def __init__(self, threshold=0.7):
        self.threshold = threshold
    
    def check(self, question, context, answer, llm):
        verify_prompt = f"""
        Faithfulness check:
        
        Context: {context}
        Question: {question}
        Answer: {answer}
        
        For each claim in answer, check if supported by context.
        
        Output JSON:
        {{
          "score": 0.0-1.0,
          "unsupported_claims": ["..."]
        }}
        """
        
        verification = llm.invoke(verify_prompt).content
        # Parse and act
        
        if score < self.threshold:
            self.alert(question, answer, unsupported_claims)
        
        return verification
    
    def alert(self, question, answer, claims):
        # Send to Slack, log, etc.
        pass
```

---

## 15. Combining Strategies

Production-grade anti-hallucination:

```python
def production_qa(question, llm, retriever):
    # 1. Retrieve relevant context
    docs = retriever.invoke(question)
    
    if not docs:
        return "I don't have information to answer this."
    
    context = "\n".join(f"[{i+1}] {d.page_content}" for i, d in enumerate(docs))
    
    # 2. Strict grounded prompt
    answer_prompt = f"""
    RULES:
    1. ONLY use info from context
    2. Cite [N] after each claim  
    3. If not in context, say so
    
    Context:
    {context}
    
    Question: {question}
    
    Answer:
    """
    
    answer = llm.invoke(answer_prompt).content
    
    # 3. Self-verify
    verify_prompt = f"""
    Context: {context}
    Answer: {answer}
    
    Are all claims in answer supported by context?
    If no, list unsupported claims.
    
    Output JSON: {{"verified": bool, "issues": [...]}}
    """
    verification = llm.invoke(verify_prompt).content
    
    # 4. Re-generate if issues
    # ... (parse verification, fix if needed)
    
    return {
        "answer": answer,
        "sources": [d.metadata for d in docs],
        "verification": verification,
    }
```

---

## 16. Bài Tập

### Bài 1: Compare Strategies
Cho 20 tricky questions (fact-based, ambiguous, unanswerable). Test:
- No anti-hallucination prompt
- Strict grounding prompt  
- + Self-verification
- + CoVe

Đo: hallucination rate.

### Bài 2: Chain-of-Verification
Implement full CoVe pipeline. Test với 10 multi-fact questions.

### Bài 3: Domain Constraints
Build prompts cho 1 trong: medical/legal/financial chatbot. Enforce all domain-specific rules.

---

## 17. Checklist

- [ ] Grounded prompt với strict rules
- [ ] Encourage "I don't know"
- [ ] Confidence scoring
- [ ] Citation requirements
- [ ] Self-verification loops
- [ ] Chain-of-Verification
- [ ] Structured output cho grounding
- [ ] Domain-specific rules
- [ ] Hedging language
- [ ] Production monitoring

➡️ **Tiếp theo**: [Prompt Security](./10-Prompt-Security.md)
