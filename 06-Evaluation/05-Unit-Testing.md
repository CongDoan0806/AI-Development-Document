# Unit Testing Cho AI App

## 1. Thử Thách Khi Test AI

Tradinational testing:
```python
def test_add():
    assert add(2, 3) == 5  # Deterministic
```

AI testing:
```python
def test_llm():
    response = llm.invoke("Hi")
    assert ???  # Non-deterministic output
```

**Strategies**:
1. **Mock LLM** cho unit tests
2. **Snapshot test** cho semantic
3. **Property-based**: test invariants
4. **Integration test** với real LLM (slow)

---

## 2. Mock LLM

### 2.1 FakeListLLM

```python
from langchain_community.llms.fake import FakeListLLM
from langchain_core.language_models.fake import FakeListChatModel

# LLM trả về các response định sẵn
fake_llm = FakeListChatModel(responses=[
    "Response 1",
    "Response 2",
    "Response 3",
])

# Test
chain = prompt | fake_llm
result = chain.invoke({"input": "test"})
assert result.content == "Response 1"
```

### 2.2 Custom Mock

```python
from unittest.mock import MagicMock
from langchain_core.messages import AIMessage

mock_llm = MagicMock()
mock_llm.invoke.return_value = AIMessage(content="mocked response")

# Test code that uses llm
result = my_function(mock_llm)
mock_llm.invoke.assert_called_once_with("expected prompt")
```

### 2.3 Mock Tool Calling

```python
from langchain_core.messages import AIMessage

mock_response = AIMessage(
    content="",
    tool_calls=[
        {
            "name": "search",
            "args": {"query": "AI"},
            "id": "call_1",
        }
    ],
)

mock_llm.invoke.return_value = mock_response
```

---

## 3. Test Pure Logic

```python
# parser.py
def extract_email(text: str) -> str | None:
    import re
    match = re.search(r'[\w\.-]+@[\w\.-]+', text)
    return match.group(0) if match else None

# test_parser.py
def test_extract_email():
    assert extract_email("Contact me at a@b.com") == "a@b.com"
    assert extract_email("No email here") is None
```

→ Logic không phụ thuộc LLM, test bình thường.

---

## 4. Test Prompt Building

```python
from langchain_core.prompts import ChatPromptTemplate

def build_qa_prompt(context: str, question: str):
    template = ChatPromptTemplate.from_template(
        "Context: {context}\nQ: {question}\nA:"
    )
    return template.format(context=context, question=question)

def test_build_prompt():
    result = build_qa_prompt("Python info", "What is Python?")
    assert "Python info" in result
    assert "What is Python?" in result
    assert result.startswith("Human:") or "Context:" in result
```

---

## 5. Test Output Parsers

```python
from langchain_core.output_parsers import JsonOutputParser

def test_json_parser():
    parser = JsonOutputParser()
    
    # Valid
    result = parser.parse('{"name": "An", "age": 25}')
    assert result["name"] == "An"
    
    # Invalid
    with pytest.raises(Exception):
        parser.parse("not json")
```

---

## 6. Test Tools

```python
@tool
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

def test_add_tool():
    # Test trực tiếp như function
    assert add.invoke({"a": 2, "b": 3}) == 5
    
    # Test schema
    schema = add.args_schema.model_json_schema()
    assert "a" in schema["properties"]
    assert "b" in schema["properties"]
```

---

## 7. Test Chains Với Mock

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

def build_chain(llm):
    prompt = ChatPromptTemplate.from_template("Hello {name}")
    return prompt | llm | StrOutputParser()

def test_chain_with_mock():
    mock_llm = FakeListChatModel(responses=["Hi An"])
    chain = build_chain(mock_llm)
    
    result = chain.invoke({"name": "An"})
    assert result == "Hi An"
```

---

## 8. Test Agents Với Mock

```python
from langgraph.prebuilt import create_react_agent

def test_agent_flow():
    # Mock LLM trả về tool call rồi answer
    mock_llm = MagicMock()
    
    # First call: tool call
    mock_llm.invoke.side_effect = [
        AIMessage(
            content="",
            tool_calls=[{"name": "add", "args": {"a": 2, "b": 3}, "id": "1"}]
        ),
        # Second call: final answer
        AIMessage(content="The result is 5"),
    ]
    
    # Add bind_tools method
    mock_llm.bind_tools = MagicMock(return_value=mock_llm)
    
    agent = create_react_agent(mock_llm, [add])
    result = agent.invoke({"messages": [("user", "2+3")]})
    
    assert "5" in result["messages"][-1].content
```

---

## 9. Snapshot Testing

Save expected output as snapshot, compare on each run:

```python
# pip install pytest-snapshot
def test_summary_snapshot(snapshot):
    summary = generate_summary("Long text...")
    
    # First run: save as snapshot
    # Subsequent runs: compare
    snapshot.assert_match(summary, "summary.txt")
```

Update snapshot khi cần:
```bash
pytest --snapshot-update
```

---

## 10. Semantic Assertions

Output LLM khác nhau nhưng nghĩa giống:

```python
from langchain_openai import OpenAIEmbeddings
import numpy as np

embeddings = OpenAIEmbeddings()

def assert_semantic_match(actual: str, expected: str, threshold=0.85):
    v1 = embeddings.embed_query(actual)
    v2 = embeddings.embed_query(expected)
    sim = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
    assert sim > threshold, f"Semantic similarity {sim} below {threshold}"

# Use
def test_semantic():
    response = llm.invoke("Capital of France?").content
    assert_semantic_match(response, "Paris is the capital of France")
```

---

## 11. Property-Based Testing

Test invariants thay vì exact values:

```python
from hypothesis import given, strategies as st

@given(st.text(min_size=1, max_size=1000))
def test_summarizer_invariants(input_text):
    summary = summarize(input_text)
    
    # Invariants
    assert len(summary) <= len(input_text)  # Shorter
    assert len(summary) > 0  # Non-empty
    # Hard to check meaning, but format invariants OK
```

---

## 12. Integration Tests Với Real LLM

```python
# pytest mark slow tests
@pytest.mark.slow
@pytest.mark.integration
def test_with_real_llm():
    """Slow, runs only with --slow flag"""
    real_llm = ChatOpenAI(model="gpt-4o-mini")
    chain = build_chain(real_llm)
    result = chain.invoke({"name": "An"})
    
    # Loose assertions
    assert len(result) > 0
    assert "An" in result or "an" in result.lower()
```

```ini
# pytest.ini
[pytest]
markers =
    slow: slow tests (deselect with '-m "not slow"')
    integration: integration tests
```

Run:
```bash
pytest                    # Skip slow
pytest -m slow             # Run slow only
pytest --co                # Collect (no run)
```

---

## 13. RAG Testing

### 13.1 Test Retriever

```python
def test_retriever_returns_relevant():
    docs = retriever.invoke("LangChain framework")
    assert len(docs) > 0
    
    # Top doc nên contain keyword
    assert "langchain" in docs[0].page_content.lower()
```

### 13.2 Test Embedding

```python
def test_embedding_dimensions():
    vec = embeddings.embed_query("test")
    assert len(vec) == 1536  # text-embedding-3-small
    assert all(isinstance(x, float) for x in vec)

def test_embedding_similarity():
    v1 = embeddings.embed_query("dog")
    v2 = embeddings.embed_query("cat")
    v3 = embeddings.embed_query("Python programming")
    
    # Dog-cat closer than dog-python
    sim_animals = cosine_sim(v1, v2)
    sim_diff = cosine_sim(v1, v3)
    assert sim_animals > sim_diff
```

---

## 14. Fixtures

```python
import pytest
from langchain_openai import ChatOpenAI
from langchain_chroma import Chroma

@pytest.fixture
def mock_llm():
    return FakeListChatModel(responses=["mock"])

@pytest.fixture
def sample_docs():
    return [
        Document(page_content="Python info"),
        Document(page_content="JavaScript info"),
    ]

@pytest.fixture
def vectorstore(sample_docs):
    return Chroma.from_documents(sample_docs, OpenAIEmbeddings())

def test_search(vectorstore):
    results = vectorstore.similarity_search("Python", k=1)
    assert "Python" in results[0].page_content
```

---

## 15. Demo: Full Test Suite

```python
# test_rag.py
import pytest
from unittest.mock import MagicMock
from langchain_core.documents import Document
from langchain_core.messages import AIMessage

@pytest.fixture
def docs():
    return [
        Document(page_content="LangChain is AI framework"),
        Document(page_content="RAG combines retrieval and generation"),
    ]

@pytest.fixture
def mock_retriever(docs):
    retriever = MagicMock()
    retriever.invoke = MagicMock(return_value=docs[:1])
    return retriever

@pytest.fixture
def mock_llm():
    llm = MagicMock()
    llm.invoke = MagicMock(return_value=AIMessage(
        content="LangChain is an AI framework"
    ))
    return llm

def test_rag_basic(mock_retriever, mock_llm):
    from my_rag import RAGChain
    
    rag = RAGChain(retriever=mock_retriever, llm=mock_llm)
    result = rag.invoke("What is LangChain?")
    
    # Test retriever was called
    mock_retriever.invoke.assert_called_once_with("What is LangChain?")
    
    # Test answer mentions framework
    assert "framework" in result.lower()

def test_rag_empty_results(mock_llm):
    empty_retriever = MagicMock()
    empty_retriever.invoke = MagicMock(return_value=[])
    
    rag = RAGChain(retriever=empty_retriever, llm=mock_llm)
    result = rag.invoke("Unknown topic")
    
    # Should still work, just with empty context
    assert result is not None
```

---

## 16. Best Practices

✅ **Unit test pure logic** với mocks - fast feedback

✅ **Integration test sparingly** - cost & slow

✅ **Snapshot for regression**

✅ **Semantic assertions** thay vì exact match

✅ **Fixtures cho reuse**

✅ **Mark slow/integration tests** để skip trong dev

✅ **Coverage > 80% cho pure logic**

---

## 17. Checklist

- [ ] Mock LLM với FakeListChatModel
- [ ] Test prompt builders
- [ ] Test output parsers
- [ ] Test tools
- [ ] Snapshot testing
- [ ] Semantic assertions
- [ ] Property-based tests
- [ ] Integration test với real LLM
- [ ] Fixtures setup

➡️ **Tiếp theo**: [Regression Testing](./06-Regression-Testing.md)
