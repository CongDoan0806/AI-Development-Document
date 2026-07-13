# Project 1: Chatbot Hỏi Đáp PDF

## 1. Mục Tiêu

Build chatbot có thể:
- Upload PDF (1 hoặc nhiều files)
- Index nội dung
- Trả lời câu hỏi về PDF
- Nhớ history conversation
- Stream response
- Cite sources (trang nào, file nào)

**Tech stack**:
- LangChain (RAG)
- ChromaDB (vector store)
- OpenAI (LLM + embedding)
- Streamlit (UI)

---

## 2. Architecture

```
User uploads PDF
       ↓
[Load + Split chunks]
       ↓
[Embed → ChromaDB]
       ↓
User asks question
       ↓
[Rewrite question with history]
       ↓
[Retrieve relevant chunks]
       ↓
[Generate answer with citations]
       ↓
Stream to UI
```

---

## 3. Setup

```bash
pip install streamlit langchain langchain-openai langchain-chroma pypdf
```

```env
# .env
OPENAI_API_KEY=sk-...
```

---

## 4. Code Đầy Đủ

### 4.1 Backend Logic

```python
# rag_engine.py
import os
import tempfile
from pathlib import Path
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_core.documents import Document
from langchain.chains import create_history_aware_retriever
from langchain_core.messages import HumanMessage, AIMessage


class PDFChatbot:
    def __init__(self, persist_dir: str = "./pdf_chroma"):
        self.persist_dir = persist_dir
        self.embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
        self.llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
        self.vectorstore = None
        self.retriever = None
        self.history = []
    
    def index_pdf(self, pdf_file_path: str, source_name: str = None):
        """Index 1 PDF file"""
        source_name = source_name or Path(pdf_file_path).name
        
        # Load
        loader = PyPDFLoader(pdf_file_path)
        docs = loader.load()
        
        # Add metadata
        for doc in docs:
            doc.metadata["source"] = source_name
        
        # Split
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200,
            separators=["\n\n", "\n", ". ", " ", ""],
        )
        chunks = splitter.split_documents(docs)
        
        # Add chunk index
        for i, chunk in enumerate(chunks):
            chunk.metadata["chunk_id"] = i
        
        # Vectorstore
        if self.vectorstore is None:
            self.vectorstore = Chroma(
                embedding_function=self.embeddings,
                persist_directory=self.persist_dir,
                collection_name="pdfs",
            )
        
        self.vectorstore.add_documents(chunks)
        self.retriever = self.vectorstore.as_retriever(search_kwargs={"k": 5})
        
        return len(chunks)
    
    def _build_chain(self):
        # History-aware rewrite
        rewrite_prompt = ChatPromptTemplate.from_messages([
            ("system", """Cho lịch sử chat và câu hỏi mới, viết lại thành câu hỏi standalone (không phụ thuộc context).
Nếu câu hỏi đã standalone, giữ nguyên."""),
            MessagesPlaceholder("chat_history"),
            ("user", "{input}"),
        ])
        
        history_aware = create_history_aware_retriever(
            self.llm, self.retriever, rewrite_prompt
        )
        
        # Answer prompt
        answer_prompt = ChatPromptTemplate.from_messages([
            ("system", """Bạn là trợ lý AI trả lời câu hỏi về PDF documents.

QUY TẮC:
1. CHỈ trả lời dựa trên context bên dưới
2. Cite source dạng [filename, p.X] sau mỗi claim
3. Nếu không có info: "Tôi không tìm thấy thông tin này trong tài liệu"
4. Trả lời bằng tiếng Việt
5. Format có cấu trúc (bullet, paragraph) nếu phù hợp

CONTEXT:
{context}"""),
            MessagesPlaceholder("chat_history"),
            ("user", "{input}"),
        ])
        
        def format_docs(docs):
            return "\n\n".join(
                f"[{d.metadata.get('source')}, p.{d.metadata.get('page', '?')+1}]\n{d.page_content}"
                for d in docs
            )
        
        # Full chain
        chain = (
            RunnablePassthrough.assign(
                context=lambda x: format_docs(history_aware.invoke({
                    "input": x["input"],
                    "chat_history": x["chat_history"],
                }))
            )
            | answer_prompt
            | self.llm
            | StrOutputParser()
        )
        
        return chain
    
    def chat(self, question: str) -> str:
        if not self.retriever:
            return "Chưa upload PDF nào."
        
        chain = self._build_chain()
        response = chain.invoke({
            "input": question,
            "chat_history": self.history,
        })
        
        # Update history
        self.history.append(HumanMessage(content=question))
        self.history.append(AIMessage(content=response))
        
        return response
    
    async def stream_chat(self, question: str):
        if not self.retriever:
            yield "Chưa upload PDF nào."
            return
        
        chain = self._build_chain()
        full_response = ""
        async for chunk in chain.astream({
            "input": question,
            "chat_history": self.history,
        }):
            full_response += chunk
            yield chunk
        
        self.history.append(HumanMessage(content=question))
        self.history.append(AIMessage(content=full_response))
    
    def reset_history(self):
        self.history = []
    
    def get_sources_for_query(self, question: str):
        """Get retrieved sources for transparency"""
        if not self.retriever:
            return []
        docs = self.retriever.invoke(question)
        return [
            {
                "source": d.metadata.get("source"),
                "page": d.metadata.get("page", 0) + 1,
                "snippet": d.page_content[:200],
            }
            for d in docs
        ]
```

### 4.2 Streamlit UI

```python
# app.py
import streamlit as st
import tempfile
import os
from rag_engine import PDFChatbot

st.set_page_config(page_title="PDF Chatbot", page_icon="📄", layout="wide")

# Initialize
if "chatbot" not in st.session_state:
    st.session_state.chatbot = PDFChatbot()
    st.session_state.messages = []
    st.session_state.indexed_files = []

# Sidebar - Upload
with st.sidebar:
    st.title("📄 Upload PDFs")
    
    uploaded_files = st.file_uploader(
        "Choose PDF files",
        type=["pdf"],
        accept_multiple_files=True,
    )
    
    if st.button("Index", disabled=not uploaded_files):
        with st.spinner("Indexing..."):
            for uploaded_file in uploaded_files:
                if uploaded_file.name in st.session_state.indexed_files:
                    continue
                
                # Save temp
                with tempfile.NamedTemporaryFile(delete=False, suffix=".pdf") as tmp:
                    tmp.write(uploaded_file.read())
                    tmp_path = tmp.name
                
                # Index
                chunks = st.session_state.chatbot.index_pdf(tmp_path, uploaded_file.name)
                st.session_state.indexed_files.append(uploaded_file.name)
                st.success(f"Indexed {uploaded_file.name} ({chunks} chunks)")
                
                # Cleanup
                os.unlink(tmp_path)
    
    if st.session_state.indexed_files:
        st.divider()
        st.subheader("📚 Indexed files")
        for f in st.session_state.indexed_files:
            st.text(f"• {f}")
    
    if st.button("🗑️ Clear chat"):
        st.session_state.messages = []
        st.session_state.chatbot.reset_history()
        st.rerun()

# Main chat
st.title("Chat with your PDFs")

# Show history
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])
        if "sources" in msg:
            with st.expander("📎 Sources"):
                for s in msg["sources"]:
                    st.text(f"{s['source']} - p.{s['page']}")
                    st.caption(s["snippet"][:200])

# Input
if prompt := st.chat_input("Ask about your PDFs..."):
    if not st.session_state.indexed_files:
        st.error("Please upload PDFs first")
    else:
        # User message
        st.session_state.messages.append({"role": "user", "content": prompt})
        with st.chat_message("user"):
            st.markdown(prompt)
        
        # Assistant response
        with st.chat_message("assistant"):
            placeholder = st.empty()
            full_response = ""
            
            # Stream
            import asyncio
            
            async def get_response():
                response = ""
                async for chunk in st.session_state.chatbot.stream_chat(prompt):
                    response += chunk
                    placeholder.markdown(response + "▌")
                return response
            
            full_response = asyncio.run(get_response())
            placeholder.markdown(full_response)
            
            # Sources
            sources = st.session_state.chatbot.get_sources_for_query(prompt)
            with st.expander("📎 Sources"):
                for s in sources:
                    st.text(f"{s['source']} - p.{s['page']}")
                    st.caption(s["snippet"][:200])
        
        # Save
        st.session_state.messages.append({
            "role": "assistant",
            "content": full_response,
            "sources": sources,
        })
```

---

## 5. Run

```bash
streamlit run app.py
```

Open browser → upload PDF → ask questions.

---

## 6. Improvements (Roadmap)

### Phase 1 (MVP - đã có)
- ✅ Upload + index PDFs
- ✅ Q&A với citations
- ✅ Streaming

### Phase 2 (Enhanced)
- 🔲 Hybrid search (BM25 + semantic)
- 🔲 Reranking với Cohere
- 🔲 Multi-query expansion
- 🔲 Better chunking (parent-child)
- 🔲 Persistent storage (across sessions)

### Phase 3 (Production)
- 🔲 User authentication
- 🔲 Per-user vector store
- 🔲 Rate limiting
- 🔲 Cost tracking
- 🔲 Deploy to cloud

### Phase 4 (Advanced)
- 🔲 Multi-modal (extract tables, images)
- 🔲 Document comparison
- 🔲 Summarization
- 🔲 Export Q&A history

---

## 7. Code Cải Tiến: Production Version

```python
# Production-ready với hybrid search + rerank
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_cohere import CohereRerank

class ProductionPDFChatbot(PDFChatbot):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.bm25_retriever = None
        self.all_chunks = []
    
    def index_pdf(self, pdf_path, source_name=None):
        n = super().index_pdf(pdf_path, source_name)
        
        # Cũng index cho BM25
        # (Cần re-fetch tất cả chunks vì BM25 cần list)
        all_docs = self.vectorstore.get()
        self.all_chunks = [
            Document(page_content=d, metadata=m)
            for d, m in zip(all_docs["documents"], all_docs["metadatas"])
        ]
        self.bm25_retriever = BM25Retriever.from_documents(self.all_chunks)
        self.bm25_retriever.k = 5
        
        # Hybrid retriever
        semantic = self.vectorstore.as_retriever(search_kwargs={"k": 10})
        hybrid = EnsembleRetriever(
            retrievers=[semantic, self.bm25_retriever],
            weights=[0.6, 0.4],
        )
        
        # Rerank
        reranker = CohereRerank(model="rerank-multilingual-v3.0", top_n=5)
        self.retriever = ContextualCompressionRetriever(
            base_compressor=reranker,
            base_retriever=hybrid,
        )
        
        return n
```

---

## 8. Deployment

### 8.1 Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.address=0.0.0.0"]
```

### 8.2 Deploy Options

- **Streamlit Cloud**: free, push to GitHub
- **HuggingFace Spaces**: free, support Streamlit/Gradio
- **Railway/Render**: $5-20/mo
- **AWS ECS**: production scale

---

## 9. Bài Tập Mở Rộng

### Bài 1: Multi-Format Support
Thêm support: docx, txt, pptx, markdown.

### Bài 2: Voice Input/Output
Thêm Whisper (speech-to-text) và TTS.

### Bài 3: Summarization
Tab "Summarize" tự động tóm tắt PDF.

### Bài 4: PDF Annotations
Highlight phần được cite trong viewer PDF.

---

## 10. Checklist

- [ ] Upload và index PDFs
- [ ] Q&A với citations
- [ ] Streaming response
- [ ] Conversation history
- [ ] Multi-PDF support
- [ ] Production version với hybrid + rerank
- [ ] Deploy demo

➡️ **Tiếp theo**: [Knowledge Base Bot](./02-Knowledge-Base-Bot.md)
