# Retrievers - Interface Truy Xuất Tài Liệu

## 1. Retriever Là Gì?

**Retriever** = abstraction để lấy documents liên quan đến query. Output là `List[Document]`.

Tất cả retriever có interface chung:
```python
retriever.invoke("query") → List[Document]
```

**Retriever khác Vector Store ra sao?**
- VectorStore: chỉ vector similarity search
- Retriever: có thể là vector store, BM25, web search, SQL, custom logic, hybrid, ...

---

## 2. Vector Store Retriever

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

vectorstore = Chroma.from_documents(docs, OpenAIEmbeddings())

# Tạo retriever từ vectorstore
retriever = vectorstore.as_retriever()

# Search
results = retriever.invoke("query")
```

### 2.1 Search Type

```python
# Default: similarity
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}
)

# MMR (đa dạng)
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 3,
        "fetch_k": 10,
        "lambda_mult": 0.5,
    }
)

# Similarity với threshold
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "k": 3,
        "score_threshold": 0.7,  # Chỉ lấy score > 0.7
    }
)
```

### 2.2 Filter

```python
retriever = vectorstore.as_retriever(
    search_kwargs={
        "k": 5,
        "filter": {"category": "tech"}
    }
)
```

---

## 3. BM25 Retriever - Keyword Search

BM25 = thuật toán keyword search cổ điển (như Google ngày xưa).

```python
from langchain_community.retrievers import BM25Retriever

retriever = BM25Retriever.from_documents(docs, k=3)
results = retriever.invoke("LangChain framework")
```

**Khi nào dùng BM25**:
- Tìm tên riêng, error code chính xác
- Domain có thuật ngữ kỹ thuật
- Kết hợp với semantic search (hybrid)

---

## 4. EnsembleRetriever - Hybrid Search

Kết hợp nhiều retriever (semantic + keyword):

```python
from langchain.retrievers import EnsembleRetriever

# Semantic retriever
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# BM25 retriever
bm25_retriever = BM25Retriever.from_documents(docs)
bm25_retriever.k = 5

# Combine
ensemble = EnsembleRetriever(
    retrievers=[semantic_retriever, bm25_retriever],
    weights=[0.7, 0.3],  # 70% semantic, 30% BM25
)

results = ensemble.invoke("query")
```

→ Sẽ học sâu ở [Hybrid Search](./06-Hybrid-Search.md)

---

## 5. MultiQueryRetriever - Mở Rộng Query

LLM sinh nhiều biến thể của query để search:

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm,
)

# Query "AI là gì?" sẽ được expand thành:
# 1. "AI là gì?"
# 2. "Artificial Intelligence định nghĩa là gì?"
# 3. "Thế nào là trí tuệ nhân tạo?"
# Sau đó search với mỗi query, gộp kết quả
results = retriever.invoke("AI là gì?")
```

**Bật verbose để xem queries**:
```python
import logging
logging.basicConfig()
logging.getLogger("langchain.retrievers.multi_query").setLevel(logging.INFO)
```

---

## 6. ContextualCompressionRetriever

Lấy nhiều docs, sau đó nén/lọc phần liên quan:

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

# Compressor dùng LLM để extract chỉ phần liên quan
compressor = LLMChainExtractor.from_llm(llm)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(),
)

# Lấy docs, mỗi doc chỉ giữ phần liên quan query
results = compression_retriever.invoke("query")
```

→ Sẽ học sâu ở [Contextual Compression](./09-Contextual-Compression.md)

---

## 7. ParentDocumentRetriever

Đã giới thiệu ở [Chunking](./03-Chunking-Strategy.md).

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore

child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=InMemoryStore(),
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
retriever.add_documents(documents)
```

---

## 8. MultiVectorRetriever - 1 Doc, Nhiều Vector

Mỗi document có nhiều "vector representation" (summary, hypothetical questions, ...).

```python
from langchain.retrievers.multi_vector import MultiVectorRetriever
from langchain_core.documents import Document
import uuid

# 1. Tạo summary cho mỗi doc
from langchain_core.prompts import ChatPromptTemplate
summary_prompt = ChatPromptTemplate.from_template("Tóm tắt: {doc}")
summary_chain = summary_prompt | llm | StrOutputParser()

# 2. Generate summaries
doc_ids = [str(uuid.uuid4()) for _ in documents]
summaries = summary_chain.batch([{"doc": d.page_content} for d in documents])

# 3. Tạo summary docs với metadata link tới original
summary_docs = [
    Document(page_content=s, metadata={"doc_id": doc_ids[i]})
    for i, s in enumerate(summaries)
]

# 4. Setup retriever
docstore = InMemoryStore()
docstore.mset(list(zip(doc_ids, documents)))

retriever = MultiVectorRetriever(
    vectorstore=vectorstore,  # chứa summary embeddings
    docstore=docstore,        # chứa original docs
    id_key="doc_id",
)
vectorstore.add_documents(summary_docs)

# 5. Search: tìm theo summary, trả về full doc
results = retriever.invoke("query")
```

---

## 9. SelfQueryRetriever - LLM Tự Tạo Filter

LLM phân tích query → tự generate metadata filter:

```python
from langchain.retrievers.self_query.base import SelfQueryRetriever
from langchain.chains.query_constructor.base import AttributeInfo

# Mô tả metadata fields
metadata_field_info = [
    AttributeInfo(
        name="genre",
        description="Thể loại phim. Một trong: 'action', 'drama', 'comedy'",
        type="string",
    ),
    AttributeInfo(
        name="year",
        description="Năm phát hành",
        type="integer",
    ),
    AttributeInfo(
        name="rating",
        description="Điểm đánh giá 0-10",
        type="float",
    ),
]

document_content_description = "Mô tả phim ngắn"

retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=metadata_field_info,
)

# Query: "Phim hành động sau 2020 rating > 8"
# → LLM tự generate filter: {"$and": [{"genre": "action"}, {"year": {"$gt": 2020}}, {"rating": {"$gt": 8}}]}
results = retriever.invoke("Phim hành động sau 2020 rating > 8")
```

---

## 10. TimeWeightedVectorStoreRetriever

Score = similarity + recency (mới hơn = điểm cao hơn).

```python
from langchain.retrievers import TimeWeightedVectorStoreRetriever
from datetime import datetime

retriever = TimeWeightedVectorStoreRetriever(
    vectorstore=vectorstore,
    decay_rate=0.01,  # 0=no decay, 1=fast decay
    k=4,
)

# Add docs với timestamp
retriever.add_documents([
    Document(
        page_content="...",
        metadata={"last_accessed_at": datetime.now()}
    )
])
```

**Use case**: chatbot có memory ưu tiên message gần đây.

---

## 11. Web Search Retriever

Search trên web (Google, Tavily, Brave):

### 11.1 Tavily

```python
# pip install langchain-tavily tavily-python
from langchain_community.retrievers import TavilySearchAPIRetriever

retriever = TavilySearchAPIRetriever(
    api_key=os.environ["TAVILY_API_KEY"],
    k=3,
)

docs = retriever.invoke("Tin tức AI mới nhất 2024")
```

### 11.2 Google Search

```python
from langchain_google_community import GoogleSearchAPIRetriever

retriever = GoogleSearchAPIRetriever(
    google_api_key="...",
    google_cse_id="...",
    k=5,
)
```

### 11.3 DuckDuckGo

```python
# Free, không cần API key
from langchain_community.tools import DuckDuckGoSearchRun

search = DuckDuckGoSearchRun()
result = search.invoke("AI news")
```

---

## 12. Wikipedia Retriever

```python
# pip install wikipedia
from langchain_community.retrievers import WikipediaRetriever

retriever = WikipediaRetriever(lang="vi", top_k_results=2)
docs = retriever.invoke("Việt Nam lịch sử")
```

---

## 13. Custom Retriever

```python
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from typing import List

class MyDBRetriever(BaseRetriever):
    """Retriever từ database tự build."""
    
    db_connection: any  # Pydantic field
    
    def _get_relevant_documents(
        self, query: str, *, run_manager
    ) -> List[Document]:
        # Logic search trong DB
        results = self.db_connection.search(query, limit=5)
        return [
            Document(
                page_content=r["text"],
                metadata={"id": r["id"], "source": "mydb"}
            )
            for r in results
        ]

retriever = MyDBRetriever(db_connection=my_db)
docs = retriever.invoke("query")
```

---

## 14. Retriever Với LCEL

### 14.1 Trong RAG Chain

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

prompt = ChatPromptTemplate.from_template(
    "Context:\n{context}\n\nQuestion: {question}"
)

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough()
    }
    | prompt
    | llm
    | StrOutputParser()
)
```

### 14.2 Async

```python
docs = await retriever.ainvoke("query")
```

### 14.3 Batch

```python
results = retriever.batch(["q1", "q2", "q3"])
```

---

## 15. Demo: Multi-Retriever System

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever, WikipediaRetriever
from langchain.retrievers.multi_query import MultiQueryRetriever

# 1. Vector retriever
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# 2. BM25
bm25 = BM25Retriever.from_documents(docs)
bm25.k = 5

# 3. Multi-query expansion on vector
multi_query = MultiQueryRetriever.from_llm(
    retriever=vector_retriever,
    llm=llm,
)

# 4. Wikipedia (external)
wiki = WikipediaRetriever(lang="vi", top_k_results=2)

# 5. Ensemble
ensemble = EnsembleRetriever(
    retrievers=[multi_query, bm25, wiki],
    weights=[0.5, 0.3, 0.2],
)

# Use
results = ensemble.invoke("LangChain là gì?")
```

---

## 16. Debug Retriever

### 16.1 Inspect Results

```python
docs = retriever.invoke("query")
for i, doc in enumerate(docs):
    print(f"\n[{i+1}] Source: {doc.metadata.get('source')}")
    print(f"    Score: {doc.metadata.get('score', 'N/A')}")
    print(f"    Content: {doc.page_content[:200]}...")
```

### 16.2 LangSmith Tracing

Set `LANGCHAIN_TRACING_V2=true` → trace mọi retriever call.

### 16.3 Add Logging

```python
from langchain.callbacks import StdOutCallbackHandler

docs = retriever.invoke(
    "query",
    config={"callbacks": [StdOutCallbackHandler()]}
)
```

---

## 17. Best Practices

✅ **Test với golden dataset**: 50-100 query với expected answer

✅ **Đo recall@k**: bao nhiêu doc đúng trong top K

✅ **Tune `k`**: thường 3-10

✅ **Hybrid > Single**: ensemble thường tốt hơn 1 retriever

✅ **Use metadata**: filter trước khi search vector

✅ **Cache**: retriever calls có thể cache

---

## 18. Bài Tập

### Bài 1: Build Hybrid Retriever
Combine vector + BM25 + Wikipedia. So sánh với từng retriever đơn lẻ.

### Bài 2: Self-Query Cho Movie DB
Index 100 movies với metadata: genre, year, rating, director.
Test SelfQueryRetriever với queries:
- "Phim hành động 90s rating > 7"
- "Phim của Christopher Nolan"

### Bài 3: Multi-Vector For Long Docs
Index 10 PDF dài (50+ pages). Mỗi PDF tạo summary + 5 hypothetical questions. Dùng MultiVectorRetriever.

---

## 19. Checklist

- [ ] Hiểu khác biệt Retriever vs Vector Store
- [ ] Tạo retriever từ vectorstore với MMR, threshold
- [ ] BM25 retriever
- [ ] EnsembleRetriever (hybrid)
- [ ] MultiQueryRetriever
- [ ] ParentDocumentRetriever
- [ ] MultiVectorRetriever
- [ ] SelfQueryRetriever
- [ ] Custom retriever
- [ ] Tích hợp web search retriever

➡️ **Tiếp theo**: [Hybrid Search](./06-Hybrid-Search.md)
