# Tổng Quan RAG (Retrieval-Augmented Generation)

## 1. RAG Là Gì?

**RAG = Retrieval-Augmented Generation** = LLM + Tìm kiếm thông tin.

Thay vì LLM trả lời dựa trên kiến thức được train (có thể outdated hoặc không có), RAG cho phép:
1. **Retrieve** (truy xuất): tìm kiếm document liên quan đến câu hỏi
2. **Augment** (bổ sung): chèn document vào prompt
3. **Generate** (sinh ra): LLM trả lời dựa trên document

```
User Question → [Retrieve] → Relevant Docs → [Augment Prompt] → LLM → Answer
```

---

## 2. Tại Sao Cần RAG?

### 2.1 Vấn Đề Của LLM Thuần

❌ **Knowledge cutoff**: GPT-4 chỉ biết data đến 2024
❌ **Hallucination**: bịa thông tin không có
❌ **Không biết private data**: không biết tài liệu nội bộ công ty bạn
❌ **Không update**: muốn thêm kiến thức mới phải re-train (rất tốn)

### 2.2 RAG Giải Quyết

✅ **Thông tin mới**: chỉ cần update vector DB
✅ **Giảm hallucination**: LLM trả lời dựa trên fact
✅ **Private data**: dùng được tài liệu công ty
✅ **Cite nguồn**: LLM có thể chỉ ra source

### 2.3 Khi Nào Dùng RAG vs Fine-tuning?

| Case | RAG | Fine-tuning |
|------|-----|-------------|
| Thêm knowledge | ✅ Tốt | ⚠️ Đắt, không real-time |
| Học style/format | ⚠️ Hạn chế | ✅ Tốt |
| Update thường xuyên | ✅ Dễ | ❌ Phải train lại |
| Cost | 💵 Embedding + storage | 💵💵💵 Training |
| Setup time | ⚡ Vài giờ | ⏳ Vài ngày |

➡️ **Khuyến nghị**: Bắt đầu với RAG, fine-tune chỉ khi RAG không đủ.

---

## 3. Kiến Trúc RAG Cơ Bản

```
┌─────────────────────────────────────────────────────┐
│  PHASE 1: INDEXING (Offline)                        │
│                                                     │
│  Documents → Load → Chunk → Embed → Vector DB       │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  PHASE 2: RETRIEVAL & GENERATION (Online)           │
│                                                     │
│  User Question                                      │
│       ↓                                             │
│  Embed Question                                     │
│       ↓                                             │
│  Search Vector DB → Top K relevant chunks           │
│       ↓                                             │
│  Build Prompt with chunks as context                │
│       ↓                                             │
│  LLM → Answer + Citations                           │
└─────────────────────────────────────────────────────┘
```

---

## 4. Các Thành Phần Của RAG

### 4.1 Document Loaders
Load text từ PDF, web, CSV, database, ...
→ [Document Loaders](./02-Document-Loaders.md)

### 4.2 Text Splitters (Chunking)
Chia document thành chunk nhỏ.
→ [Chunking Strategy](./03-Chunking-Strategy.md)

### 4.3 Embeddings
Convert text → vector.
→ [Foundation: Embeddings](../00-Foundation/03-Embeddings-Vectors.md)

### 4.4 Vector Store
Lưu trữ và search vector.
→ [Vector Stores](./04-Vector-Stores.md)

### 4.5 Retriever
Interface để query vector store + reranker.
→ [Retrievers](./05-Retrievers.md)

### 4.6 LLM
Generate answer.

### 4.7 (Advanced) Reranking, Query Rewriting, Compression
Tối ưu chất lượng retrieval.
→ Sẽ học ở các doc sau

---

## 5. Demo RAG Đơn Giản (Hello World)

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

load_dotenv()

# ========== PHASE 1: INDEXING ==========

# 1. Chuẩn bị documents (giả lập đã loaded + split)
documents = [
    Document(
        page_content="LangChain được phát triển bởi Harrison Chase vào năm 2022.",
        metadata={"source": "wiki-langchain"}
    ),
    Document(
        page_content="LangChain hỗ trợ Python và JavaScript. Có thể tích hợp với 60+ LLM providers.",
        metadata={"source": "wiki-langchain"}
    ),
    Document(
        page_content="OpenAI GPT-4o là model multimodal, hỗ trợ text, image và audio.",
        metadata={"source": "wiki-openai"}
    ),
    Document(
        page_content="Vector database giúp lưu trữ và search embeddings hiệu quả.",
        metadata={"source": "wiki-vectordb"}
    ),
    Document(
        page_content="RAG kết hợp retrieval và LLM để tạo câu trả lời chính xác.",
        metadata={"source": "wiki-rag"}
    ),
]

# 2. Tạo vector store
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(
    documents=documents,
    embedding=embeddings,
)

# 3. Tạo retriever
retriever = vectorstore.as_retriever(search_kwargs={"k": 2})

# ========== PHASE 2: RETRIEVAL & GENERATION ==========

# 4. Tạo prompt template
template = """Trả lời câu hỏi DỰA TRÊN context sau. 
Nếu không có thông tin trong context, hãy nói "Tôi không biết".

Context:
{context}

Câu hỏi: {question}

Câu trả lời:"""

prompt = ChatPromptTemplate.from_template(template)

# 5. LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 6. Helper: format documents
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

# 7. RAG Chain với LCEL
rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough()
    }
    | prompt
    | llm
    | StrOutputParser()
)

# 8. Hỏi
questions = [
    "Ai tạo ra LangChain?",
    "LangChain hỗ trợ những ngôn ngữ nào?",
    "Vector database dùng để làm gì?",
    "Thủ đô Pháp là gì?",  # Không có trong context
]

for q in questions:
    print(f"\n❓ {q}")
    answer = rag_chain.invoke(q)
    print(f"💬 {answer}")
```

**Output mẫu**:
```
❓ Ai tạo ra LangChain?
💬 LangChain được phát triển bởi Harrison Chase vào năm 2022.

❓ Vector database dùng để làm gì?
💬 Vector database giúp lưu trữ và search embeddings hiệu quả.

❓ Thủ đô Pháp là gì?
💬 Tôi không biết. (vì không có trong context)
```

---

## 6. RAG Với Citations

```python
def format_docs_with_source(docs):
    formatted = []
    for i, doc in enumerate(docs, 1):
        source = doc.metadata.get("source", "unknown")
        formatted.append(f"[Source {i}: {source}]\n{doc.page_content}")
    return "\n\n".join(formatted)

template_with_citation = """Trả lời dựa trên context. CHỈ DẪN nguồn ở cuối câu trả lời [Source 1], [Source 2], ...

Context:
{context}

Câu hỏi: {question}

Câu trả lời (có citation):"""

prompt = ChatPromptTemplate.from_template(template_with_citation)

rag_chain = (
    {
        "context": retriever | format_docs_with_source,
        "question": RunnablePassthrough()
    }
    | prompt
    | llm
    | StrOutputParser()
)

print(rag_chain.invoke("LangChain hỗ trợ những ngôn ngữ nào?"))
# "LangChain hỗ trợ Python và JavaScript [Source 1: wiki-langchain]."
```

---

## 7. RAG Với Conversation History

```python
from langchain_core.messages import HumanMessage, AIMessage

template = """Trả lời câu hỏi dựa trên context và lịch sử hội thoại.

Context:
{context}

Lịch sử hội thoại:
{chat_history}

Câu hỏi: {question}"""

prompt = ChatPromptTemplate.from_template(template)

chat_history = []

def chat(question):
    # Format history
    history_str = "\n".join([
        f"User: {m.content}" if isinstance(m, HumanMessage) else f"AI: {m.content}"
        for m in chat_history
    ])
    
    # Retrieve
    docs = retriever.invoke(question)
    context = format_docs(docs)
    
    # Generate
    response = llm.invoke(prompt.format(
        context=context,
        chat_history=history_str,
        question=question
    ))
    
    # Update history
    chat_history.append(HumanMessage(content=question))
    chat_history.append(AIMessage(content=response.content))
    
    return response.content

# Demo
print(chat("Ai tạo ra LangChain?"))
# "Harrison Chase tạo ra LangChain vào năm 2022."

print(chat("Anh ấy còn làm gì khác?"))
# (LLM hiểu "anh ấy" = Harrison Chase từ history)
```

---

## 8. Vấn Đề Của RAG Cơ Bản

### 8.1 Retrieval Quality
- Chunk quá nhỏ → mất context
- Chunk quá lớn → embedding không chính xác
- Top-K quá ít → bỏ sót
- Top-K quá nhiều → noise

### 8.2 Hallucination Trong Context
LLM vẫn có thể bịa thông tin **không có trong context**.

**Giải pháp**:
- Prompt rõ: "CHỈ trả lời nếu có trong context"
- Yêu cầu cite source
- Verification step

### 8.3 Khi User Hỏi Mơ Hồ
"Cái đó là gì?" - không biết "cái đó" là gì để search.

**Giải pháp**:
- Query rewriting với history
- Multi-query expansion

### 8.4 Multi-Hop Reasoning
Câu hỏi cần combine nhiều document:
"So sánh giá iPhone 15 và Samsung S24"

**Giải pháp**:
- Agent-based RAG
- Self-querying retriever

---

## 9. Advanced RAG Techniques (Sẽ Học Sâu)

| Technique | Mục đích |
|-----------|----------|
| **Hybrid Search** | Kết hợp semantic + keyword (BM25) |
| **Reranking** | Sắp xếp lại kết quả với cross-encoder |
| **Query Rewriting** | LLM viết lại query để search tốt hơn |
| **HyDE** | Hypothetical Document Embedding |
| **Multi-Query** | Generate nhiều variants của query |
| **Contextual Compression** | Cắt phần không liên quan |
| **Parent Document Retriever** | Search chunk nhỏ, trả về chunk lớn |
| **Self-Querying** | LLM tự generate filter metadata |

---

## 10. Đo Lường Chất Lượng RAG

### 10.1 Retrieval Metrics
- **Recall@K**: bao nhiêu doc đúng trong top K
- **MRR**: Mean Reciprocal Rank
- **NDCG**: Normalized Discounted Cumulative Gain

### 10.2 Generation Metrics
- **Faithfulness**: answer có dựa trên context không
- **Answer Relevancy**: answer có liên quan câu hỏi không
- **Context Precision/Recall**: chất lượng context

### 10.3 Tools
- **RAGAS**: framework eval RAG
- **LangSmith**: trace + eval
- **TruLens**: monitor RAG

→ Sẽ học sâu ở [Phase 6 - Evaluation](../06-Evaluation/01-RAG-Evaluation.md)

---

## 11. RAG Workflow Đầy Đủ (Roadmap)

```
1. Document Loaders (load từ nguồn nào)
   ↓
2. Chunking (chia thế nào)
   ↓
3. Embeddings (model nào)
   ↓
4. Vector Store (lưu ở đâu)
   ↓
5. Retriever (search như thế nào)
   ↓
6. (Optional) Query Rewriting
   ↓
7. (Optional) Hybrid Search
   ↓
8. (Optional) Reranking
   ↓
9. (Optional) Contextual Compression
   ↓
10. Prompt với context
    ↓
11. LLM Generation
    ↓
12. Citation + Verification
    ↓
13. Evaluation
```

→ Mỗi step sẽ có doc riêng trong Phase 2 này.

---

## 12. Checklist

- [ ] Hiểu RAG là gì và khi nào dùng
- [ ] So sánh được RAG vs Fine-tuning
- [ ] Hiểu kiến trúc Indexing + Retrieval
- [ ] Build được mini RAG end-to-end
- [ ] Thêm citation vào answer
- [ ] Tích hợp conversation history
- [ ] Hiểu các vấn đề của RAG cơ bản

➡️ **Tiếp theo**: [Document Loaders](./02-Document-Loaders.md)
