# Chunking Strategy - Chia Nhỏ Document

## 1. Tại Sao Cần Chunking?

### 1.1 Embedding Có Giới Hạn
Embedding model có max token (thường 512-8192). Text quá dài → bị truncate hoặc lỗi.

### 1.2 Context Window Hạn Chế
Nếu cho LLM cả document → tốn token, đắt, có thể vượt context.

### 1.3 Retrieval Quality
- Chunk quá lớn → embedding mất ý → tìm kém chính xác
- Chunk quá nhỏ → mất context → LLM không hiểu

### 1.4 Granularity
Search chunk nhỏ → match chính xác câu hỏi cụ thể.

---

## 2. Các Tham Số Chunking

```python
chunk_size = 1000      # Kích thước mỗi chunk (token hoặc char)
chunk_overlap = 200    # Overlap giữa các chunk
```

**Tại sao cần overlap?**
- Tránh cắt ngang câu/ý quan trọng
- Khi câu hỏi nằm ở ranh giới 2 chunk, vẫn match được

```
[Chunk 1: ........overlap]
                 [overlap.........Chunk 2: ........overlap]
                                            [overlap.........Chunk 3]
```

---

## 3. CharacterTextSplitter - Cơ Bản

Chia theo ký tự đơn thuần.

```python
from langchain_text_splitters import CharacterTextSplitter

text = """LangChain là framework AI. LangChain hỗ trợ nhiều LLM.

RAG là kỹ thuật quan trọng. RAG kết hợp retrieval và generation.

Vector database lưu embeddings. Phổ biến là Chroma, FAISS."""

splitter = CharacterTextSplitter(
    separator="\n\n",      # Tách theo paragraph
    chunk_size=100,
    chunk_overlap=20,
    length_function=len,   # Đếm theo ký tự
)

chunks = splitter.split_text(text)
for i, c in enumerate(chunks):
    print(f"--- Chunk {i+1} ({len(c)}) ---\n{c}\n")
```

**Hạn chế**: chỉ tách theo 1 separator → có thể cắt mất context.

---

## 4. RecursiveCharacterTextSplitter ⭐ (Phổ Biến Nhất)

Thử nhiều separator theo thứ tự ưu tiên:

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=[
        "\n\n",   # 1. Tách paragraph trước
        "\n",     # 2. Sau đó tách dòng
        ". ",     # 3. Tách câu
        " ",      # 4. Tách từ
        "",       # 5. Cuối cùng tách ký tự
    ],
    length_function=len,
)

chunks = splitter.split_text(long_text)
```

**Tại sao "recursive"?**
- Thử `\n\n` trước, nếu chunk vẫn quá lớn → thử `\n`
- Cứ thế cho đến khi chunk đạt chunk_size

➡️ **Đây là default cho hầu hết use cases**.

---

## 5. Token-Based Splitting

Thay vì đếm ký tự, đếm token (chính xác hơn cho LLM).

### 5.1 TokenTextSplitter

```python
from langchain_text_splitters import TokenTextSplitter

splitter = TokenTextSplitter(
    chunk_size=500,    # 500 tokens
    chunk_overlap=50,
    encoding_name="cl100k_base",  # Encoding của GPT-4
)
```

### 5.2 RecursiveCharacterTextSplitter Với Token Counter

```python
import tiktoken

def token_count(text):
    enc = tiktoken.get_encoding("cl100k_base")
    return len(enc.encode(text))

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    length_function=token_count,  # Đo theo token
)
```

---

## 6. Language-Specific Splitter

### 6.1 Code (Python, JavaScript, ...)

```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,
    Language,
)

python_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=500,
    chunk_overlap=50,
)

js_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.JS,
    chunk_size=500,
)

# Các language khác:
# Language.CPP, JAVA, GO, RUST, PHP, RUBY, HTML, MARKDOWN, ...
```

**Python splitter dùng separators**:
```
["\nclass ", "\ndef ", "\n\tdef ", "\n\n", "\n", " ", ""]
```
→ Cắt ưu tiên ở ranh giới class/function.

### 6.2 Markdown

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

markdown = """
# Chapter 1
## Section 1.1
Content 1.1...

## Section 1.2
Content 1.2...

# Chapter 2
Content...
"""

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "Chapter"),
        ("##", "Section"),
    ]
)

chunks = splitter.split_text(markdown)
for c in chunks:
    print(f"Metadata: {c.metadata}")
    print(f"Content: {c.page_content[:50]}")
```

Output:
```
Metadata: {"Chapter": "Chapter 1", "Section": "Section 1.1"}
Content: Content 1.1...
```

### 6.3 HTML

```python
from langchain_text_splitters import HTMLHeaderTextSplitter

html_splitter = HTMLHeaderTextSplitter(
    headers_to_split_on=[
        ("h1", "Header 1"),
        ("h2", "Header 2"),
    ]
)

chunks = html_splitter.split_text(html_string)
```

---

## 7. Semantic Chunking ⭐ (Advanced)

Chia theo **nghĩa** thay vì kích thước. Embedding các câu, chỗ nào nghĩa thay đổi nhiều → cắt.

```python
# pip install langchain-experimental
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",  # "standard_deviation", "interquartile"
    breakpoint_threshold_amount=95,
)

chunks = splitter.create_documents([long_text])
```

**Ưu điểm**: chunk giữ nguyên ngữ nghĩa hoàn chỉnh.
**Nhược điểm**: chậm hơn, tốn embedding cost.

---

## 8. Chunking Theo Use Case

### 8.1 Q&A Chatbot
- Chunk size: 500-1000 chars
- Overlap: 100-200 chars
- Splitter: RecursiveCharacterTextSplitter

### 8.2 Tóm Tắt Document Dài
- Chunk size: 2000-4000 chars (lớn để LLM hiểu context)
- Overlap: 200-400
- Sau đó dùng map-reduce summarization

### 8.3 Code Search
- Chunk theo function/class
- Language-specific splitter

### 8.4 Long Document (Sách, Luận Án)
- Chunk theo chapter/section (markdown header)
- Hierarchical: chunk lớn cho overview + chunk nhỏ cho detail
- Parent-child retrieval

---

## 9. Parent-Child Chunking

**Ý tưởng**: search chunk nhỏ (precise) nhưng trả về chunk lớn (context).

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain_chroma import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter

# Parent (chunk lớn) - dùng để truyền vào LLM
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

# Child (chunk nhỏ) - dùng để search
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

# Vector store cho child
vectorstore = Chroma(embedding_function=OpenAIEmbeddings())

# Doc store cho parent
docstore = InMemoryStore()

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=docstore,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(documents)

# Khi search: tìm child chunk match → trả về parent chunk
results = retriever.invoke("câu hỏi")
```

---

## 10. Demo: So Sánh Các Splitter

```python
long_text = """
LangChain là framework AI mạnh mẽ. Nó được tạo ra bởi Harrison Chase vào năm 2022. 
LangChain hỗ trợ Python và JavaScript. Nó có hơn 60 LLM providers.

RAG là Retrieval-Augmented Generation. RAG kết hợp tìm kiếm và sinh text.
RAG giúp LLM trả lời chính xác hơn dựa trên dữ liệu thật.

Vector database lưu trữ embeddings. Chroma, FAISS, Pinecone là các vector DB phổ biến.
Vector DB dùng cosine similarity để tìm vector gần nhất.
"""

# 1. Character splitter
from langchain_text_splitters import CharacterTextSplitter
char_splitter = CharacterTextSplitter(chunk_size=100, chunk_overlap=20)
chunks_char = char_splitter.split_text(long_text)
print(f"Character: {len(chunks_char)} chunks")

# 2. Recursive splitter
from langchain_text_splitters import RecursiveCharacterTextSplitter
rec_splitter = RecursiveCharacterTextSplitter(chunk_size=100, chunk_overlap=20)
chunks_rec = rec_splitter.split_text(long_text)
print(f"Recursive: {len(chunks_rec)} chunks")

# 3. Token splitter
from langchain_text_splitters import TokenTextSplitter
token_splitter = TokenTextSplitter(chunk_size=30, chunk_overlap=5)
chunks_token = token_splitter.split_text(long_text)
print(f"Token: {len(chunks_token)} chunks")

for label, chunks in [("CHAR", chunks_char), ("REC", chunks_rec), ("TOKEN", chunks_token)]:
    print(f"\n=== {label} ===")
    for i, c in enumerate(chunks):
        print(f"  [{i}] ({len(c)}) {c[:60]}...")
```

---

## 11. Best Practices

### 11.1 Chọn Chunk Size

| Use case | Chunk Size | Overlap |
|----------|------------|---------|
| Short Q&A | 300-500 chars | 50-100 |
| General RAG | 1000 chars | 200 |
| Long-form summarization | 2000-4000 | 200-400 |
| Code search | 500 chars | 50 |

### 11.2 Test Chunk Size

```python
import matplotlib.pyplot as plt

chunk_sizes = [200, 500, 1000, 2000]
for size in chunk_sizes:
    splitter = RecursiveCharacterTextSplitter(chunk_size=size, chunk_overlap=50)
    chunks = splitter.split_documents(documents)
    
    # Đo recall trên test set
    recall = evaluate(chunks)  # Custom function
    print(f"Size {size}: {len(chunks)} chunks, Recall={recall}")
```

### 11.3 Giữ Metadata Qua Chunking

```python
documents = [
    Document(
        page_content="long text...",
        metadata={"source": "doc1.pdf", "author": "A"}
    ),
]

splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(documents)

# Tất cả chunks giữ metadata của doc gốc
for chunk in chunks:
    print(chunk.metadata)  # {"source": "doc1.pdf", "author": "A"}
```

### 11.4 Thêm Chunk Index Vào Metadata

```python
for i, chunk in enumerate(chunks):
    chunk.metadata["chunk_index"] = i
    chunk.metadata["chunk_total"] = len(chunks)
```

### 11.5 Avoid Mid-Sentence Cuts

Dùng `separators` tốt:
```python
splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", ". ", "! ", "? ", "; ", ", ", " ", ""]
)
```

### 11.6 Đặc Biệt Cho Tiếng Việt

```python
# Tiếng Việt: thêm separator "。" và xử lý punctuation Việt
vn_separators = ["\n\n", "\n", "。 ", ". ", "! ", "？ ", "? ", "; ", " ", ""]
```

---

## 12. Chunking Với Document Structure

### 12.1 Sectioned Document (PDF, Book)

```python
# Bước 1: Tách theo chapter trước
chapter_splitter = MarkdownHeaderTextSplitter([("#", "Chapter")])

# Bước 2: Trong mỗi chapter, chia nhỏ
content_splitter = RecursiveCharacterTextSplitter(chunk_size=500)

chapters = chapter_splitter.split_text(book_markdown)
final_chunks = []
for chapter in chapters:
    sub_chunks = content_splitter.split_text(chapter.page_content)
    for sc in sub_chunks:
        final_chunks.append(Document(
            page_content=sc,
            metadata={**chapter.metadata, "type": "section"}
        ))
```

### 12.2 Table-Aware Chunking

```python
# Cho PDF có nhiều table, dùng Unstructured để extract table riêng
from langchain_unstructured import UnstructuredLoader

loader = UnstructuredLoader(
    "report.pdf",
    strategy="hi_res",
    chunking_strategy="by_title",  # Smart chunking
    max_characters=2000,
)
docs = loader.load()
# Mỗi table sẽ là 1 Document riêng (không bị cắt giữa table)
```

---

## 13. Bài Tập

### Bài 1: Compare Splitters
Cho 1 PDF 50 trang, dùng 3 splitter khác nhau (Recursive, Token, Semantic). So sánh:
- Số chunk
- Trung bình tokens/chunk
- Distribution chunk size

### Bài 2: Markdown Documentation
Index documentation viết bằng markdown:
- Chia theo header (level 1, 2, 3)
- Giữ metadata header trong chunk
- Trong mỗi section, recursive chunk

### Bài 3: Code Repository
Index source code của 1 project:
- Python files → Python splitter
- Markdown files → Header splitter
- Test retrieval với query về function cụ thể

---

## 14. Checklist

- [ ] Hiểu tại sao cần chunking
- [ ] Dùng được CharacterTextSplitter và RecursiveCharacterTextSplitter
- [ ] Token-based splitting với tiktoken
- [ ] Language-specific splitting (Python, JS)
- [ ] Markdown/HTML header splitting
- [ ] Semantic chunking
- [ ] Parent-child chunking
- [ ] Choose chunk_size phù hợp use case

➡️ **Tiếp theo**: [Vector Stores](./04-Vector-Stores.md)
