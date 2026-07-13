# 🎨 Prompt Engineering - Nghệ Thuật Viết Prompt

Prompt Engineering là **kỹ năng cốt lõi** khi làm việc với LLM. Cùng 1 model, prompt khác nhau → kết quả khác xa nhau (accuracy có thể từ 50% lên 95%).

## 📚 Nội Dung

| File | Chủ đề |
|------|--------|
| [01](./01-Tong-Quan-Prompt-Engineering.md) | Tổng quan Prompt Engineering |
| [02](./02-Anatomy-Cua-Prompt.md) | Cấu trúc 1 prompt tốt |
| [03](./03-Basic-Techniques.md) | Zero-shot, Few-shot, Instruction-based |
| [04](./04-Chain-of-Thought.md) | Chain-of-Thought (CoT) |
| [05](./05-Advanced-Reasoning.md) | ToT, Self-Consistency, ReAct, Reflexion, Step-back |
| [06](./06-Role-Personas.md) | Role playing & Personas |
| [07](./07-Output-Formatting.md) | JSON, XML, Structured Output |
| [08](./08-Task-Specific-Prompts.md) | Templates cho task phổ biến |
| [09](./09-Anti-Hallucination.md) | Giảm hallucination, grounding |
| [10](./10-Prompt-Security.md) | Prompt injection, jailbreaking |
| [11](./11-Optimization-AB-Testing.md) | Tối ưu & A/B test prompt |
| [12](./12-Templates-Library.md) | Thư viện templates sẵn dùng |

## 🎯 Mục Tiêu

Sau khi hoàn thành section này, bạn sẽ:
- ✅ Hiểu **cấu trúc** một prompt hiệu quả
- ✅ Áp dụng các kỹ thuật: CoT, Few-shot, ToT, ReAct
- ✅ Viết prompt cho **mọi task**: summarize, classify, extract, code...
- ✅ Giảm **hallucination** và tăng accuracy
- ✅ Phòng chống **prompt injection** và jailbreaking
- ✅ Có **template library** để dùng ngay

## 🔗 Liên Quan

- [01-LangChain-Core/04-Prompts.md](../01-LangChain-Core/04-Prompts.md) - PromptTemplate trong LangChain
- [06-Evaluation/02-Prompt-Evaluation.md](../06-Evaluation/02-Prompt-Evaluation.md) - Eval prompts
- [02-RAG/08-Query-Rewriting.md](../02-RAG/08-Query-Rewriting.md) - Prompts cho RAG

## 💡 Quy Tắc Vàng

> **"Prompt = giao tiếp với LLM. Càng rõ ràng, kết quả càng tốt."**

Trước khi tune model hay thêm tools, hãy thử **fix prompt** trước - đây là cách rẻ và nhanh nhất để tăng chất lượng AI.
