# LLM Concepts - Khái Niệm Cơ Bản Về LLM

## 1. LLM Là Gì?

**LLM (Large Language Model)** là mô hình AI được train trên lượng text khổng lồ, có khả năng:
- Hiểu và sinh ngôn ngữ tự nhiên
- Dịch thuật, tóm tắt, viết code
- Trả lời câu hỏi, lý luận (reasoning)

**Các LLM phổ biến (2026)**:
| Model | Provider | Đặc điểm |
|-------|----------|----------|
| GPT-4o, GPT-4o-mini | OpenAI | Cân bằng giá/hiệu năng |
| Claude Opus 4.7, Sonnet 4.6, Haiku 4.5 | Anthropic | Reasoning tốt, an toàn |
| Gemini 2.0 | Google | Multimodal mạnh |
| Llama 3.3 | Meta | Open source |
| DeepSeek V3 | DeepSeek | Mã nguồn mở, rẻ |

---

## 2. Token - Đơn Vị Cơ Bản

### 2.1 Token Là Gì?

LLM không xử lý chữ cái mà xử lý **token** - đơn vị nhỏ hơn từ.

**Ví dụ tokenization của GPT-4**:
```
"Hello world" → ["Hello", " world"] = 2 tokens
"Xin chào các bạn" → ["Xin", " chào", " các", " bạn"] = 4 tokens
"unbelievable" → ["un", "believ", "able"] = 3 tokens
```

**Quy tắc gần đúng**:
- Tiếng Anh: 1 token ≈ 4 ký tự ≈ 0.75 từ
- Tiếng Việt: tốn nhiều token hơn (do unicode)

### 2.2 Đếm Token

```python
import tiktoken

# Encoding cho GPT-4
enc = tiktoken.encoding_for_model("gpt-4")

text = "Xin chào, tôi đang học AI"
tokens = enc.encode(text)
print(f"Số tokens: {len(tokens)}")
print(f"Tokens: {tokens}")
# Decode lại
print(enc.decode(tokens))
```

### 2.3 Tại Sao Token Quan Trọng?

- **Pricing**: API tính tiền theo token (input + output)
- **Context limit**: Mỗi model có giới hạn token (xem mục 3)
- **Tốc độ**: Càng nhiều token, càng chậm

---

## 3. Context Window

**Context window** = số token tối đa model có thể xử lý trong 1 lần gọi (input + output).

| Model | Context Window |
|-------|----------------|
| GPT-3.5-turbo | 16K tokens |
| GPT-4o | 128K tokens |
| Claude Opus 4.7 (1M) | 1,000,000 tokens |
| Gemini 2.0 | 2M tokens |

**Vấn đề khi vượt context**:
- API trả về lỗi
- Hoặc bị truncate (cắt phần đầu) → mất thông tin

**Giải pháp**:
- Chunking (chia nhỏ document)
- Summarization (tóm tắt context cũ)
- RAG (chỉ retrieve phần liên quan)

---

## 4. Prompt - Đầu Vào Của LLM

### 4.1 Cấu Trúc Prompt

```python
messages = [
    {"role": "system", "content": "Bạn là trợ lý AI thân thiện."},
    {"role": "user", "content": "Hello!"},
    {"role": "assistant", "content": "Chào bạn!"},
    {"role": "user", "content": "Bạn tên gì?"},
]
```

**Vai trò các role**:
- **system**: Chỉ dẫn ban đầu cho model (personality, rules)
- **user**: Câu hỏi/yêu cầu từ user
- **assistant**: Câu trả lời trước đó của model (history)
- **tool**: Kết quả từ tool call (sẽ học ở phase Agents)

### 4.2 Prompt Engineering Cơ Bản

**Bad prompt**:
```
"Viết code"
```

**Good prompt**:
```
"Viết function Python tên `calculate_tax(income: float) -> float` 
tính thuế thu nhập cá nhân Việt Nam theo bậc thang.
Input: thu nhập 1 tháng (VND)
Output: số tiền thuế phải nộp (VND)
Yêu cầu: Có type hints, docstring và xử lý edge case."
```

**Nguyên tắc prompt tốt**:
1. **Cụ thể**: nêu rõ input, output, format
2. **Có ví dụ**: cho 1-2 examples
3. **Phân vai**: "Bạn là chuyên gia X..."
4. **Step-by-step**: yêu cầu suy luận từng bước

---

## 5. Temperature & Sampling Parameters

### 5.1 Temperature

Kiểm soát độ "sáng tạo" của response.

```python
from langchain_openai import ChatOpenAI

# Temperature = 0: deterministic, luôn cho 1 kết quả
llm_strict = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Temperature = 1: cân bằng
llm_balanced = ChatOpenAI(model="gpt-4o-mini", temperature=1)

# Temperature = 2: cực sáng tạo (có thể vô nghĩa)
llm_creative = ChatOpenAI(model="gpt-4o-mini", temperature=2)
```

**Khi nào dùng**:
| Use case | Temperature |
|----------|-------------|
| Code generation | 0 - 0.3 |
| Q&A factual | 0 - 0.3 |
| Tóm tắt | 0.3 - 0.7 |
| Sáng tác văn học | 0.7 - 1.2 |
| Brainstorm idea | 0.8 - 1.5 |

### 5.2 Top-p (Nucleus Sampling)

Chỉ chọn từ top-p% xác suất cao nhất.

```python
llm = ChatOpenAI(temperature=0.7, top_p=0.9)
# Chỉ xét các token có cumulative prob >= 0.9
```

### 5.3 Max Tokens

Giới hạn số token output.

```python
llm = ChatOpenAI(max_tokens=500)  # Output không quá 500 tokens
```

---

## 6. API Key & Authentication

### 6.1 Lấy API Key

**OpenAI**:
- Truy cập https://platform.openai.com/api-keys
- Click "Create new secret key"
- Copy key (sk-...) - **chỉ hiển thị 1 lần!**

**Anthropic**:
- Truy cập https://console.anthropic.com/settings/keys
- Tạo và copy key (sk-ant-...)

### 6.2 Bảo Mật API Key

❌ **KHÔNG BAO GIỜ**:
```python
# SAI - hard code key
llm = ChatOpenAI(api_key="sk-abcd1234...")
```

✅ **NÊN LÀM**:
```python
# 1. Tạo file .env
# OPENAI_API_KEY=sk-...

# 2. Load từ env
from dotenv import load_dotenv
load_dotenv()

# 3. LangChain tự đọc env
llm = ChatOpenAI(model="gpt-4o-mini")
```

**Thêm `.env` vào `.gitignore`**:
```
.env
*.env
.env.local
```

---

## 7. Pricing - Cách LLM Tính Tiền

**Giá điển hình (2026, USD/1M tokens)**:

| Model | Input | Output |
|-------|-------|--------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude Opus 4.7 | $15.00 | $75.00 |
| Claude Sonnet 4.6 | $3.00 | $15.00 |
| Claude Haiku 4.5 | $1.00 | $5.00 |

**Ví dụ tính chi phí**:
```python
# Prompt 1000 tokens, response 500 tokens với GPT-4o
input_cost = 1000 / 1_000_000 * 2.50  # $0.0025
output_cost = 500 / 1_000_000 * 10.00  # $0.005
total = input_cost + output_cost  # $0.0075
```

**Tip giảm chi phí**:
- Dùng model nhỏ (mini, haiku) cho task đơn giản
- **Prompt caching**: cache phần prompt cố định
- **Batch API**: rẻ hơn 50% nhưng chậm hơn

---

## 8. Code Demo Tổng Hợp

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
from langchain.callbacks import get_openai_callback

load_dotenv()

# Khởi tạo LLM
llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0.7,
    max_tokens=500,
)

# Tạo prompt với system + user message
messages = [
    SystemMessage(content="Bạn là chuyên gia Python."),
    HumanMessage(content="Giải thích decorator trong 3 câu."),
]

# Gọi LLM và theo dõi cost
with get_openai_callback() as cb:
    response = llm.invoke(messages)
    print(response.content)
    print(f"\n--- Stats ---")
    print(f"Prompt tokens: {cb.prompt_tokens}")
    print(f"Completion tokens: {cb.completion_tokens}")
    print(f"Total cost: ${cb.total_cost:.4f}")
```

---

## 9. Hallucination - Vấn Đề Lớn Của LLM

**Hallucination** = LLM "bịa" thông tin không có thật nhưng nói rất tự tin.

**Ví dụ**:
```
User: "Ai phát minh ra Python?"
LLM (đúng): "Guido van Rossum"

User: "Cuốn sách 'AI Cho Người Mới Bắt Đầu' do ai viết?"
LLM (hallucination): "Cuốn sách này do tác giả Nguyễn Văn A viết năm 2020"
(thực tế không có cuốn sách này)
```

**Cách giảm hallucination**:
1. **RAG**: cung cấp context thật
2. **Temperature thấp**: 0 - 0.3
3. **Yêu cầu cite nguồn**: "Chỉ trả lời nếu có nguồn trong context"
4. **Chain-of-thought**: yêu cầu suy luận từng bước

---

## 10. Checklist Kiến Thức

- [ ] Hiểu token và biết cách đếm token
- [ ] Hiểu context window và giới hạn
- [ ] Phân biệt được system/user/assistant role
- [ ] Biết cách viết prompt tốt
- [ ] Hiểu temperature và khi nào dùng giá trị nào
- [ ] Biết lưu API key an toàn
- [ ] Hiểu cách tính chi phí LLM
- [ ] Hiểu hallucination và cách giảm

➡️ **Tiếp theo**: [Embeddings & Vectors](./03-Embeddings-Vectors.md)
