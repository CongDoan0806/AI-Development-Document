# Embeddings & Vectors - Nền Tảng Của RAG

## 1. Embedding Là Gì?

**Embedding** = chuyển text/image/audio thành **vector số** (mảng số) trong không gian nhiều chiều.

```
"con mèo"   → [0.12, -0.43, 0.88, ..., 0.31]  (1536 chiều)
"con chó"   → [0.15, -0.40, 0.85, ..., 0.29]  (gần "con mèo")
"xe ô tô"   → [-0.55, 0.78, 0.10, ..., -0.22] (xa "con mèo")
```

**Ý tưởng cốt lõi**: Text có nghĩa tương tự → vector gần nhau trong không gian.

---

## 2. Tại Sao Cần Embedding?

Máy tính không hiểu chữ, chỉ hiểu số. Để tìm kiếm theo nghĩa (semantic search), cần:
1. Convert text → vector (embedding)
2. So sánh khoảng cách giữa các vector
3. Vector càng gần → nghĩa càng tương tự

**Ứng dụng**:
- 🔍 **Semantic search**: tìm theo nghĩa, không phải keyword
- 📚 **RAG**: retrieve document liên quan đến câu hỏi
- 🎯 **Recommendation**: gợi ý sản phẩm tương tự
- 🏷️ **Classification**: phân loại text
- 🔬 **Clustering**: gom nhóm document

---

## 3. Embedding Model

### 3.1 Các Model Phổ Biến

| Model | Provider | Dim | Đặc điểm |
|-------|----------|-----|----------|
| `text-embedding-3-small` | OpenAI | 1536 | Rẻ, nhanh, đủ dùng |
| `text-embedding-3-large` | OpenAI | 3072 | Chất lượng cao nhất |
| `voyage-3` | Voyage AI | 1024 | Tốt cho RAG |
| `bge-large-en` | BAAI | 1024 | Open source |
| `all-MiniLM-L6-v2` | HuggingFace | 384 | Free, chạy local |

### 3.2 Sử Dụng OpenAI Embedding

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Embed 1 câu
vector = embeddings.embed_query("Xin chào")
print(f"Số chiều: {len(vector)}")  # 1536
print(f"5 giá trị đầu: {vector[:5]}")

# Embed nhiều document cùng lúc
documents = [
    "Mèo là động vật có vú",
    "Chó là bạn của con người",
    "Xe ô tô chạy bằng xăng"
]
doc_vectors = embeddings.embed_documents(documents)
print(f"Số document: {len(doc_vectors)}")  # 3
print(f"Mỗi vector có {len(doc_vectors[0])} chiều")  # 1536
```

### 3.3 Free Embedding Local

```python
# Cài: pip install sentence-transformers
from langchain_huggingface import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)
vector = embeddings.embed_query("Hello world")
```

---

## 4. Cosine Similarity - Đo Khoảng Cách Vector

### 4.1 Công Thức

**Cosine similarity** đo góc giữa 2 vector:

```
cos(A, B) = (A · B) / (|A| × |B|)
```

- **= 1**: hoàn toàn giống nhau (cùng hướng)
- **= 0**: không liên quan (vuông góc)
- **= -1**: hoàn toàn ngược nhau

### 4.2 Code Implementation

```python
import numpy as np

def cosine_similarity(a, b):
    a = np.array(a)
    b = np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Ví dụ
v1 = embeddings.embed_query("Con mèo dễ thương")
v2 = embeddings.embed_query("Mèo con đáng yêu")
v3 = embeddings.embed_query("Trời mưa to")

print(f"Mèo vs Mèo: {cosine_similarity(v1, v2):.4f}")  # ~0.85
print(f"Mèo vs Mưa: {cosine_similarity(v1, v3):.4f}")  # ~0.30
```

### 4.3 Các Distance Khác

```python
# Euclidean distance - khoảng cách thẳng
def euclidean(a, b):
    return np.linalg.norm(np.array(a) - np.array(b))

# Dot product - nhân vô hướng
def dot_product(a, b):
    return np.dot(a, b)
```

**Khi nào dùng cái nào**:
- **Cosine**: phổ biến nhất, dùng cho text
- **Dot product**: nhanh hơn nếu vector đã normalize
- **Euclidean**: hiếm dùng với text

---

## 5. Vector Database

### 5.1 Tại Sao Cần Vector DB?

Nếu chỉ có vài chục document, có thể tính cosine với từng vector. Nhưng với **hàng triệu vector**, tính tuần tự rất chậm.

**Vector database** tối ưu việc:
- Lưu trữ vector hiệu quả
- Tìm kiếm **Approximate Nearest Neighbor (ANN)** rất nhanh
- Filter theo metadata
- Update/delete vector

### 5.2 Các Vector DB Phổ Biến

| DB | Loại | Đặc điểm |
|----|------|----------|
| **Chroma** | Local/Cloud | Dễ dùng, free, tốt cho dev |
| **FAISS** | Local | Của Facebook, rất nhanh |
| **Pinecone** | Cloud | Managed, scale tốt |
| **Weaviate** | Self-host/Cloud | Open source, hybrid search |
| **Milvus** | Self-host | Production-grade |
| **Qdrant** | Self-host/Cloud | Rust, nhanh |
| **pgvector** | PostgreSQL extension | Tích hợp DB SQL |

### 5.3 Demo Với Chroma

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_core.documents import Document

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Tạo documents
docs = [
    Document(page_content="Python là ngôn ngữ lập trình phổ biến"),
    Document(page_content="LangChain giúp xây dựng ứng dụng AI"),
    Document(page_content="Cà phê Việt Nam rất nổi tiếng"),
    Document(page_content="Phở là món ăn truyền thống Việt Nam"),
]

# Tạo vector store
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    persist_directory="./chroma_db"  # Lưu xuống disk
)

# Tìm kiếm
query = "Lập trình AI"
results = vectorstore.similarity_search(query, k=2)

for i, doc in enumerate(results):
    print(f"{i+1}. {doc.page_content}")

# Tìm kiếm với score
results_with_score = vectorstore.similarity_search_with_score(query, k=2)
for doc, score in results_with_score:
    print(f"Score: {score:.4f} - {doc.page_content}")
```

### 5.4 Demo Với FAISS

```python
from langchain_community.vectorstores import FAISS

# Tạo
vectorstore = FAISS.from_documents(docs, embeddings)

# Lưu
vectorstore.save_local("./faiss_index")

# Load lại
loaded = FAISS.load_local(
    "./faiss_index", 
    embeddings, 
    allow_dangerous_deserialization=True
)

# Tìm kiếm
results = loaded.similarity_search("AI framework", k=2)
```

---

## 6. Metadata Filtering

Embedding tốt cho semantic, nhưng đôi khi cần filter chính xác.

```python
docs = [
    Document(
        page_content="LangChain version 0.1 release",
        metadata={"source": "blog", "date": "2024-01-15", "category": "tech"}
    ),
    Document(
        page_content="Phở Hà Nội ngon nhất",
        metadata={"source": "food_blog", "date": "2024-03-20", "category": "food"}
    ),
]

vectorstore = Chroma.from_documents(docs, embeddings)

# Filter theo metadata
results = vectorstore.similarity_search(
    "version release",
    k=5,
    filter={"category": "tech"}  # Chỉ lấy tech
)
```

---

## 7. Indexing Strategies (ANN Algorithms)

Vector DB dùng các thuật toán để search nhanh:

### 7.1 HNSW (Hierarchical Navigable Small World)
- Phổ biến nhất hiện nay
- Cân bằng tốt giữa tốc độ và accuracy
- Dùng trong: Chroma, Weaviate, Qdrant, pgvector

### 7.2 IVF (Inverted File Index)
- Chia vector thành cluster
- Dùng trong: FAISS, Milvus

### 7.3 Flat (Brute Force)
- Tính cosine với mọi vector
- Chính xác 100% nhưng chậm
- Dùng khi data < 10K vectors

```python
# FAISS - chọn index type
from langchain_community.vectorstores import FAISS

# Flat - exact, dùng cho data nhỏ
vectorstore = FAISS.from_documents(docs, embeddings, distance_strategy="COSINE")

# HNSW - approximate, fast
# (Cần config sâu hơn, thường để default)
```

---

## 8. Chunking - Chia Nhỏ Document

Embedding không hiệu quả với text quá dài. Cần chia thành chunk nhỏ.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

long_text = """
Python là ngôn ngữ lập trình phổ biến. Nó được tạo bởi Guido van Rossum 
vào năm 1991. Python có cú pháp đơn giản, dễ học. Python được dùng nhiều 
trong AI, web development, data science.

LangChain là framework AI mạnh mẽ. Nó cho phép xây dựng ứng dụng LLM 
nhanh chóng. LangChain hỗ trợ nhiều LLM provider khác nhau.
"""

splitter = RecursiveCharacterTextSplitter(
    chunk_size=100,      # Mỗi chunk ~100 ký tự
    chunk_overlap=20,    # Overlap 20 ký tự giữa các chunk
)

chunks = splitter.split_text(long_text)
for i, chunk in enumerate(chunks):
    print(f"--- Chunk {i+1} ({len(chunk)} chars) ---")
    print(chunk)
```

(Sẽ học sâu hơn ở Phase 2 - RAG)

---

## 9. Demo End-to-End: Mini Semantic Search

```python
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_core.documents import Document

# 1. Chuẩn bị data
knowledge_base = [
    "Python là ngôn ngữ lập trình interpreted, high-level, có cú pháp đơn giản.",
    "JavaScript là ngôn ngữ chính cho web development front-end.",
    "Java là ngôn ngữ OOP, chạy trên JVM, phổ biến cho enterprise.",
    "Rust là ngôn ngữ system programming an toàn về memory.",
    "Go (Golang) được Google tạo ra, tốt cho concurrent programming.",
    "Phở là món ăn truyền thống của Việt Nam, ăn kèm thịt bò hoặc gà.",
    "Bún chả là đặc sản Hà Nội, gồm thịt nướng và nước mắm chua ngọt.",
    "Cà phê sữa đá Việt Nam pha bằng phin, nổi tiếng thế giới.",
]

docs = [Document(page_content=text) for text in knowledge_base]

# 2. Tạo vector store
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(docs, embeddings)

# 3. Search
queries = [
    "Tôi muốn học ngôn ngữ lập trình cho web",
    "Món ăn nào của Việt Nam?",
    "Ngôn ngữ nào an toàn nhất?"
]

for query in queries:
    print(f"\n🔍 Query: {query}")
    results = vectorstore.similarity_search_with_score(query, k=2)
    for doc, score in results:
        print(f"   [{score:.3f}] {doc.page_content}")
```

**Output mẫu**:
```
🔍 Query: Tôi muốn học ngôn ngữ lập trình cho web
   [0.345] JavaScript là ngôn ngữ chính cho web development front-end.
   [0.567] Python là ngôn ngữ lập trình interpreted, high-level...

🔍 Query: Món ăn nào của Việt Nam?
   [0.412] Phở là món ăn truyền thống của Việt Nam...
   [0.445] Bún chả là đặc sản Hà Nội...
```

---

## 10. Tips Quan Trọng

### 10.1 Embedding Cùng Loại
❌ Không trộn embedding từ 2 model khác nhau:
```python
# SAI - trộn 2 embedding
v1 = openai_embed.embed_query("Hello")
v2 = bge_embed.embed_query("Xin chào")
# Không thể so sánh!
```

### 10.2 Normalize Vector
Một số model trả vector chưa normalize. Khi đó:
```python
import numpy as np

def normalize(v):
    return v / np.linalg.norm(v)

# Sau khi normalize, cosine = dot product (nhanh hơn)
```

### 10.3 Caching Embedding
Embedding tốn tiền. Cache lại để không phải embed lại text cũ:
```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain.storage import LocalFileStore

store = LocalFileStore("./cache/")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    embeddings, store, namespace="openai-small"
)
# Lần đầu: gọi API. Lần sau: lấy từ cache
```

---

## 11. Bài Tập Thực Hành

### Bài 1: Semantic Search
Cho list 20 mô tả sản phẩm. Build semantic search để user nhập query và trả về top 3 sản phẩm liên quan nhất.

### Bài 2: So Sánh Embedding Models
So sánh 3 embedding models:
- OpenAI text-embedding-3-small
- HuggingFace all-MiniLM-L6-v2
- HuggingFace BGE
Đo: tốc độ, chất lượng search, chi phí.

### Bài 3: Vector DB Comparison
Implement cùng 1 use case với Chroma, FAISS, và PostgreSQL pgvector. So sánh.

---

## 12. Checklist Kiến Thức

- [ ] Hiểu embedding là gì và tại sao cần
- [ ] Biết cách gọi embedding API
- [ ] Tính được cosine similarity
- [ ] Hiểu vector database hoạt động ra sao
- [ ] Dùng được Chroma, FAISS cơ bản
- [ ] Hiểu metadata filtering
- [ ] Biết về ANN algorithms (HNSW, IVF)
- [ ] Hiểu khi nào cần normalize vector

➡️ **Tiếp theo**: [Phase 1 - LangChain Core](../01-LangChain-Core/01-Gioi-Thieu-LangChain.md)
