# Document Loaders - Load Tài Liệu Vào RAG

## 1. Document Là Gì?

```python
from langchain_core.documents import Document

doc = Document(
    page_content="Đây là nội dung text...",
    metadata={
        "source": "file.pdf",
        "page": 1,
        "author": "Nguyen Van A",
        "date": "2024-01-15",
    }
)
```

**Document Loader** = class chuyển nguồn dữ liệu (PDF, web, CSV, ...) thành `List[Document]`.

---

## 2. PDF Loader

### 2.1 PyPDFLoader - Đơn Giản

```python
# pip install pypdf
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("path/to/file.pdf")
documents = loader.load()  # List[Document]

# Mỗi page là 1 Document
for doc in documents:
    print(f"Page {doc.metadata['page']}: {doc.page_content[:100]}...")
```

### 2.2 PyMuPDFLoader - Nhanh & Chính Xác

```python
# pip install pymupdf
from langchain_community.document_loaders import PyMuPDFLoader

loader = PyMuPDFLoader("file.pdf")
docs = loader.load()
```

### 2.3 PDFPlumberLoader - Tốt Cho Table

```python
# pip install pdfplumber
from langchain_community.document_loaders import PDFPlumberLoader

loader = PDFPlumberLoader("file.pdf")
docs = loader.load()
# PDFPlumber extract table tốt hơn
```

### 2.4 UnstructuredPDFLoader - Đa Dạng

```python
# pip install unstructured[pdf]
from langchain_community.document_loaders import UnstructuredPDFLoader

# Mode "single": 1 document tổng
loader = UnstructuredPDFLoader("file.pdf", mode="single")

# Mode "elements": chia theo element (paragraph, table, image)
loader = UnstructuredPDFLoader("file.pdf", mode="elements")
docs = loader.load()
```

### 2.5 PDF Có Hình - PyPDFLoader + Vision

```python
# Cho PDF có hình ảnh, dùng vision model
from langchain_unstructured import UnstructuredLoader

loader = UnstructuredLoader(
    "file.pdf",
    strategy="hi_res",  # High resolution OCR
    extract_image_block_types=["Image", "Table"],
)
docs = loader.load()
```

### 2.6 So Sánh PDF Loaders

| Loader | Pros | Cons |
|--------|------|------|
| `PyPDFLoader` | Đơn giản, fast | Không extract table tốt |
| `PyMuPDFLoader` | Nhanh, chất lượng | Cần install pymupdf |
| `PDFPlumberLoader` | Table tốt | Chậm hơn |
| `UnstructuredPDFLoader` | Versatile | Nặng, slow |

---

## 3. Word/PowerPoint Loader

```python
# pip install python-docx
from langchain_community.document_loaders import Docx2txtLoader

loader = Docx2txtLoader("doc.docx")
docs = loader.load()

# PowerPoint
# pip install python-pptx
from langchain_community.document_loaders import UnstructuredPowerPointLoader
loader = UnstructuredPowerPointLoader("file.pptx")
docs = loader.load()
```

---

## 4. Web/HTML Loader

### 4.1 WebBaseLoader - Cơ Bản

```python
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://example.com/article")
docs = loader.load()
```

### 4.2 Nhiều URLs

```python
loader = WebBaseLoader([
    "https://example.com/1",
    "https://example.com/2",
])
docs = loader.load()
```

### 4.3 BeautifulSoup Selector

```python
import bs4

loader = WebBaseLoader(
    web_paths=["https://example.com"],
    bs_kwargs={
        "parse_only": bs4.SoupStrainer(
            class_=("post-title", "post-content")
        )
    },
)
docs = loader.load()
```

### 4.4 Sitemap Loader

```python
# pip install lxml
from langchain_community.document_loaders import SitemapLoader

loader = SitemapLoader("https://example.com/sitemap.xml")
docs = loader.load()
```

### 4.5 Async Web Loading

```python
from langchain_community.document_loaders import AsyncHtmlLoader

loader = AsyncHtmlLoader(urls=["url1", "url2", "url3"])
docs = loader.load()
```

### 4.6 Playwright (Cho SPA)

```python
# pip install playwright
# playwright install
from langchain_community.document_loaders import PlaywrightURLLoader

urls = ["https://spa-website.com"]
loader = PlaywrightURLLoader(
    urls=urls,
    remove_selectors=["header", "footer"]
)
docs = loader.load()
```

---

## 5. CSV Loader

```python
from langchain_community.document_loaders.csv_loader import CSVLoader

loader = CSVLoader(
    file_path="data.csv",
    csv_args={
        "delimiter": ",",
        "fieldnames": ["name", "age", "email"]
    },
)
docs = loader.load()
# Mỗi row là 1 Document
```

**Custom source column**:
```python
loader = CSVLoader(
    file_path="data.csv",
    source_column="url",  # Cột "url" được dùng làm source
)
```

---

## 6. JSON Loader

```python
# pip install jq
from langchain_community.document_loaders import JSONLoader

# data.json: [{"title": "...", "content": "..."}]
loader = JSONLoader(
    file_path="data.json",
    jq_schema=".[].content",  # Lấy field "content" của mỗi item
    text_content=False,
)

# Với metadata
def metadata_func(record: dict, metadata: dict) -> dict:
    metadata["title"] = record.get("title")
    metadata["author"] = record.get("author")
    return metadata

loader = JSONLoader(
    file_path="data.json",
    jq_schema=".[]",
    content_key="content",
    metadata_func=metadata_func,
)
```

---

## 7. Markdown Loader

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

loader = UnstructuredMarkdownLoader("README.md")
docs = loader.load()
```

**MarkdownHeaderTextSplitter** - chia theo header:
```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "Header 1"),
        ("##", "Header 2"),
        ("###", "Header 3"),
    ]
)

with open("README.md") as f:
    text = f.read()

chunks = splitter.split_text(text)
for chunk in chunks:
    print(chunk.metadata)  # {"Header 1": "...", "Header 2": "..."}
    print(chunk.page_content)
```

---

## 8. Notion / Confluence

### Notion
```python
# pip install langchain-notion
from langchain_community.document_loaders import NotionDirectoryLoader

loader = NotionDirectoryLoader("path/to/notion_export/")
docs = loader.load()
```

### Confluence
```python
from langchain_community.document_loaders import ConfluenceLoader

loader = ConfluenceLoader(
    url="https://yoursite.atlassian.net/wiki",
    username="...",
    api_key="...",
    space_key="SPACE",
)
docs = loader.load(limit=50)
```

---

## 9. Code Loader

```python
# Cho repo source code
from langchain_community.document_loaders import GitLoader

loader = GitLoader(
    repo_path="./my_repo",
    branch="main",
    file_filter=lambda x: x.endswith(".py"),
)
docs = loader.load()
```

**Language-specific splitter**:
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=500,
    chunk_overlap=50,
)
```

---

## 10. Database Loader

### SQL
```python
from langchain_community.document_loaders import SQLDatabaseLoader
from sqlalchemy import create_engine

engine = create_engine("postgresql://...")
loader = SQLDatabaseLoader(
    query="SELECT id, title, content FROM articles",
    db=engine,
)
docs = loader.load()
```

### MongoDB
```python
from langchain_community.document_loaders import MongodbLoader

loader = MongodbLoader(
    connection_string="mongodb://localhost:27017",
    db_name="mydb",
    collection_name="articles",
    field_names=["title", "content"]
)
docs = loader.load()
```

---

## 11. YouTube Transcript

```python
# pip install youtube-transcript-api
from langchain_community.document_loaders import YoutubeLoader

loader = YoutubeLoader.from_youtube_url(
    "https://www.youtube.com/watch?v=...",
    add_video_info=True,
    language=["vi", "en"],  # Ưu tiên tiếng Việt
)
docs = loader.load()
```

---

## 12. Cloud Storage

### AWS S3
```python
# pip install boto3
from langchain_community.document_loaders import S3DirectoryLoader

loader = S3DirectoryLoader("my-bucket", prefix="docs/")
docs = loader.load()
```

### Google Drive
```python
from langchain_community.document_loaders import GoogleDriveLoader

loader = GoogleDriveLoader(
    folder_id="...",
    credentials_path="credentials.json",
)
docs = loader.load()
```

### Azure Blob
```python
from langchain_community.document_loaders import AzureBlobStorageContainerLoader

loader = AzureBlobStorageContainerLoader(
    conn_str="...",
    container="my-container"
)
docs = loader.load()
```

---

## 13. Directory Loader - Load Cả Folder

```python
from langchain_community.document_loaders import DirectoryLoader, PyPDFLoader

loader = DirectoryLoader(
    "./documents/",
    glob="**/*.pdf",       # Pattern
    loader_cls=PyPDFLoader,
    show_progress=True,
    use_multithreading=True,
    max_concurrency=4,
)
docs = loader.load()
print(f"Loaded {len(docs)} documents")
```

**Load nhiều loại file**:
```python
loaders = [
    DirectoryLoader("./docs/", glob="**/*.pdf", loader_cls=PyPDFLoader),
    DirectoryLoader("./docs/", glob="**/*.docx", loader_cls=Docx2txtLoader),
    DirectoryLoader("./docs/", glob="**/*.md", loader_cls=UnstructuredMarkdownLoader),
]

all_docs = []
for loader in loaders:
    all_docs.extend(loader.load())
```

---

## 14. Lazy Loading - Tiết Kiệm Memory

```python
loader = PyPDFLoader("large_file.pdf")

# Load tất cả vào memory
docs = loader.load()

# Lazy load - yield từng doc
for doc in loader.lazy_load():
    process(doc)  # Xử lý từng doc, không tải hết
```

---

## 15. Async Loading

```python
loader = WebBaseLoader([url1, url2, url3])

# Async load
docs = await loader.aload()
```

---

## 16. Custom Loader

```python
from langchain_core.document_loaders import BaseLoader
from langchain_core.documents import Document
from typing import Iterator

class MyAPILoader(BaseLoader):
    def __init__(self, api_url: str, api_key: str):
        self.api_url = api_url
        self.api_key = api_key
    
    def lazy_load(self) -> Iterator[Document]:
        import requests
        response = requests.get(
            self.api_url,
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        for item in response.json():
            yield Document(
                page_content=item["content"],
                metadata={
                    "id": item["id"],
                    "title": item["title"],
                    "source": self.api_url,
                }
            )

loader = MyAPILoader("https://api.example.com/articles", "key")
docs = loader.load()
```

---

## 17. Best Practices

### 17.1 Thêm Metadata Đầy Đủ
```python
for doc in docs:
    doc.metadata.update({
        "loaded_at": datetime.now().isoformat(),
        "document_type": "pdf",
        "language": "vi",
        "category": "tech",
    })
```

### 17.2 Cleaning Text
```python
import re

def clean_text(text):
    # Remove extra whitespace
    text = re.sub(r'\s+', ' ', text)
    # Remove special chars
    text = re.sub(r'[^\w\s.,!?-]', '', text)
    return text.strip()

for doc in docs:
    doc.page_content = clean_text(doc.page_content)
```

### 17.3 Filter Empty Documents
```python
docs = [d for d in docs if len(d.page_content) > 50]
```

### 17.4 Validate Encoding
```python
# Đôi khi PDF/HTML có encoding lạ
for doc in docs:
    try:
        doc.page_content.encode('utf-8')
    except UnicodeEncodeError:
        # Fix hoặc skip
        doc.page_content = doc.page_content.encode('utf-8', 'ignore').decode('utf-8')
```

---

## 18. Demo: Multi-Source RAG Indexing

```python
from langchain_community.document_loaders import (
    PyPDFLoader,
    WebBaseLoader,
    Docx2txtLoader,
)

all_docs = []

# 1. Load PDFs
import os
pdf_folder = "./pdfs"
for filename in os.listdir(pdf_folder):
    if filename.endswith(".pdf"):
        loader = PyPDFLoader(os.path.join(pdf_folder, filename))
        docs = loader.load()
        for d in docs:
            d.metadata["source_type"] = "pdf"
        all_docs.extend(docs)

# 2. Load web pages
web_urls = [
    "https://example.com/article1",
    "https://example.com/article2",
]
web_loader = WebBaseLoader(web_urls)
web_docs = web_loader.load()
for d in web_docs:
    d.metadata["source_type"] = "web"
all_docs.extend(web_docs)

# 3. Load Word docs
docx_loader = Docx2txtLoader("./contract.docx")
docx_docs = docx_loader.load()
for d in docx_docs:
    d.metadata["source_type"] = "docx"
all_docs.extend(docx_docs)

print(f"Total documents: {len(all_docs)}")
print(f"By type: {set(d.metadata['source_type'] for d in all_docs)}")
```

---

## 19. Bài Tập

### Bài 1: Multi-Format Loader
Viết function `load_all(folder_path)` tự động load:
- *.pdf → PyPDFLoader
- *.docx → Docx2txtLoader
- *.txt → TextLoader
- *.md → UnstructuredMarkdownLoader
- *.csv → CSVLoader

### Bài 2: Custom Wikipedia Loader
Tạo custom loader load Wikipedia article theo title, output Document với metadata: title, url, last_modified, summary.

### Bài 3: Website Scraper
Crawl 1 blog (10 articles) → cleaned Documents với metadata: url, title, date, author.

---

## 20. Checklist

- [ ] Load được PDF với nhiều loader khác nhau
- [ ] Load Word, PowerPoint, Markdown
- [ ] Web scraping với WebBaseLoader, Playwright
- [ ] Load CSV, JSON với schema
- [ ] DirectoryLoader cho cả folder
- [ ] Lazy loading cho file lớn
- [ ] Cleaning + metadata enrichment
- [ ] Tạo custom loader

➡️ **Tiếp theo**: [Chunking Strategy](./03-Chunking-Strategy.md)
