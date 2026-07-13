# RAG Chain End-to-End - Pipeline Hoàn Chỉnh

## 1. Tổng Hợp Lại Mọi Thứ

Pipeline RAG production-grade gồm:

```
USER QUERY
    ↓
[1. Query Preprocessing]
    ├── Spell correction
    ├── History-aware rewrite (chatbot)
    └── Intent classification
    ↓
[2. Query Enhancement]
    ├── Multi-query expansion
    ├── HyDE
    └── Step-back prompting
    ↓
[3. Retrieval]
    ├── Vector search (dense)
    ├── BM25 (sparse)
    └── Hybrid ensemble
    ↓
[4. Reranking]
    └── Cohere/BGE/LLM rerank
    ↓
[5. Compression]
    ├── Relevance filter
    └── Long context reorder
    ↓
[6. Generation]
    ├── Format docs with citations
    ├── Build prompt with context
    └── LLM generate
    ↓
[7. Post-processing]
    ├── Validate answer
    ├── Add citations
    └── Stream output
    ↓
ANSWER
```

---

## 2. Build Từng Bước

### 2.1 Setup

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableLambda

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    embedding_function=embeddings,
    persist_directory="./db",
)
```

### 2.2 Step 1: History-Aware Rewrite

```python
rewrite_prompt = ChatPromptTemplate.from_template("""
Cho lịch sử hội thoại và câu hỏi mới, viết lại câu hỏi thành standalone (không phụ thuộc context).

Lịch sử:
{chat_history}

Câu hỏi mới: {question}

Chỉ trả về câu standalone, không giải thích:
""")

rewrite_chain = rewrite_prompt | llm | StrOutputParser()
```

### 2.3 Step 2: Multi-Query Expansion

```python
expand_prompt = ChatPromptTemplate.from_template("""
Sinh 3 biến thể khác nhau của câu hỏi để search đa chiều.

Query: {query}

3 biến thể (mỗi dòng 1 query, không thêm số thứ tự):
""")

class LineParser(StrOutputParser):
    def parse(self, text):
        return [l.strip() for l in text.strip().split("\n") if l.strip()]

expand_chain = expand_prompt | llm | LineParser()
```

### 2.4 Step 3: Hybrid Retrieval

```python
from langchain_community.retrievers import BM25Retriever
from langchain.retrievers import EnsembleRetriever

# Load BM25 từ docs đã indexed (giả sử có sẵn)
all_docs_for_bm25 = ...  # Load từ DB hoặc file

semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})
bm25_retriever = BM25Retriever.from_documents(all_docs_for_bm25)
bm25_retriever.k = 10

hybrid_retriever = EnsembleRetriever(
    retrievers=[semantic_retriever, bm25_retriever],
    weights=[0.6, 0.4],
)
```

### 2.5 Step 4: Reranking

```python
from langchain_cohere import CohereRerank
from langchain.retrievers import ContextualCompressionRetriever

reranker = CohereRerank(
    model="rerank-multilingual-v3.0",
    top_n=5,
)

final_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=hybrid_retriever,
)
```

### 2.6 Step 5: Long Context Reorder

```python
from langchain_community.document_transformers import LongContextReorder

reorderer = LongContextReorder()

def reorder(docs):
    return reorderer.transform_documents(docs)
```

### 2.7 Step 6: Format Docs With Citations

```python
def format_docs_with_citations(docs):
    formatted = []
    for i, doc in enumerate(docs, 1):
        source = doc.metadata.get("source", "unknown")
        page = doc.metadata.get("page", "")
        formatted.append(
            f"[Source {i}: {source}{f' p.{page}' if page else ''}]\n"
            f"{doc.page_content}\n"
        )
    return "\n---\n".join(formatted)
```

### 2.8 Step 7: Answer Generation

```python
answer_prompt = ChatPromptTemplate.from_messages([
    ("system", """Bạn là trợ lý AI chuyên môn. Quy tắc:
1. CHỈ trả lời dựa trên context. Nếu không có thông tin, nói "Tôi không có thông tin về việc này".
2. Cite source [Source X] sau mỗi claim.
3. Trả lời ngắn gọn, rõ ràng, có cấu trúc.
4. Tiếng Việt."""),
    ("user", """Context:
{context}

Câu hỏi: {question}

Câu trả lời:""")
])

answer_chain = answer_prompt | llm | StrOutputParser()
```

### 2.9 Step 8: Combine Multi-Query Results

```python
def dedupe_docs(docs_list):
    """Combine nhiều list docs, loại trùng lặp"""
    seen = set()
    unique = []
    for docs in docs_list:
        for doc in docs:
            key = doc.page_content[:100]  # Dùng 100 chars đầu làm key
            if key not in seen:
                seen.add(key)
                unique.append(doc)
    return unique
```

### 2.10 Full Pipeline

```python
def full_rag_pipeline(question: str, chat_history: list = None) -> dict:
    # 1. Rewrite if has history
    if chat_history:
        history_str = "\n".join([f"{m['role']}: {m['content']}" for m in chat_history])
        question = rewrite_chain.invoke({
            "chat_history": history_str,
            "question": question
        })
    
    # 2. Expand to multiple queries
    queries = expand_chain.invoke({"query": question})
    queries.append(question)  # Include original
    
    # 3. Hybrid retrieve + rerank cho mỗi query
    all_docs = []
    for q in queries:
        docs = final_retriever.invoke(q)
        all_docs.append(docs)
    
    # 4. Dedupe
    unique_docs = dedupe_docs(all_docs)[:10]  # Top 10
    
    # 5. Reorder
    reordered = reorder(unique_docs)
    
    # 6. Format with citations
    context = format_docs_with_citations(reordered)
    
    # 7. Generate
    answer = answer_chain.invoke({
        "context": context,
        "question": question,
    })
    
    return {
        "question": question,
        "answer": answer,
        "sources": [d.metadata for d in reordered],
        "num_docs_used": len(reordered),
    }

# Test
result = full_rag_pipeline("LangChain là gì?")
print(result["answer"])
print(f"\nSources: {result['sources']}")
```

---

## 3. LCEL Version (Composable)

```python
from langchain_core.runnables import RunnableParallel

# Subchain: rewrite (only if history exists)
def maybe_rewrite(input):
    if input.get("chat_history"):
        return rewrite_chain.invoke(input)
    return input["question"]

rewrite_step = RunnableLambda(maybe_rewrite)

# Subchain: retrieve + rerank
def retrieve_step(question):
    return final_retriever.invoke(question)

# Subchain: format
format_step = RunnableLambda(format_docs_with_citations)

# Full chain
rag_chain = (
    {
        "context": rewrite_step | retrieve_step | format_step,
        "question": rewrite_step,
    }
    | answer_prompt
    | llm
    | StrOutputParser()
)

# Use
result = rag_chain.invoke({
    "question": "AI là gì?",
    "chat_history": [],
})
```

---

## 4. Streaming Version

```python
async def stream_rag(question, history=None):
    if history:
        question = rewrite_chain.invoke({...})
    
    docs = final_retriever.invoke(question)
    context = format_docs_with_citations(docs)
    
    # Stream answer
    async for chunk in answer_chain.astream({
        "context": context,
        "question": question,
    }):
        yield chunk

# Use in FastAPI
async def chat_endpoint(question: str):
    async for chunk in stream_rag(question):
        yield f"data: {chunk}\n\n"
```

---

## 5. Structured Output - Answer + Citations

```python
from pydantic import BaseModel, Field
from typing import List

class Citation(BaseModel):
    source: str
    quote: str
    relevance: float = Field(description="0-1 score")

class RAGAnswer(BaseModel):
    answer: str
    citations: List[Citation]
    confidence: float = Field(description="0-1: confidence in answer")
    needs_clarification: bool

structured_llm = llm.with_structured_output(RAGAnswer)

structured_chain = answer_prompt | structured_llm

result = structured_chain.invoke({
    "context": context,
    "question": question,
})

print(result.answer)
print(f"Confidence: {result.confidence}")
for c in result.citations:
    print(f"- [{c.source}] {c.quote}")
```

---

## 6. Conversational RAG Với Memory

```python
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.chat_history import InMemoryChatMessageHistory

store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

# Wrap chain với history
conversational_rag = RunnableWithMessageHistory(
    rag_chain,
    get_session_history,
    input_messages_key="question",
    history_messages_key="chat_history",
)

# Use
result1 = conversational_rag.invoke(
    {"question": "Python là gì?"},
    config={"configurable": {"session_id": "user_1"}}
)

result2 = conversational_rag.invoke(
    {"question": "Ai tạo ra nó?"},  # "nó" = Python (từ history)
    config={"configurable": {"session_id": "user_1"}}
)
```

---

## 7. Validation & Guardrails

```python
def validate_answer(question, context, answer):
    """Kiểm tra answer có hallucinate không"""
    
    validate_prompt = ChatPromptTemplate.from_template("""
    Đánh giá answer dưới đây có dựa hoàn toàn vào context không.

    Context: {context}
    Question: {question}
    Answer: {answer}

    Trả về JSON:
    {{
        "is_grounded": true/false,
        "issues": ["list of issues if any"],
        "missing_info": ["info missing from context"]
    }}
    """)
    
    validator = validate_prompt | llm | StrOutputParser()
    return validator.invoke({
        "context": context,
        "question": question,
        "answer": answer
    })
```

### Self-Correction Loop

```python
def rag_with_correction(question, max_retries=2):
    for attempt in range(max_retries + 1):
        docs = final_retriever.invoke(question)
        context = format_docs_with_citations(docs)
        answer = answer_chain.invoke({"context": context, "question": question})
        
        # Validate
        validation = validate_answer(question, context, answer)
        if "is_grounded: true" in validation:
            return answer
        
        # Retry với augmented context
        question = f"{question} (Cần thêm context cụ thể về: {validation})"
    
    return f"Không thể trả lời chính xác sau {max_retries} lần thử."
```

---

## 8. Production-Ready Class

```python
class ProductionRAG:
    def __init__(
        self,
        vectorstore,
        llm,
        embeddings,
        bm25_retriever=None,
        rerank_model="rerank-multilingual-v3.0",
        top_k_retrieve=20,
        top_n_rerank=5,
    ):
        self.vectorstore = vectorstore
        self.llm = llm
        self.embeddings = embeddings
        
        # Build retrievers
        semantic = vectorstore.as_retriever(search_kwargs={"k": top_k_retrieve})
        
        if bm25_retriever:
            self.hybrid = EnsembleRetriever(
                retrievers=[semantic, bm25_retriever],
                weights=[0.6, 0.4],
            )
        else:
            self.hybrid = semantic
        
        # Rerank
        self.reranker = CohereRerank(model=rerank_model, top_n=top_n_rerank)
        self.retriever = ContextualCompressionRetriever(
            base_compressor=self.reranker,
            base_retriever=self.hybrid,
        )
        
        # Prompts
        self._setup_prompts()
    
    def _setup_prompts(self):
        self.rewrite_chain = rewrite_prompt | self.llm | StrOutputParser()
        self.expand_chain = expand_prompt | self.llm | LineParser()
        self.answer_chain = answer_prompt | self.llm | StrOutputParser()
    
    def query(self, question, chat_history=None, use_multi_query=True):
        # 1. Rewrite
        if chat_history:
            history_str = "\n".join([
                f"{m['role']}: {m['content']}" for m in chat_history
            ])
            question = self.rewrite_chain.invoke({
                "chat_history": history_str,
                "question": question
            })
        
        # 2. Retrieve
        if use_multi_query:
            queries = self.expand_chain.invoke({"query": question}) + [question]
            all_docs = [self.retriever.invoke(q) for q in queries]
            docs = dedupe_docs(all_docs)[:10]
        else:
            docs = self.retriever.invoke(question)
        
        # 3. Reorder
        docs = LongContextReorder().transform_documents(docs)
        
        # 4. Format & generate
        context = format_docs_with_citations(docs)
        answer = self.answer_chain.invoke({
            "context": context,
            "question": question
        })
        
        return {
            "question": question,
            "answer": answer,
            "docs": docs,
        }
    
    def stream(self, question, chat_history=None):
        # Stream version
        ...

# Use
rag = ProductionRAG(vectorstore, llm, embeddings)
result = rag.query("AI là gì?", use_multi_query=True)
print(result["answer"])
```

---

## 9. Performance Tips

### 9.1 Caching

```python
from langchain.cache import SQLiteCache
from langchain.globals import set_llm_cache
set_llm_cache(SQLiteCache(database_path=".cache.db"))
```

### 9.2 Async For Throughput

```python
async def rag_async(question):
    docs = await final_retriever.ainvoke(question)
    answer = await answer_chain.ainvoke({
        "context": format_docs(docs),
        "question": question
    })
    return answer

# Process 10 questions song song
import asyncio
results = asyncio.gather(*[rag_async(q) for q in questions])
```

### 9.3 Batch

```python
results = rag_chain.batch([
    {"question": q, "chat_history": []}
    for q in questions
])
```

---

## 10. Monitoring (LangSmith)

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_PROJECT"] = "production-rag"

# Mọi call tự log lên LangSmith
result = rag.query("question")
```

Dashboard hiển thị:
- Mỗi step latency
- Token usage
- Retrieved docs quality
- LLM calls cost

---

## 11. Bài Tập

### Bài 1: Mini Production RAG
Build full pipeline cho 1 use case (vd: Q&A về documentation Python):
- Load docs từ Python docs
- Hybrid + rerank
- Streaming API endpoint

### Bài 2: A/B Test Components
So sánh 4 variants:
- V1: Basic RAG
- V2: + Hybrid search
- V3: + Reranking
- V4: + Multi-query

Đo: faithfulness, answer_relevancy, latency, cost.

### Bài 3: Self-Correcting RAG
Implement validation + retry loop. Test trên 20 câu hỏi tricky.

---

## 12. Checklist Phase 2 - RAG

- [ ] Build basic RAG end-to-end
- [ ] Document loaders đa dạng
- [ ] Chunking strategy phù hợp
- [ ] Vector store production
- [ ] Hybrid retrieval (semantic + BM25)
- [ ] Reranking với Cohere/BGE
- [ ] Query rewriting cho chatbot
- [ ] Contextual compression
- [ ] Production-ready class
- [ ] Monitoring với LangSmith

🎉 **Hoàn thành Phase 2 - RAG!**

➡️ **Tiếp theo**: [Phase 3 - Agents](../03-Agents/01-Tong-Quan-Agents.md)
