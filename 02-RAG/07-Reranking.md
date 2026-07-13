# Reranking - Sắp Xếp Lại Kết Quả

## 1. Reranking Là Gì?

**Reranking** = bước thứ 2 sau retrieval, dùng model mạnh hơn để **sắp xếp lại** top K documents.

```
Query → [Retriever: lấy 20 docs] → [Reranker: chọn 5 best] → LLM
```

**Tại sao cần reranking?**
- Retrieval (embedding) chỉ "approximate"
- Embedding cosine không tối ưu cho ranking
- **Cross-encoder** đo tốt hơn nhưng chậm
  → Dùng cross-encoder để rerank chỉ top K (không phải toàn bộ)

---

## 2. Bi-Encoder vs Cross-Encoder

### 2.1 Bi-Encoder (Embedding)

```
Query → [Encoder] → Q_vector
Doc1  → [Encoder] → D1_vector
Doc2  → [Encoder] → D2_vector

score = cosine(Q_vector, D_vector)
```

- ✅ **Nhanh**: encode 1 lần, reuse
- ❌ **Kém chính xác**: query và doc encode độc lập

### 2.2 Cross-Encoder

```
(Query, Doc1) → [Cross-Encoder] → score1
(Query, Doc2) → [Cross-Encoder] → score2
```

- ✅ **Chính xác cao**: model thấy cả query + doc cùng lúc
- ❌ **Chậm**: phải encode (query, doc) cho mỗi pair

### 2.3 Best Of Both: Retrieve + Rerank

1. **Retrieve** với bi-encoder: lấy top 50 nhanh
2. **Rerank** với cross-encoder: pick top 5 chính xác

---

## 3. Cohere Rerank ⭐ (Phổ Biến Nhất)

### 3.1 Setup

```bash
pip install langchain-cohere cohere
```

Lấy API key: https://cohere.com/

### 3.2 Sử Dụng

```python
from langchain_cohere import CohereRerank
from langchain.retrievers import ContextualCompressionRetriever
import os

# Cohere reranker
reranker = CohereRerank(
    cohere_api_key=os.environ["COHERE_API_KEY"],
    model="rerank-multilingual-v3.0",  # hoặc rerank-english-v3.0
    top_n=5,  # Lấy 5 top sau rerank
)

# Wrap retriever
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)

# Lấy 20 docs, rerank xuống 5
results = compression_retriever.invoke("query")
```

### 3.3 Pricing
- Rerank v3: $2.00 / 1000 search units
- 1 search unit = 1 query + up to 100 docs

---

## 4. FlashRank - Open Source, Local

Nhẹ, free, không cần API.

```bash
pip install flashrank langchain-community
```

```python
from langchain_community.document_compressors import FlashrankRerank

reranker = FlashrankRerank(model="ms-marco-MiniLM-L-12-v2", top_n=5)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)

results = compression_retriever.invoke("query")
```

**Models available**:
- `ms-marco-MiniLM-L-12-v2` (default, balance)
- `ms-marco-MultiBERT-L-12` (multilingual)
- `rank-T5-flan` (T5-based)

---

## 5. Cross-Encoder Từ HuggingFace

```python
# pip install sentence-transformers
from sentence_transformers import CrossEncoder

model = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-12-v2")

query = "Ai tạo ra Python?"
docs = [
    "Python is a programming language",
    "Guido van Rossum created Python in 1991",
    "Java is also popular",
]

pairs = [[query, doc] for doc in docs]
scores = model.predict(pairs)

# Sort
ranked = sorted(zip(docs, scores), key=lambda x: x[1], reverse=True)
for doc, score in ranked:
    print(f"[{score:.3f}] {doc}")
```

### Tích Hợp Vào LangChain

```python
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain.retrievers.document_compressors import CrossEncoderReranker

cross_encoder = HuggingFaceCrossEncoder(
    model_name="BAAI/bge-reranker-large"  # Multilingual
)

reranker = CrossEncoderReranker(model=cross_encoder, top_n=5)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)
```

**BGE Reranker** (BAAI) - rất tốt:
- `BAAI/bge-reranker-large` - tiếng Anh
- `BAAI/bge-reranker-v2-m3` - đa ngôn ngữ (recommend)

---

## 6. Jina Reranker

```python
# pip install langchain-community
from langchain_community.document_compressors import JinaRerank

reranker = JinaRerank(
    jina_api_key="...",
    model="jina-reranker-v2-base-multilingual",
    top_n=5,
)
```

---

## 7. LLM-Based Reranking

Dùng LLM trực tiếp để rerank (chậm nhưng chất lượng cao).

```python
from langchain.retrievers.document_compressors import LLMListwiseRerank

reranker = LLMListwiseRerank.from_llm(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    top_n=5,
)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)
```

LLM nhận query + 20 docs, output ranked list.

### Custom LLM Rerank

```python
from langchain_core.prompts import ChatPromptTemplate

rerank_prompt = ChatPromptTemplate.from_template("""
Cho query và list docs. Hãy xếp hạng các doc theo độ liên quan với query.

Query: {query}

Documents:
{docs}

Output JSON: [{{"doc_index": 0, "score": 0-10, "reason": "..."}}, ...]
""")

def llm_rerank(query: str, docs: List[Document], llm, top_n=5):
    docs_text = "\n".join(f"[{i}] {d.page_content}" for i, d in enumerate(docs))
    response = llm.invoke(rerank_prompt.format(query=query, docs=docs_text))
    
    import json
    rankings = json.loads(response.content)
    rankings.sort(key=lambda x: x["score"], reverse=True)
    
    return [docs[r["doc_index"]] for r in rankings[:top_n]]
```

---

## 8. Demo End-to-End: Retrieve + Rerank

```python
import os
from dotenv import load_dotenv
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_chroma import Chroma
from langchain_cohere import CohereRerank
from langchain.retrievers import ContextualCompressionRetriever
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

load_dotenv()

embeddings = OpenAIEmbeddings()
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 1. Load + index (giả sử đã có)
vectorstore = Chroma(persist_directory="./db", embedding_function=embeddings)

# 2. Base retriever: lấy 20 docs
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# 3. Rerank xuống 5
reranker = CohereRerank(model="rerank-multilingual-v3.0", top_n=5)

retriever_with_rerank = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever,
)

# 4. RAG chain
prompt = ChatPromptTemplate.from_template("""
Trả lời câu hỏi dựa trên context.

Context:
{context}

Question: {question}
""")

def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

rag_chain = (
    {"context": retriever_with_rerank | format_docs, "question": lambda x: x}
    | prompt
    | llm
    | StrOutputParser()
)

# 5. Test
result = rag_chain.invoke("LangChain là gì?")
print(result)
```

---

## 9. Đo Lường Tác Động Của Reranking

```python
def measure_recall(retriever, test_set, k=5):
    correct = 0
    for query, expected_id in test_set:
        docs = retriever.invoke(query)
        if any(d.metadata.get("id") == expected_id for d in docs[:k]):
            correct += 1
    return correct / len(test_set)

# So sánh
print(f"No rerank:  {measure_recall(base_retriever):.2%}")
print(f"With rerank: {measure_recall(retriever_with_rerank):.2%}")
```

**Kết quả thường thấy**:
- Retrieval @ k=5: 65-75%
- Retrieval @ k=5 + Rerank: 80-90% (tăng 10-15%)

---

## 10. Khi Nào Cần Reranking?

### Cần Rerank
- ✅ Accuracy quan trọng (legal, medical)
- ✅ Top K nhỏ (chỉ chọn 3-5 docs)
- ✅ Domain knowledge khó match bằng embedding
- ✅ Cần ranking quality cao (RAG cho support bot)

### Không Cần
- ❌ Latency critical (<200ms total)
- ❌ K lớn (LLM context có thể chứa tất cả)
- ❌ Đơn giản, embedding đủ tốt

---

## 11. Latency & Cost

| Reranker | Latency (20 docs) | Cost |
|----------|-------------------|------|
| FlashRank (local) | 100-300ms | Free |
| BGE Reranker (local) | 200-500ms | Free |
| Cohere API | 100-200ms | $0.002 / 100 docs |
| Jina API | 100-200ms | Tương tự Cohere |
| LLM (GPT-4o-mini) | 1-3s | ~$0.001 / call |
| LLM (GPT-4o) | 2-5s | ~$0.01 / call |

---

## 12. Pipeline Complete: Hybrid + Rerank

```python
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_cohere import CohereRerank

# Layer 1: Hybrid retrieval (vector + BM25)
semantic = vectorstore.as_retriever(search_kwargs={"k": 30})
bm25 = BM25Retriever.from_documents(docs)
bm25.k = 30

hybrid = EnsembleRetriever(retrievers=[semantic, bm25], weights=[0.5, 0.5])

# Layer 2: Rerank
reranker = CohereRerank(top_n=5)
final_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=hybrid,
)

# Use
docs = final_retriever.invoke("query")
```

**Flow**:
1. Vector search: 30 docs
2. BM25 search: 30 docs
3. Ensemble: ~50 unique docs
4. Cohere rerank: top 5
5. Feed vào LLM

---

## 13. Best Practices

✅ **Retrieve 20-50, rerank xuống 3-5**

✅ **Cache reranker results**: query giống nhau → cùng kết quả

✅ **Multilingual model**: nếu data đa ngôn ngữ

✅ **Test offline**: measure recall@k với/không rerank

✅ **Monitor latency**: rerank thêm 100-500ms

---

## 14. Bài Tập

### Bài 1: Compare Rerankers
Test 3 reranker (Cohere, FlashRank, BGE) trên cùng 50 queries. Đo:
- Recall@5
- Latency
- Cost

### Bài 2: Self-Built Rerank
Implement LLM-based rerank chỉ dùng GPT-4o-mini. So sánh với Cohere.

### Bài 3: Adaptive Rerank
Chỉ rerank khi top similarity score thấp (uncertain). Save cost.

---

## 15. Checklist

- [ ] Hiểu bi-encoder vs cross-encoder
- [ ] Setup Cohere Rerank
- [ ] Setup FlashRank local
- [ ] Dùng BGE Reranker
- [ ] LLM-based reranking
- [ ] Đo lường impact của rerank
- [ ] Pipeline Hybrid + Rerank

➡️ **Tiếp theo**: [Query Rewriting](./08-Query-Rewriting.md)
