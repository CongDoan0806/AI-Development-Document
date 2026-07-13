# Contextual Compression - Nén Context

## 1. Vấn Đề

Retrieval trả về docs **dài**, nhưng chỉ một phần liên quan đến query.

**Ví dụ**:
- Query: "Ai tạo ra Python?"
- Retrieved doc: 1000 từ về lịch sử lập trình
- Chỉ 1 câu liên quan: "Python được Guido van Rossum tạo ra năm 1991"

→ Truyền cả doc vào LLM = lãng phí token + có noise.

**Giải pháp**: **Contextual Compression** - lọc/cắt chỉ phần liên quan.

---

## 2. ContextualCompressionRetriever

Wrapper bọc retriever cơ sở + compressor:

```python
from langchain.retrievers import ContextualCompressionRetriever

compression_retriever = ContextualCompressionRetriever(
    base_compressor=...,
    base_retriever=...,
)

docs = compression_retriever.invoke("query")
```

---

## 3. LLMChainExtractor - LLM Extract Phần Liên Quan

```python
from langchain.retrievers.document_compressors import LLMChainExtractor
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

compressor = LLMChainExtractor.from_llm(llm)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(),
)

# Input: doc 1000 từ
# Output: chỉ 1-2 câu liên quan
docs = compression_retriever.invoke("Ai tạo ra Python?")
for d in docs:
    print(d.page_content)
```

**Hoạt động**:
- Với mỗi doc, gọi LLM với prompt: "Extract phần liên quan đến query"
- Nếu doc không liên quan → loại bỏ

**Pros**: Chính xác cao
**Cons**: Tốn N LLM calls (N = số docs)

---

## 4. LLMChainFilter - Lọc Doc

Không cắt content, chỉ lọc bỏ docs không liên quan.

```python
from langchain.retrievers.document_compressors import LLMChainFilter

filter_compressor = LLMChainFilter.from_llm(llm)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=filter_compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10}),
)

# 10 docs → có thể chỉ 3-5 docs liên quan
docs = compression_retriever.invoke("query")
```

---

## 5. EmbeddingsFilter - Filter Bằng Embedding

Không gọi LLM, dùng embedding similarity để filter:

```python
from langchain.retrievers.document_compressors import EmbeddingsFilter
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

embeddings_filter = EmbeddingsFilter(
    embeddings=embeddings,
    similarity_threshold=0.76,  # Chỉ giữ doc có cosine > 0.76
)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=embeddings_filter,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)

docs = compression_retriever.invoke("query")
```

**Pros**: Nhanh, không tốn LLM
**Cons**: Threshold khó tune

---

## 6. EmbeddingsRedundantFilter - Loại Bỏ Trùng

Loại docs trùng lặp về nội dung:

```python
from langchain_community.document_transformers import EmbeddingsRedundantFilter

redundant_filter = EmbeddingsRedundantFilter(
    embeddings=embeddings,
    similarity_threshold=0.95,  # Hai doc gần nhau > 0.95 → loại 1
)
```

---

## 7. DocumentCompressorPipeline - Combine Multiple

```python
from langchain.retrievers.document_compressors import DocumentCompressorPipeline
from langchain_text_splitters import CharacterTextSplitter

# Pipeline:
# 1. Split docs thành chunks nhỏ hơn
# 2. Filter redundant
# 3. Filter by embedding similarity
# 4. Final extract with LLM

splitter = CharacterTextSplitter(chunk_size=300, chunk_overlap=0, separator=". ")
redundant_filter = EmbeddingsRedundantFilter(embeddings=embeddings)
relevance_filter = EmbeddingsFilter(embeddings=embeddings, similarity_threshold=0.76)

pipeline_compressor = DocumentCompressorPipeline(
    transformers=[splitter, redundant_filter, relevance_filter]
)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=pipeline_compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)
```

---

## 8. Cohere Rerank Như Compressor

Cohere Rerank cũng là compressor (vừa rank vừa lọc):

```python
from langchain_cohere import CohereRerank

rerank_compressor = CohereRerank(top_n=3)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=rerank_compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)
```

---

## 9. LongContextReorder - Sắp Xếp Cho LLM

Vấn đề **"Lost in the Middle"**: LLM kém attention ở giữa context dài.
→ Đặt doc quan trọng ở **đầu** và **cuối**.

```python
from langchain_community.document_transformers import LongContextReorder

reorder = LongContextReorder()

# Docs: [d1, d2, d3, d4, d5] (theo relevance giảm dần)
# Sau reorder: [d1, d3, d5, d4, d2] (quan trọng ở 2 đầu)
reordered_docs = reorder.transform_documents(docs)
```

### Dùng Trong Chain

```python
from langchain_core.runnables import RunnableLambda

def reorder_docs(docs):
    return LongContextReorder().transform_documents(docs)

chain = (
    retriever
    | RunnableLambda(reorder_docs)
    | format_docs
    | prompt
    | llm
)
```

---

## 10. Custom Compressor

```python
from langchain.retrievers.document_compressors.base import BaseDocumentCompressor
from langchain_core.documents import Document
from typing import Sequence

class TruncateCompressor(BaseDocumentCompressor):
    """Truncate mỗi doc xuống max_length"""
    
    max_length: int = 500
    
    def compress_documents(
        self,
        documents: Sequence[Document],
        query: str,
        callbacks=None,
    ) -> Sequence[Document]:
        return [
            Document(
                page_content=d.page_content[:self.max_length],
                metadata=d.metadata,
            )
            for d in documents
        ]

# Use
compressor = TruncateCompressor(max_length=300)
retriever_truncated = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(),
)
```

---

## 11. Demo Pipeline Hoàn Chỉnh

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_text_splitters import CharacterTextSplitter
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import (
    DocumentCompressorPipeline,
    EmbeddingsFilter,
    LLMChainExtractor,
)
from langchain_community.document_transformers import (
    EmbeddingsRedundantFilter,
    LongContextReorder,
)

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(embedding_function=embeddings, persist_directory="./db")

# Compressor pipeline
compressors = DocumentCompressorPipeline(transformers=[
    CharacterTextSplitter(chunk_size=200, chunk_overlap=0, separator=". "),
    EmbeddingsRedundantFilter(embeddings=embeddings, similarity_threshold=0.95),
    EmbeddingsFilter(embeddings=embeddings, similarity_threshold=0.76),
    LongContextReorder(),
])

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressors,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20}),
)

# Use
docs = compression_retriever.invoke("AI là gì?")
print(f"Final docs: {len(docs)}")
for d in docs:
    print(f"- {d.page_content[:100]}")
```

---

## 12. Token-Aware Compression

```python
import tiktoken

def smart_truncate(docs, max_total_tokens=2000):
    """Giữ docs sao cho tổng không quá max_total_tokens"""
    enc = tiktoken.get_encoding("cl100k_base")
    result = []
    total = 0
    
    for doc in docs:
        tokens = len(enc.encode(doc.page_content))
        if total + tokens > max_total_tokens:
            # Truncate doc cuối
            remaining = max_total_tokens - total
            text = enc.decode(enc.encode(doc.page_content)[:remaining])
            result.append(Document(page_content=text, metadata=doc.metadata))
            break
        result.append(doc)
        total += tokens
    
    return result
```

---

## 13. Đo Lường Compression

```python
def measure_compression(retriever, comp_retriever, queries):
    original_tokens = 0
    compressed_tokens = 0
    enc = tiktoken.get_encoding("cl100k_base")
    
    for q in queries:
        orig_docs = retriever.invoke(q)
        comp_docs = comp_retriever.invoke(q)
        
        original_tokens += sum(len(enc.encode(d.page_content)) for d in orig_docs)
        compressed_tokens += sum(len(enc.encode(d.page_content)) for d in comp_docs)
    
    reduction = (original_tokens - compressed_tokens) / original_tokens
    print(f"Original: {original_tokens} tokens")
    print(f"Compressed: {compressed_tokens} tokens")
    print(f"Reduction: {reduction:.1%}")
```

---

## 14. Trade-offs

| Method | Quality | Speed | Cost |
|--------|---------|-------|------|
| EmbeddingsFilter | Medium | Fast | $ (embed) |
| EmbeddingsRedundantFilter | High | Fast | $ |
| LLMChainExtractor | High | Slow | $$$ (LLM/doc) |
| LLMChainFilter | High | Slow | $$$ |
| CohereRerank | Very High | Medium | $$ |
| LongContextReorder | - | Free | $ |

---

## 15. Best Practices

✅ **Pipeline**: split → redundant filter → relevance filter → reorder

✅ **Retrieve 20, compress xuống 5**: cân bằng coverage và noise

✅ **EmbeddingsFilter trước, LLM Extract sau**: nhanh + chính xác

✅ **LongContextReorder cho context >= 5 docs**

✅ **Test với golden set**: đảm bảo không lose critical info

---

## 16. Khi Nào Cần Compression?

### Cần
- ✅ Docs dài (1000+ words)
- ✅ Top K cao (10+)
- ✅ Context window LLM giới hạn
- ✅ Cost concern (giảm input tokens)

### Có Thể Skip
- ❌ Docs đã chunk nhỏ (300-500 tokens)
- ❌ Top K nhỏ (3-5)
- ❌ LLM context lớn (Claude 1M, Gemini 2M)

---

## 17. Bài Tập

### Bài 1: Compression Benchmark
So sánh:
- No compression
- EmbeddingsFilter only
- Full pipeline (split + redundant + relevance + reorder)
- Cohere Rerank

Đo: tokens saved, quality (faithfulness), latency, cost.

### Bài 2: Custom Compressor
Build compressor extract **chỉ những câu chứa entity** từ query (dùng NER).

### Bài 3: Adaptive Compression
Tự động compress nếu total tokens > threshold (vd: 4000 tokens), không compress nếu đủ ngắn.

---

## 18. Checklist

- [ ] ContextualCompressionRetriever interface
- [ ] LLMChainExtractor
- [ ] LLMChainFilter
- [ ] EmbeddingsFilter
- [ ] EmbeddingsRedundantFilter
- [ ] DocumentCompressorPipeline
- [ ] LongContextReorder
- [ ] Custom compressor

➡️ **Tiếp theo**: [RAG Chain End-to-End](./10-RAG-Chain-End-to-End.md)
