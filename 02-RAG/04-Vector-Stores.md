# Vector Stores - Lưu Trữ & Tìm Kiếm Vector

## 1. So Sánh Các Vector Store Phổ Biến

| Vector Store | Type | Pros | Cons | Use Case |
|--------------|------|------|------|----------|
| **Chroma** | Local/Cloud | Dễ dùng, free | Limited scale | Dev, prototype |
| **FAISS** | Local | Cực nhanh | No metadata filter, no persist API tốt | Local search |
| **Pinecone** | Cloud | Managed, scale tốt | Có phí | Production cloud |
| **Weaviate** | Self-host/Cloud | Hybrid search, modules | Setup phức tạp | Enterprise |
| **Qdrant** | Self-host/Cloud | Rust, nhanh, filter mạnh | Mới hơn | Production self-host |
| **Milvus** | Self-host/Cloud | Scale lớn, GPU support | Setup phức tạp | Big data |
| **pgvector** | PostgreSQL ext | Tích hợp Postgres | Slower than dedicated | Đã có Postgres |
| **Elasticsearch** | Self-host/Cloud | Hybrid (semantic + keyword) | Heavy | Enterprise search |

---

## 2. Chroma - Dễ Bắt Đầu Nhất

### 2.1 Setup

```bash
pip install langchain-chroma
```

### 2.2 In-Memory

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# In-memory
vectorstore = Chroma(embedding_function=embeddings)

# Add documents
vectorstore.add_documents(documents)

# Search
results = vectorstore.similarity_search("query", k=3)
```

### 2.3 Persistent (Lưu Disk)

```python
vectorstore = Chroma(
    embedding_function=embeddings,
    persist_directory="./chroma_db",  # Folder lưu
    collection_name="my_docs",
)

vectorstore.add_documents(documents)
# Chroma tự persist sau mỗi add (không cần gọi .persist() từ v0.4+)

# Load lại
vectorstore = Chroma(
    embedding_function=embeddings,
    persist_directory="./chroma_db",
    collection_name="my_docs",
)
```

### 2.4 from_documents (One-Shot)

```python
vectorstore = Chroma.from_documents(
    documents=documents,
    embedding=embeddings,
    persist_directory="./chroma_db",
)
```

### 2.5 Search Methods

```python
# Similarity search
results = vectorstore.similarity_search("query", k=3)

# Với score
results_with_score = vectorstore.similarity_search_with_score("query", k=3)
for doc, score in results_with_score:
    print(f"[{score:.3f}] {doc.page_content}")

# MMR (Maximum Marginal Relevance) - đa dạng kết quả
results = vectorstore.max_marginal_relevance_search(
    "query",
    k=3,
    fetch_k=10,        # Lấy 10 candidates
    lambda_mult=0.5,   # 0=đa dạng, 1=relevant
)
```

### 2.6 Metadata Filter

```python
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={"category": "tech"}
)

# Filter với operator
results = vectorstore.similarity_search(
    "query",
    k=5,
    filter={
        "$and": [
            {"category": {"$eq": "tech"}},
            {"date": {"$gte": "2024-01-01"}}
        ]
    }
)
```

### 2.7 Delete & Update

```python
# Add với explicit ID
vectorstore.add_documents(documents, ids=["doc1", "doc2"])

# Update
vectorstore.update_documents(
    ids=["doc1"],
    documents=[updated_doc]
)

# Delete
vectorstore.delete(ids=["doc1"])
```

### 2.8 Chroma Server (Production)

```bash
# Chạy Chroma server
chroma run --host 0.0.0.0 --port 8000
```

```python
import chromadb
from chromadb.config import Settings

client = chromadb.HttpClient(host="localhost", port=8000)

vectorstore = Chroma(
    client=client,
    embedding_function=embeddings,
    collection_name="my_docs",
)
```

---

## 3. FAISS - Facebook AI Similarity Search

### 3.1 Setup

```bash
pip install faiss-cpu   # hoặc faiss-gpu
```

### 3.2 Cơ Bản

```python
from langchain_community.vectorstores import FAISS

vectorstore = FAISS.from_documents(documents, embeddings)

results = vectorstore.similarity_search("query", k=3)
```

### 3.3 Save/Load

```python
# Save
vectorstore.save_local("./faiss_index")

# Load
vectorstore = FAISS.load_local(
    "./faiss_index",
    embeddings,
    allow_dangerous_deserialization=True,  # Cần thiết vì pickle
)
```

### 3.4 Merge Indexes

```python
vs1 = FAISS.from_documents(docs_set_1, embeddings)
vs2 = FAISS.from_documents(docs_set_2, embeddings)

# Merge vs2 vào vs1
vs1.merge_from(vs2)
```

### 3.5 FAISS Index Types

```python
# Flat - exact, dùng cho data nhỏ (< 10K vectors)
vs = FAISS.from_documents(docs, embeddings, distance_strategy="COSINE")

# Để dùng index khác (HNSW, IVF), cần dùng faiss-python trực tiếp
import faiss
import numpy as np

# Tạo HNSW index
d = 1536  # dimension
index = faiss.IndexHNSWFlat(d, 32)
# Sau đó wrap vào FAISS LangChain
```

**Khi nào dùng FAISS**:
- Local app, không cần server
- Performance critical
- Không cần metadata filter phức tạp

---

## 4. Pinecone - Managed Cloud

### 4.1 Setup

```bash
pip install langchain-pinecone pinecone-client
```

Tạo account: https://www.pinecone.io/

### 4.2 Create Index

```python
from pinecone import Pinecone, ServerlessSpec
import os

pc = Pinecone(api_key=os.environ["PINECONE_API_KEY"])

# Tạo index
pc.create_index(
    name="my-index",
    dimension=1536,  # Khớp với embedding model
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
)
```

### 4.3 Sử Dụng

```python
from langchain_pinecone import PineconeVectorStore

vectorstore = PineconeVectorStore.from_documents(
    documents,
    embeddings,
    index_name="my-index",
)

# Hoặc connect index có sẵn
vectorstore = PineconeVectorStore(
    index_name="my-index",
    embedding=embeddings,
)

results = vectorstore.similarity_search("query", k=3)
```

### 4.4 Namespaces

```python
# Phân namespace cho multi-tenant
vectorstore.add_documents(docs_user1, namespace="user_1")
vectorstore.add_documents(docs_user2, namespace="user_2")

# Search trong namespace
results = vectorstore.similarity_search(
    "query",
    k=3,
    namespace="user_1"
)
```

---

## 5. Qdrant - Production Self-Host

### 5.1 Setup

```bash
pip install langchain-qdrant qdrant-client

# Run Qdrant via Docker
docker run -p 6333:6333 qdrant/qdrant
```

### 5.2 Sử Dụng

```python
from langchain_qdrant import QdrantVectorStore
from qdrant_client import QdrantClient

client = QdrantClient(url="http://localhost:6333")

vectorstore = QdrantVectorStore.from_documents(
    documents,
    embeddings,
    url="http://localhost:6333",
    collection_name="my_collection",
)

results = vectorstore.similarity_search("query", k=3)
```

### 5.3 Filter Mạnh

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

results = vectorstore.similarity_search(
    "query",
    k=3,
    filter=Filter(
        must=[
            FieldCondition(key="category", match=MatchValue(value="tech")),
            FieldCondition(key="views", range=Range(gte=100)),
        ]
    )
)
```

---

## 6. pgvector - PostgreSQL Extension

### 6.1 Setup

```bash
# Trong Postgres:
CREATE EXTENSION vector;

# Python
pip install langchain-postgres psycopg2-binary
```

### 6.2 Sử Dụng

```python
from langchain_postgres import PGVector

CONNECTION_STRING = "postgresql+psycopg://user:pass@localhost:5432/dbname"

vectorstore = PGVector.from_documents(
    documents,
    embeddings,
    connection=CONNECTION_STRING,
    collection_name="my_docs",
    use_jsonb=True,  # Metadata trong JSONB
)
```

### 6.3 SQL Query Trực Tiếp

```sql
-- Tìm 5 documents gần nhất với 1 query vector
SELECT * FROM langchain_pg_embedding
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

---

## 7. Weaviate - Hybrid Search

### 7.1 Setup

```bash
pip install langchain-weaviate weaviate-client

# Docker
docker run -p 8080:8080 cr.weaviate.io/semitechnologies/weaviate:latest
```

### 7.2 Sử Dụng

```python
import weaviate
from langchain_weaviate import WeaviateVectorStore

client = weaviate.connect_to_local()

vectorstore = WeaviateVectorStore.from_documents(
    documents,
    embeddings,
    client=client,
    index_name="MyIndex",
)

# Hybrid search: semantic + keyword
results = vectorstore.similarity_search(
    "query",
    k=3,
    alpha=0.5,  # 0=keyword, 1=semantic
)
```

---

## 8. Elasticsearch

```python
# pip install langchain-elasticsearch
from langchain_elasticsearch import ElasticsearchStore

vectorstore = ElasticsearchStore.from_documents(
    documents,
    embeddings,
    es_url="http://localhost:9200",
    index_name="my_index",
    strategy=ElasticsearchStore.ApproxRetrievalStrategy(),
)

# Hybrid search
vectorstore = ElasticsearchStore(
    es_url="http://localhost:9200",
    index_name="my_index",
    embedding=embeddings,
    strategy=ElasticsearchStore.ApproxRetrievalStrategy(
        hybrid=True,
    ),
)
```

---

## 9. Update Strategy

### 9.1 Incremental Update

Nếu document có ID, có thể update mà không phải re-index toàn bộ:

```python
# Add với ID
vectorstore.add_documents(
    documents=[doc1, doc2],
    ids=["doc-001", "doc-002"]
)

# Update doc-001
vectorstore.update_documents(
    ids=["doc-001"],
    documents=[updated_doc1]
)

# Delete
vectorstore.delete(ids=["doc-002"])
```

### 9.2 LangChain Indexing API

API giúp đồng bộ vector store với nguồn dữ liệu:

```python
from langchain.indexes import SQLRecordManager, index
from langchain_core.documents import Document

# Setup record manager
namespace = "my_docs"
record_manager = SQLRecordManager(
    namespace=namespace,
    db_url="sqlite:///record_manager.db"
)
record_manager.create_schema()

# Index documents
result = index(
    documents,
    record_manager,
    vectorstore,
    cleanup="incremental",  # "incremental", "full", None
    source_id_key="source",  # Field để identify nguồn
)

print(result)
# {"num_added": 5, "num_updated": 2, "num_skipped": 10, "num_deleted": 1}
```

**Cleanup modes**:
- `None`: chỉ add mới
- `"incremental"`: update changed docs từ cùng source
- `"full"`: xóa docs không còn trong nguồn

---

## 10. Best Practices

### 10.1 Chọn Vector Store

```
Dev/prototype           → Chroma (in-memory hoặc local)
Production, small data  → FAISS hoặc Chroma server
Production, cloud      → Pinecone hoặc Qdrant Cloud
Production, self-host  → Qdrant, Weaviate, Milvus
Đã có Postgres         → pgvector
Cần keyword search    → Weaviate, Elasticsearch
```

### 10.2 Index Sizing

```
< 100K vectors  → Chroma, FAISS Flat
100K - 10M      → FAISS HNSW, Qdrant
10M - 100M+     → Milvus, Pinecone Pod-based
```

### 10.3 Backup

```python
# Chroma: copy folder persist_directory
# FAISS: save_local()
# Pinecone: dùng Pinecone backup feature
# pgvector: pg_dump

import shutil
shutil.copytree("./chroma_db", "./chroma_db_backup_2024-01-15")
```

### 10.4 Monitor

- **Số vectors**: theo dõi growth
- **Query latency**: p50, p95, p99
- **Cache hit rate**: nếu có cache layer
- **Storage size**: vectors + metadata

---

## 11. Demo End-to-End

```python
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain.indexes import SQLRecordManager, index

# 1. Load
loader = PyPDFLoader("guide.pdf")
docs = loader.load()

# 2. Split
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = splitter.split_documents(docs)

# 3. Setup vectorstore
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    embedding_function=embeddings,
    persist_directory="./db",
    collection_name="guides",
)

# 4. Setup record manager
record_manager = SQLRecordManager("guides", db_url="sqlite:///record.db")
record_manager.create_schema()

# 5. Index với incremental update
result = index(
    chunks,
    record_manager,
    vectorstore,
    cleanup="incremental",
    source_id_key="source",
)
print(result)

# 6. Search
results = vectorstore.similarity_search_with_score("query", k=3)
for doc, score in results:
    print(f"[{score:.3f}] {doc.metadata['source']} - {doc.page_content[:100]}")
```

---

## 12. Bài Tập

### Bài 1: So Sánh 3 Vector Store
Index cùng dataset 1000 docs vào Chroma, FAISS, Qdrant. So sánh:
- Tốc độ insert
- Tốc độ search
- Memory usage

### Bài 2: Multi-Tenant System
Build system multi-user, mỗi user có vector store riêng (dùng Pinecone namespace hoặc Chroma collection).

### Bài 3: Incremental Indexing
Tạo cron job mỗi ngày scan folder PDF mới và sync vào vector store (dùng Indexing API).

---

## 13. Checklist

- [ ] Hiểu sự khác biệt giữa các vector store
- [ ] Setup được Chroma, FAISS local
- [ ] Save/load vector store
- [ ] Metadata filtering
- [ ] MMR search
- [ ] Update/delete documents
- [ ] Indexing API với incremental cleanup
- [ ] Hiểu khi nào dùng cloud vs self-host

➡️ **Tiếp theo**: [Retrievers](./05-Retrievers.md)
