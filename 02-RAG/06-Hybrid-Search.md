# Hybrid Search - Kết Hợp Semantic & Keyword

## 1. Vấn Đề Của Semantic Search

Semantic (vector) search hiểu nghĩa nhưng **kém với keyword chính xác**:

| Query | Semantic Search | Keyword Search |
|-------|-----------------|----------------|
| "AI là gì?" | ✅ Hiểu nghĩa | ⚠️ Cần khớp từ "AI" |
| "Error code E404" | ⚠️ Có thể miss | ✅ Khớp chính xác |
| "iPhone 15 Pro Max" | ⚠️ Trả về iPhone khác | ✅ Khớp model |
| "Người sáng tạo Python" | ✅ Hiểu paraphrasing | ❌ Không có từ |

→ **Hybrid Search** = kết hợp cả 2 → tốt nhất cả 2 thế giới.

---

## 2. BM25 - Keyword Search

**BM25 (Best Matching 25)** là thuật toán tìm kiếm cổ điển dựa trên:
- **Term Frequency (TF)**: từ xuất hiện nhiều → relevant hơn
- **Inverse Document Frequency (IDF)**: từ hiếm có giá trị hơn từ phổ biến
- **Document Length Normalization**: doc ngắn quá hoặc dài quá đều penalize

### 2.1 BM25 Trong LangChain

```python
from langchain_community.retrievers import BM25Retriever

docs = [
    "Python is a programming language",
    "JavaScript is used for web",
    "AI is artificial intelligence",
]

# pip install rank_bm25
retriever = BM25Retriever.from_texts(docs, k=2)
results = retriever.invoke("programming")
```

### 2.2 BM25 Cho Tiếng Việt

Tiếng Việt cần tokenize tốt:

```python
# pip install pyvi
from pyvi import ViTokenizer
from langchain_community.retrievers import BM25Retriever

# Custom preprocessing
def preprocess_vi(text: str) -> list:
    tokenized = ViTokenizer.tokenize(text.lower())
    return tokenized.split()

retriever = BM25Retriever.from_documents(
    documents,
    preprocess_func=preprocess_vi,
)
```

---

## 3. EnsembleRetriever - Combine Search

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# Semantic
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# BM25
bm25_retriever = BM25Retriever.from_documents(docs)
bm25_retriever.k = 5

# Ensemble với weights
ensemble = EnsembleRetriever(
    retrievers=[semantic_retriever, bm25_retriever],
    weights=[0.6, 0.4],
)

results = ensemble.invoke("query")
```

### 3.1 Algorithm: Reciprocal Rank Fusion (RRF)

EnsembleRetriever dùng RRF để merge results:

```
score(d) = Σ weight_i / (k + rank_i(d))
```

Trong đó:
- `rank_i(d)` = rank của doc d trong retriever i
- `k` = constant (default 60)

**Ví dụ**:
| Doc | Vector Rank | BM25 Rank | RRF Score |
|-----|-------------|-----------|-----------|
| A | 1 | 3 | 0.6/(60+1) + 0.4/(60+3) = 0.0162 |
| B | 2 | 1 | 0.6/(60+2) + 0.4/(60+1) = 0.0162 |
| C | 3 | 5 | 0.6/(60+3) + 0.4/(60+5) = 0.0157 |

→ A và B được rank cao nhất.

---

## 4. Native Hybrid Search Của Vector DB

Một số DB hỗ trợ hybrid sẵn (không cần ensemble).

### 4.1 Weaviate

```python
from langchain_weaviate import WeaviateVectorStore

vectorstore = WeaviateVectorStore(
    client=client,
    index_name="MyIndex",
    embedding=embeddings,
)

# Alpha: 0=BM25 only, 1=Vector only, 0.5=hybrid
results = vectorstore.similarity_search(
    "query",
    k=5,
    alpha=0.5,
)
```

### 4.2 Qdrant

```python
from langchain_qdrant import QdrantVectorStore
from qdrant_client.models import SparseVectorParams

# Tạo collection hybrid
client.create_collection(
    collection_name="hybrid",
    vectors_config={...},
    sparse_vectors_config={
        "text-sparse": SparseVectorParams()
    }
)
```

### 4.3 Elasticsearch

```python
from langchain_elasticsearch import ElasticsearchStore

vectorstore = ElasticsearchStore(
    es_url="http://localhost:9200",
    index_name="hybrid",
    embedding=embeddings,
    strategy=ElasticsearchStore.ApproxRetrievalStrategy(
        hybrid=True,  # Bật hybrid
    ),
)
```

### 4.4 Pinecone Hybrid

```python
# Cần index hỗ trợ sparse-dense
# Dùng SPLADE để generate sparse vector
from langchain_pinecone import PineconeHybridSearchRetriever
from pinecone_text.sparse import BM25Encoder

bm25_encoder = BM25Encoder().default()

retriever = PineconeHybridSearchRetriever(
    embeddings=embeddings,
    sparse_encoder=bm25_encoder,
    index=index,
    top_k=5,
    alpha=0.5,
)
```

---

## 5. Dense + Sparse Vectors

### 5.1 Khái Niệm

- **Dense vector**: từ embedding model (1536 dimensions, mỗi vị trí có giá trị)
- **Sparse vector**: từ TF-IDF/BM25 (vocab size dimensions, hầu hết là 0)

```
Dense:  [0.1, -0.2, 0.5, 0.3, ..., 0.7]  (1536 dim, dense)
Sparse: {0: 0, 1: 2.1, 2: 0, 3: 1.5, 4: 0, ...}  (50K dim, sparse)
```

### 5.2 SPLADE - Learned Sparse Retrieval

SPLADE = model train ra sparse vector tốt hơn BM25:

```python
# pip install splade
# Tạo sparse vector từ text
from splade.models.transformer_rep import Splade

model = Splade("naver/splade-cocondenser-ensembledistil")
sparse_vec = model.encode("query text")  # Tự động sparse
```

### 5.3 Hybrid = Dense + SPLADE

Tốt hơn BM25 + Dense vì cả 2 đều được train.

---

## 6. ColBERT - Late Interaction

ColBERT lưu **multiple vectors** per document (1 vector per token), tính similarity ở level token.

```python
# pip install ragatouille
from ragatouille import RAGPretrainedModel

# Index
RAG = RAGPretrainedModel.from_pretrained("colbert-ir/colbertv2.0")
RAG.index(
    collection=docs,
    index_name="my_index",
)

# Search
results = RAG.search(query="LangChain framework", k=3)
```

**Pros**: chính xác hơn dense retrieval
**Cons**: index lớn hơn, slower

---

## 7. Demo End-to-End: Hybrid RAG

```python
import os
from dotenv import load_dotenv
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_chroma import Chroma
from langchain_community.retrievers import BM25Retriever
from langchain.retrievers import EnsembleRetriever
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.documents import Document

load_dotenv()

# 1. Documents
documents = [
    Document(page_content="Python được tạo bởi Guido van Rossum năm 1991", 
             metadata={"id": "py1"}),
    Document(page_content="LangChain là framework AI tạo bởi Harrison Chase", 
             metadata={"id": "lc1"}),
    Document(page_content="Error code E404 nghĩa là Not Found", 
             metadata={"id": "err1"}),
    Document(page_content="iPhone 15 Pro Max ra mắt tháng 9/2023", 
             metadata={"id": "ip1"}),
    Document(page_content="Vector database lưu trữ embeddings", 
             metadata={"id": "vdb1"}),
]

# 2. Tạo cả 2 retriever
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(documents, embeddings)

semantic = vectorstore.as_retriever(search_kwargs={"k": 3})

bm25 = BM25Retriever.from_documents(documents)
bm25.k = 3

# 3. Ensemble
hybrid_retriever = EnsembleRetriever(
    retrievers=[semantic, bm25],
    weights=[0.5, 0.5],
)

# 4. RAG Chain
prompt = ChatPromptTemplate.from_template("""
Trả lời dựa trên context:

Context:
{context}

Question: {question}
""")

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def format_docs(docs):
    return "\n".join(d.page_content for d in docs)

chain = (
    {"context": hybrid_retriever | format_docs, "question": lambda x: x}
    | prompt
    | llm
    | StrOutputParser()
)

# 5. Test
queries = [
    "Ai tạo ra Python?",          # Semantic mạnh
    "E404",                       # Keyword mạnh  
    "iPhone 15 Pro Max",          # Keyword chính xác
    "Cái dùng để lưu vector?",    # Semantic paraphrase
]

for q in queries:
    print(f"\n❓ {q}")
    print(f"💬 {chain.invoke(q)}")
```

---

## 8. Tuning Weights

Cách tune weights cho ensemble:

```python
# Test set: (query, expected_doc_id)
test_set = [
    ("Ai tạo Python?", "py1"),
    ("E404", "err1"),
    ("iPhone 15", "ip1"),
    # ...
]

best_score = 0
best_weights = None

for w1 in [0.3, 0.5, 0.7]:
    w2 = 1 - w1
    ensemble = EnsembleRetriever(
        retrievers=[semantic, bm25],
        weights=[w1, w2],
    )
    
    correct = 0
    for query, expected_id in test_set:
        results = ensemble.invoke(query)
        if any(r.metadata["id"] == expected_id for r in results[:3]):
            correct += 1
    
    score = correct / len(test_set)
    print(f"Weights [{w1}, {w2}]: Score={score:.2f}")
    
    if score > best_score:
        best_score = score
        best_weights = (w1, w2)

print(f"Best: {best_weights} with score {best_score}")
```

---

## 9. Khi Nào Cần Hybrid?

### Cần Hybrid
- Có nhiều **technical terms** (error code, product name, ID)
- Mix giữa **factual queries** và **conceptual queries**
- Domain có **acronyms** đặc biệt (medical, legal)
- User search bằng keyword Google-style

### Không Cần Hybrid
- Pure semantic Q&A (chatbot)
- Document toàn natural language
- Data ít, dễ test semantic đã đủ tốt

---

## 10. Best Practices

✅ **Start simple**: vector search trước, thêm BM25 nếu cần

✅ **Test trên golden set**: measure recall@k với và không có BM25

✅ **Tune weights**: thường 0.5/0.5 hoặc 0.7/0.3 cho semantic

✅ **Cache BM25 index**: khá nhẹ, có thể keep in memory

✅ **Re-rank sau hybrid**: thêm reranker để tăng độ chính xác (next doc)

---

## 11. Bài Tập

### Bài 1: Hybrid Search Benchmark
Build 3 retriever: vector-only, BM25-only, hybrid. Đo recall@5 trên 50 test queries. So sánh.

### Bài 2: Tiếng Việt Hybrid
Index 100 bài báo tiếng Việt. Implement BM25 với pyvi tokenizer + vector search. So sánh với chỉ semantic.

### Bài 3: Multi-Hybrid System
Combine: vector + BM25 + SelfQueryRetriever. Test với queries kết hợp filter + semantic.

---

## 12. Checklist

- [ ] Hiểu khi nào semantic không đủ
- [ ] Setup được BM25 retriever
- [ ] Hiểu RRF algorithm
- [ ] Sử dụng EnsembleRetriever
- [ ] Native hybrid của Weaviate/Qdrant/Elastic
- [ ] Tune weights bằng test set

➡️ **Tiếp theo**: [Reranking](./07-Reranking.md)
