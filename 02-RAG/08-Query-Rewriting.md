# Query Rewriting - Viết Lại Câu Hỏi

## 1. Tại Sao Cần Query Rewriting?

User query thường:
- ❌ Quá ngắn: "iPhone?"
- ❌ Có lỗi chính tả: "lang chain framewark"
- ❌ Mơ hồ: "Cái đó là gì?"
- ❌ Có context phụ thuộc: "Còn cái này?"
- ❌ Multiple intents: "Python vs Java và cái nào dễ học hơn?"

→ Cần **biến query thô thành query tốt** trước khi search.

---

## 2. Multi-Query Generation

### 2.1 Ý Tưởng

LLM sinh ra **nhiều biến thể** của query → search với từng cái → gộp kết quả.

```
Original: "AI là gì?"
↓ LLM
Variants:
1. "Định nghĩa Artificial Intelligence"
2. "Trí tuệ nhân tạo bao gồm những gì?"
3. "AI khác Machine Learning như thế nào?"
↓ Search mỗi query
↓ Merge results
```

### 2.2 LangChain MultiQueryRetriever

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm,
)

# Debug để xem queries sinh ra
import logging
logging.basicConfig()
logging.getLogger("langchain.retrievers.multi_query").setLevel(logging.INFO)

docs = retriever.invoke("AI là gì?")
```

### 2.3 Custom Prompt

```python
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser

class LineListOutputParser(StrOutputParser):
    def parse(self, text: str) -> List[str]:
        return [line.strip() for line in text.strip().split("\n") if line.strip()]

QUERY_PROMPT = PromptTemplate(
    input_variables=["question"],
    template="""Sinh ra 5 biến thể khác nhau của câu hỏi sau để search trong vector database.
Mục đích: tăng coverage retrieval bằng cách rephrase câu hỏi từ nhiều góc độ.

Câu hỏi gốc: {question}

5 biến thể (mỗi dòng 1 query):"""
)

llm_chain = QUERY_PROMPT | llm | LineListOutputParser()

retriever = MultiQueryRetriever(
    retriever=vectorstore.as_retriever(),
    llm_chain=llm_chain,
)
```

---

## 3. HyDE (Hypothetical Document Embedding)

### 3.1 Ý Tưởng

Thay vì embed query, ta:
1. LLM tạo ra **giả định câu trả lời** (hypothetical answer)
2. Embed câu trả lời đó
3. Search với embedding của answer

**Tại sao**: embedding của answer thường gần hơn với real documents (cùng "shape").

### 3.2 Implementation

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda

# 1. Generate hypothetical answer
hyde_prompt = ChatPromptTemplate.from_template("""
Hãy viết một đoạn văn 100 từ trả lời câu hỏi sau (kể cả nếu bạn không chắc):

Câu hỏi: {question}

Đoạn văn:
""")

hyde_chain = hyde_prompt | llm | StrOutputParser()

# 2. Tạo retriever dùng hypothetical doc
def hyde_search(question: str):
    hypothetical_doc = hyde_chain.invoke({"question": question})
    return vectorstore.similarity_search(hypothetical_doc, k=5)

# 3. Wrap thành Runnable
hyde_retriever = RunnableLambda(hyde_search)

# 4. Use
docs = hyde_retriever.invoke("AI là gì?")
```

### 3.3 LangChain Native HyDE

```python
from langchain.chains import HypotheticalDocumentEmbedder

embeddings = OpenAIEmbeddings()

hyde_embeddings = HypotheticalDocumentEmbedder.from_llm(
    llm,
    embeddings,
    prompt_key="web_search",  # Hoặc "sci_fact", "arguana", custom
)

# Dùng cho vector store
vectorstore = Chroma(embedding_function=hyde_embeddings)
```

---

## 4. Query Decomposition

### 4.1 Sub-Question Decomposition

Câu hỏi phức tạp → chia thành sub-questions.

```python
decompose_prompt = ChatPromptTemplate.from_template("""
Chia câu hỏi phức tạp sau thành các câu hỏi nhỏ hơn, độc lập nhau:

Câu hỏi: {question}

Output (mỗi câu 1 dòng):
""")

decompose_chain = decompose_prompt | llm | LineListOutputParser()

# "So sánh Python và Java về tốc độ và độ phổ biến"
# →
# 1. Python tốc độ thế nào?
# 2. Java tốc độ thế nào?
# 3. Python độ phổ biến?
# 4. Java độ phổ biến?

sub_questions = decompose_chain.invoke({"question": "So sánh Python và Java"})

# Search từng sub-question
all_docs = []
for q in sub_questions:
    docs = retriever.invoke(q)
    all_docs.extend(docs)

# Hoặc parallel
all_docs = retriever.batch(sub_questions)
```

### 4.2 Sequential Decomposition

Mỗi sub-question dùng kết quả của câu trước:

```
"Năm nay tôi 30 tuổi. Khi tôi 50 tuổi, GDP Việt Nam sẽ bằng bao nhiêu?"

Step 1: Năm nay là năm nào? → 2025
Step 2: 30 + 20 = 50, vậy năm đó là 2045
Step 3: Dự báo GDP Việt Nam năm 2045?
```

```python
def sequential_decompose(question, llm, retriever):
    # Plan: chia thành step
    plan_prompt = "Lập kế hoạch giải câu hỏi này thành các bước: {q}"
    steps = ...
    
    # Execute từng step
    context = ""
    for step in steps:
        # Search với context
        docs = retriever.invoke(f"{context} {step}")
        answer = llm.invoke(f"Context: {docs}\nStep: {step}").content
        context += f"\n{step}: {answer}"
    
    return context
```

---

## 5. Step-Back Prompting

Trước khi search, "lùi lại" hỏi câu tổng quát hơn:

```python
stepback_prompt = ChatPromptTemplate.from_template("""
Cho câu hỏi cụ thể, hãy đặt 1 câu hỏi tổng quát hơn để tìm context background.

Câu hỏi cụ thể: {question}
Câu hỏi tổng quát:
""")

# "Cú pháp f-string trong Python 3.12 có gì mới?"
# → "Cú pháp f-string trong Python hoạt động ra sao?"

stepback_chain = stepback_prompt | llm | StrOutputParser()

# Sau đó search cả 2 query
specific_docs = retriever.invoke(specific_question)
general_docs = retriever.invoke(stepback_chain.invoke({"question": specific_question}))

all_context = specific_docs + general_docs
```

---

## 6. Query Rewriting Với History

Khi user hỏi tiếp theo (follow-up), query phụ thuộc context:

```python
rewrite_prompt = ChatPromptTemplate.from_template("""
Cho lịch sử hội thoại và câu hỏi mới. Hãy viết lại câu hỏi mới thành câu standalone (không phụ thuộc context).

Lịch sử:
{chat_history}

Câu hỏi mới: {question}

Câu hỏi standalone:
""")

rewrite_chain = rewrite_prompt | llm | StrOutputParser()

# Lịch sử:
# User: "Python là gì?"
# AI: "Python là ngôn ngữ lập trình..."

# User: "Ai tạo ra nó?"
# → Rewrite: "Ai tạo ra Python?"

standalone = rewrite_chain.invoke({
    "chat_history": "User: Python là gì?\nAI: Python là ngôn ngữ lập trình...",
    "question": "Ai tạo ra nó?"
})
```

### LangChain Helper

```python
from langchain.chains import create_history_aware_retriever

history_aware_retriever = create_history_aware_retriever(
    llm,
    retriever,
    rewrite_prompt,
)

# Sử dụng
docs = history_aware_retriever.invoke({
    "chat_history": history,
    "input": "Ai tạo ra nó?"
})
```

---

## 7. Query Expansion

Thêm từ liên quan vào query:

```python
expansion_prompt = ChatPromptTemplate.from_template("""
Cho câu hỏi, hãy thêm các synonyms và related terms để tăng coverage search.

Query gốc: {query}

Query mở rộng (giữ ý chính, thêm related terms):
""")

# "ô tô điện"
# → "ô tô điện EV electric vehicle xe hơi pin lithium tesla"

expand_chain = expansion_prompt | llm | StrOutputParser()
expanded = expand_chain.invoke({"query": "ô tô điện"})

# Search với query mở rộng
docs = retriever.invoke(expanded)
```

---

## 8. Spell Correction

```python
spell_prompt = ChatPromptTemplate.from_template("""
Sửa lỗi chính tả và ngữ pháp. Giữ ý nghĩa gốc.

Original: {query}
Corrected:
""")

spell_chain = spell_prompt | llm | StrOutputParser()

# "lang chain framewark" → "langchain framework"
corrected = spell_chain.invoke({"query": "lang chain framewark"})
```

---

## 9. Intent Routing

Phân loại query → route đến retriever phù hợp:

```python
from langchain_core.runnables import RunnableBranch

route_prompt = ChatPromptTemplate.from_template("""
Phân loại query thuộc category nào: 
- tech (lập trình, công nghệ)
- food (ẩm thực)
- travel (du lịch)

Query: {query}
Category (1 từ):
""")

route_chain = route_prompt | llm | StrOutputParser()

def get_retriever_by_category(input):
    category = route_chain.invoke({"query": input["query"]}).strip().lower()
    if category == "tech":
        return tech_retriever
    elif category == "food":
        return food_retriever
    elif category == "travel":
        return travel_retriever
    return general_retriever

# Use
from langchain_core.runnables import RunnableLambda

router = RunnableLambda(get_retriever_by_category)

# Pipeline
result = router.invoke({"query": "Cách nấu phở"})
```

---

## 10. Demo Tổng Hợp: Advanced RAG Pipeline

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(embedding_function=embeddings, persist_directory="./db")
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# 1. Spell correction + standalone
preprocess_prompt = ChatPromptTemplate.from_template("""
Cho lịch sử và query mới. Sửa chính tả và biến thành câu standalone.

History:
{history}

Query: {query}

Output (chỉ trả câu standalone, không giải thích):
""")

preprocess_chain = preprocess_prompt | llm | StrOutputParser()

# 2. Multi-query expansion
expand_prompt = ChatPromptTemplate.from_template("""
Sinh 3 biến thể của query để search đa chiều.

Query: {query}

3 variants (mỗi dòng 1 query):
""")

class LineParser(StrOutputParser):
    def parse(self, text):
        return [l.strip() for l in text.split("\n") if l.strip()]

expand_chain = expand_prompt | llm | LineParser()

# 3. Full pipeline
def advanced_rag(query, history):
    # Step 1: preprocess
    standalone = preprocess_chain.invoke({"history": history, "query": query})
    print(f"Standalone: {standalone}")
    
    # Step 2: expand to multiple queries
    queries = expand_chain.invoke({"query": standalone})
    queries.append(standalone)  # Include original
    print(f"Queries: {queries}")
    
    # Step 3: retrieve for each
    all_docs = []
    for q in queries:
        docs = retriever.invoke(q)
        all_docs.extend(docs)
    
    # Step 4: dedupe
    seen = set()
    unique_docs = []
    for d in all_docs:
        if d.page_content not in seen:
            seen.add(d.page_content)
            unique_docs.append(d)
    
    # Step 5: generate answer
    context = "\n".join(d.page_content for d in unique_docs[:10])
    answer_prompt = f"Context:\n{context}\n\nQuestion: {standalone}\n\nAnswer:"
    answer = llm.invoke(answer_prompt).content
    
    return answer

# Test
history = "User: Python là gì?\nAI: Python là ngôn ngữ lập trình."
result = advanced_rag("Ai tạo ra nó?", history)
print(result)
```

---

## 11. Khi Nào Cần Query Rewriting?

### Cần
- ✅ Multi-turn chatbot (follow-up questions)
- ✅ Domain technical (cần normalize terminology)
- ✅ User query ngắn/mơ hồ
- ✅ Khi có spelling errors

### Có Thể Skip
- ❌ Single-turn API queries
- ❌ User là chuyên gia (biết term chính xác)
- ❌ Latency critical

---

## 12. Trade-off

| Technique | Cost | Improvement | Latency |
|-----------|------|-------------|---------|
| Multi-Query | +N LLM calls | +5-10% recall | +500-1000ms |
| HyDE | +1 LLM call | +5-15% trên domain xa | +300-500ms |
| Decomposition | +N LLM calls | +20% cho complex Q | +1-2s |
| History rewrite | +1 LLM call | Critical cho chatbot | +200ms |
| Spell correction | +1 LLM call | +5% cho noisy input | +200ms |

---

## 13. Bài Tập

### Bài 1: Multi-Query Benchmark
So sánh recall@5 giữa:
- No rewriting
- Multi-query (3 variants)
- HyDE
- Multi-query + HyDE

Trên 50 test queries.

### Bài 2: Chatbot Với History
Build chatbot có history rewrite. Test với conversation 5-10 turn.

### Bài 3: Complex Question Decomposition
Cho 10 câu hỏi phức tạp ("So sánh X và Y về Z và W"). Implement decomposition + parallel retrieval + synthesis.

---

## 14. Checklist

- [ ] MultiQueryRetriever
- [ ] HyDE (hypothetical document)
- [ ] Query decomposition
- [ ] Step-back prompting
- [ ] History-aware rewriting
- [ ] Query expansion
- [ ] Intent routing

➡️ **Tiếp theo**: [Contextual Compression](./09-Contextual-Compression.md)
