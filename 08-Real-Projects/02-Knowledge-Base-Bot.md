# Project 2: Knowledge Base Bot (Confluence/Notion)

## 1. Mục Tiêu

Build bot trả lời câu hỏi từ knowledge base nội bộ (Confluence, Notion, hoặc folder docs):
- Tự động sync data
- Hybrid search (semantic + keyword)
- Slack integration
- Citation với links

---

## 2. Architecture

```
[Confluence/Notion] ─sync→ [Vector DB + Document Store]
                              ↓
[Slack Bot] ←─→ [RAG Pipeline] ←─→ [LLM]
                              ↓
                         [Citations + Links]
```

---

## 3. Tech Stack

- LangChain
- ChromaDB / Pinecone
- BM25 (hybrid)
- Cohere Rerank
- Slack SDK
- APScheduler (sync jobs)

---

## 4. Setup

```bash
pip install langchain langchain-openai langchain-chroma
pip install atlassian-python-api  # Confluence
pip install notion-client  # Notion
pip install slack-bolt slack-sdk
pip install rank-bm25 langchain-cohere
pip install apscheduler
```

---

## 5. Data Sync

### 5.1 Confluence

```python
from langchain_community.document_loaders import ConfluenceLoader

class ConfluenceSync:
    def __init__(self, url, username, api_key, space_key):
        self.loader = ConfluenceLoader(
            url=url,
            username=username,
            api_key=api_key,
            space_key=space_key,
        )
    
    def fetch_all(self):
        docs = self.loader.load(
            include_attachments=False,
            limit=500,
        )
        return docs
    
    def fetch_modified(self, since_date):
        # Use cql query
        cql = f'lastmodified > "{since_date}" AND space = "{self.space_key}"'
        docs = self.loader.load(cql=cql)
        return docs
```

### 5.2 Notion

```python
from notion_client import Client

class NotionSync:
    def __init__(self, token, database_id):
        self.client = Client(auth=token)
        self.database_id = database_id
    
    def fetch_all(self):
        results = []
        cursor = None
        while True:
            response = self.client.databases.query(
                database_id=self.database_id,
                start_cursor=cursor,
            )
            results.extend(response["results"])
            if not response.get("has_more"):
                break
            cursor = response.get("next_cursor")
        
        return [self._page_to_document(p) for p in results]
    
    def _page_to_document(self, page):
        from langchain_core.documents import Document
        
        # Extract content (blocks API)
        blocks = self.client.blocks.children.list(page["id"])["results"]
        content = "\n".join(self._block_to_text(b) for b in blocks)
        
        # Extract title
        title = ""
        for prop_name, prop in page["properties"].items():
            if prop["type"] == "title":
                title = "".join(t["plain_text"] for t in prop["title"])
                break
        
        return Document(
            page_content=f"# {title}\n\n{content}",
            metadata={
                "id": page["id"],
                "title": title,
                "url": page["url"],
                "last_edited": page["last_edited_time"],
            }
        )
    
    def _block_to_text(self, block):
        block_type = block["type"]
        if block_type == "paragraph":
            return "".join(t["plain_text"] for t in block["paragraph"]["rich_text"])
        elif block_type.startswith("heading"):
            return "## " + "".join(t["plain_text"] for t in block[block_type]["rich_text"])
        elif block_type == "bulleted_list_item":
            return "- " + "".join(t["plain_text"] for t in block["bulleted_list_item"]["rich_text"])
        return ""
```

### 5.3 File System

```python
from langchain_community.document_loaders import DirectoryLoader, TextLoader

class FileSync:
    def __init__(self, base_dir):
        self.base_dir = base_dir
    
    def fetch_all(self):
        loader = DirectoryLoader(
            self.base_dir,
            glob="**/*.md",
            loader_cls=TextLoader,
            show_progress=True,
        )
        return loader.load()
```

---

## 6. Indexing With Sync

```python
from langchain.indexes import SQLRecordManager, index
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

class KnowledgeBaseIndexer:
    def __init__(self, persist_dir="./kb_chroma"):
        self.embeddings = OpenAIEmbeddings()
        self.vectorstore = Chroma(
            embedding_function=self.embeddings,
            persist_directory=persist_dir,
            collection_name="kb",
        )
        self.splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200,
        )
        self.record_manager = SQLRecordManager(
            namespace="kb",
            db_url="sqlite:///kb_records.db",
        )
        self.record_manager.create_schema()
    
    def sync(self, source_docs):
        """Sync incrementally"""
        # Split
        chunks = self.splitter.split_documents(source_docs)
        
        # Index with cleanup
        result = index(
            chunks,
            self.record_manager,
            self.vectorstore,
            cleanup="incremental",
            source_id_key="id",  # Or "url", "source"
        )
        
        return result

# Daily cron job
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

def daily_sync():
    confluence = ConfluenceSync(...)
    notion = NotionSync(...)
    indexer = KnowledgeBaseIndexer()
    
    docs = confluence.fetch_all() + notion.fetch_all()
    result = indexer.sync(docs)
    print(f"Synced: {result}")

scheduler.add_job(daily_sync, "cron", hour=2)  # 2am daily
scheduler.start()
```

---

## 7. Hybrid Retriever + Rerank

```python
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_cohere import CohereRerank

class KBRetriever:
    def __init__(self, vectorstore, all_chunks):
        # Semantic
        self.semantic = vectorstore.as_retriever(search_kwargs={"k": 10})
        
        # BM25
        self.bm25 = BM25Retriever.from_documents(all_chunks)
        self.bm25.k = 10
        
        # Hybrid
        self.hybrid = EnsembleRetriever(
            retrievers=[self.semantic, self.bm25],
            weights=[0.5, 0.5],
        )
        
        # Rerank
        reranker = CohereRerank(model="rerank-multilingual-v3.0", top_n=5)
        self.retriever = ContextualCompressionRetriever(
            base_compressor=reranker,
            base_retriever=self.hybrid,
        )
    
    def invoke(self, query):
        return self.retriever.invoke(query)
```

---

## 8. RAG Chain

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là trợ lý nội bộ trả lời từ knowledge base.

QUY TẮC:
1. CHỈ dùng context bên dưới
2. Cite source link sau mỗi claim, format: [Title](URL)
3. Nếu không có info: "Tôi không tìm thấy trong knowledge base. Bạn có thể hỏi #help-desk."
4. Tiếng Việt, ngắn gọn

CONTEXT:
{context}"""),
    ("user", "{question}"),
])

def format_docs(docs):
    return "\n\n---\n\n".join(
        f"Title: {d.metadata.get('title')}\nURL: {d.metadata.get('url')}\n\n{d.page_content}"
        for d in docs
    )

def build_chain(retriever, llm):
    return (
        {
            "context": retriever | format_docs,
            "question": lambda x: x,
        }
        | prompt
        | llm
        | StrOutputParser()
    )
```

---

## 9. Slack Bot Integration

```python
# slack_bot.py
import os
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

app = App(token=os.environ["SLACK_BOT_TOKEN"])

# Initialize RAG
from kb_engine import build_chain, KBRetriever
retriever = KBRetriever(...)
chain = build_chain(retriever, llm)

@app.event("app_mention")
def handle_mention(event, say):
    text = event["text"]
    user = event["user"]
    
    # Remove bot mention
    question = text.split(">", 1)[1].strip() if ">" in text else text
    
    # Get answer
    say(f"<@{user}> Đang tìm thông tin...", thread_ts=event["ts"])
    
    try:
        answer = chain.invoke(question)
        say(answer, thread_ts=event["ts"])
    except Exception as e:
        say(f"Sorry, error: {e}", thread_ts=event["ts"])

@app.command("/kb-search")
def kb_search(ack, command, respond):
    ack()
    question = command["text"]
    
    answer = chain.invoke(question)
    respond({
        "response_type": "in_channel",
        "text": answer,
    })

if __name__ == "__main__":
    SocketModeHandler(app, os.environ["SLACK_APP_TOKEN"]).start()
```

### Setup Slack App

1. https://api.slack.com/apps → Create New App
2. **OAuth**: Bot Token Scopes
   - `app_mentions:read`
   - `chat:write`
   - `commands`
3. **Event Subscriptions**: subscribe to `app_mention`
4. **Slash Commands**: `/kb-search`
5. **Socket Mode**: enable (no public URL needed)
6. Install to workspace

---

## 10. Web UI Alternative

```python
# web_app.py
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

app = FastAPI()

class Query(BaseModel):
    question: str
    user_id: str

@app.post("/ask")
async def ask(query: Query):
    answer = chain.invoke(query.question)
    return {"answer": answer}

@app.post("/ask-stream")
async def ask_stream(query: Query):
    async def stream():
        async for chunk in chain.astream(query.question):
            yield chunk
    return StreamingResponse(stream(), media_type="text/plain")
```

---

## 11. Analytics & Feedback

```python
# Track queries
from sqlalchemy import create_engine
import sqlalchemy as sa

engine = create_engine("sqlite:///kb_analytics.db")

with engine.connect() as conn:
    conn.execute(sa.text("""
        CREATE TABLE IF NOT EXISTS queries (
            id INTEGER PRIMARY KEY,
            user_id TEXT,
            question TEXT,
            answer TEXT,
            sources TEXT,
            feedback INTEGER,  -- -1, 0, 1
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """))

def log_query(user_id, question, answer, sources):
    with engine.connect() as conn:
        conn.execute(sa.text("""
            INSERT INTO queries (user_id, question, answer, sources)
            VALUES (:uid, :q, :a, :s)
        """), {"uid": user_id, "q": question, "a": answer, "s": json.dumps(sources)})
        conn.commit()
        return conn.execute(sa.text("SELECT last_insert_rowid()")).scalar()

# Slack feedback buttons
@app.action("feedback_good")
def feedback_good(ack, body):
    ack()
    query_id = body["actions"][0]["value"]
    update_feedback(query_id, 1)

# Identify gaps in KB
def find_gaps():
    """Queries with negative feedback or "I don't know" responses"""
    with engine.connect() as conn:
        result = conn.execute(sa.text("""
            SELECT question, COUNT(*) as count
            FROM queries
            WHERE feedback = -1 OR answer LIKE '%không tìm thấy%'
            GROUP BY question
            ORDER BY count DESC
            LIMIT 20
        """))
        return result.fetchall()
```

---

## 12. Deployment

### docker-compose.yml

```yaml
services:
  kb-bot:
    build: .
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - SLACK_BOT_TOKEN=${SLACK_BOT_TOKEN}
      - SLACK_APP_TOKEN=${SLACK_APP_TOKEN}
      - CONFLUENCE_TOKEN=${CONFLUENCE_TOKEN}
    volumes:
      - ./kb_chroma:/app/kb_chroma
      - ./kb_records.db:/app/kb_records.db
    restart: unless-stopped
  
  sync-cron:
    build: .
    command: python sync_job.py
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - CONFLUENCE_TOKEN=${CONFLUENCE_TOKEN}
    volumes:
      - ./kb_chroma:/app/kb_chroma
    restart: unless-stopped
```

---

## 13. Bài Tập Mở Rộng

### Bài 1: Multi-Source
Combine Confluence + Notion + GitHub Wiki + Google Drive.

### Bài 2: Permission-Aware
Mỗi user chỉ thấy docs họ có quyền access.

### Bài 3: Smart Sync
Webhook real-time thay vì cron (instant update).

### Bài 4: Analytics Dashboard
Top queries, gaps, user engagement.

---

## 14. Checklist

- [ ] Sync từ Confluence/Notion
- [ ] Incremental indexing
- [ ] Hybrid retrieval + rerank
- [ ] Slack bot integration
- [ ] Citations với links
- [ ] Analytics & feedback
- [ ] Daily sync cron job

➡️ **Tiếp theo**: [AI Agent Gọi API](./03-AI-Agent-API.md)
