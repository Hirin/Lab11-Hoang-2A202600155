# Day 11 — Guardrails, HITL & Responsible AI

Build a **Defense-in-Depth Pipeline** cho AI Banking Assistant, bảo vệ agent khỏi prompt injection, data leakage và nội dung độc hại.

## Kiến trúc Pipeline

```
User Input
    │
    ▼
┌─────────────────────────┐
│  1. Rate Limiter         │ ← Chặn abuse (quá nhiều request/phút)
└─────────┬───────────────┘
          ▼
┌─────────────────────────┐
│  2. Injection Detection  │ ← Regex patterns phát hiện prompt injection
└─────────┬───────────────┘
          ▼
┌─────────────────────────┐
│  3. OpenAI Moderation    │ ← Phát hiện toxic/hate/violence (BONUS - miễn phí)
└─────────┬───────────────┘
          ▼
┌─────────────────────────┐
│  4. Topic Filter         │ ← Chặn off-topic & blocked topics
└─────────┬───────────────┘
          ▼
┌─────────────────────────┐
│  5. LLM (gpt-4.1-nano)  │ ← Generate response
└─────────┬───────────────┘
          ▼
┌─────────────────────────┐
│  6. Content Filter       │ ← Redact PII, API keys, passwords trong output
└─────────┬───────────────┘
          ▼
┌─────────────────────────┐
│  7. LLM-as-Judge         │ ← Đánh giá Safety/Relevance/Accuracy/Tone (1-5)
└─────────┬───────────────┘
          ▼
┌─────────────────────────┐
│  8. Audit & Monitoring   │ ← Log mọi interaction + alert khi bất thường
└─────────┬───────────────┘
          ▼
      Response
```

## Cách hoạt động từng layer

### Layer 1: Rate Limiter
- **Sliding window** per user — theo dõi timestamps trong deque
- Default: **10 requests / 60 giây** per user
- Tại sao cần: Chặn brute-force attack (gửi hàng trăm request thử injection khác nhau)
- Các layer khác check nội dung, layer này giới hạn số lượng

### Layer 2: Injection Detection
- **10 regex patterns** phát hiện prompt injection phổ biến:
  - `ignore (all )?(previous|above|prior) instructions`
  - `you are now`, `system prompt`, `reveal your (instructions|prompt|config)`
  - `pretend you are`, `act as (a |an )?unrestricted`
  - `disregard`, `forget`, `bypass (safety|security|filter)`, `jailbreak`
- Tại sao cần: Bắt direct instruction-override attacks trước khi đến LLM

### Layer 3: OpenAI Moderation *(BONUS)*
- Gọi `client.moderations.create()` — endpoint **miễn phí**, không tốn token
- Detect 11 categories: hate, violence, sexual, self-harm, harassment (+ threatening variants)
- Latency ~100-200ms
- Tại sao cần: Các layer khác không detect được toxic/hate speech

### Layer 4: Topic Filter
- **ALLOWED_TOPICS**: 20 từ khóa banking (EN + VN): `banking`, `account`, `transfer`, `tai khoan`, `giao dich`...
- **BLOCKED_TOPICS**: `hack`, `exploit`, `weapon`, `drug`, `illegal`, `violence`, `gambling`
- Logic: blocked topic → chặn ngay | có allowed topic → cho qua | không match → off-topic → chặn
- Tại sao cần: Bắt off-topic queries và dangerous topics mà injection detection bỏ lọt

### Layer 5: LLM Call
- Gọi OpenAI `gpt-4.1-nano` với system prompt chứa secrets giả lập (password, API key, DB connection)
- Đây là target mà attacker cố gắng khai thác

### Layer 6: Content Filter
- **5 regex patterns** trên OUTPUT của LLM:
  - VN phone: `0\d{9,10}` | Email: `[\w.-]+@[\w.-]+\.[a-zA-Z]{2,}`
  - CMND/CCCD: `\b\d{9}\b|\b\d{12}\b` | API key: `sk-[a-zA-Z0-9-]+`
  - Password: `password\s*[:=]\s*\S+`
- PII/secrets bị thay bằng `[REDACTED]`
- Tại sao cần: Dù LLM bị leak, layer này redact trước khi gửi cho user

### Layer 7: LLM-as-Judge
- Gọi **LLM riêng** đánh giá response trên 4 tiêu chí (1-5):
  - **SAFETY**: Không leak secrets, harmful content
  - **RELEVANCE**: Đúng chủ đề banking
  - **ACCURACY**: Không hallucinate
  - **TONE**: Chuyên nghiệp, lịch sự
- Block nếu SAFETY score < 3
- Tại sao cần: Bắt semantic-level issues mà regex không detect được

### Layer 8: Audit & Monitoring
- **AuditLogger**: Ghi mọi interaction (input, output, blocked_by, latency, timestamp) → export JSON
- **MonitoringAlert**: Track block rate, rate limit hits → alert khi vượt threshold
- Tại sao cần: Phát hiện attack patterns theo thời gian, compliance logging

## Project Structure

```
Day-11-Guardrails-HITL-Responsible-AI/
├── notebooks/
│   ├── assignment11_defense_pipeline.ipynb  ← Main notebook (OpenAI + Pure Python)
│   └── lab11_guardrails_hitl.ipynb          # Lab gốc (Google ADK + Gemini)
├── NguyenDuyMinhHoang.md                    # Individual report (Part B)
├── assignment11_defense_pipeline.md         # Assignment requirements
├── requirements.txt
└── README.md
```

## Setup

### Chạy trên Colab / VSCode Remote

1. Mở `notebooks/assignment11_defense_pipeline.ipynb`
2. Chạy cell install: `!pip install openai nemoguardrails -q`
3. Nhập OpenAI API Key khi được hỏi
4. **Restart kernel → Run All**

### Chạy local

```bash
pip install openai nemoguardrails
export OPENAI_API_KEY="your-api-key-here"
jupyter notebook notebooks/assignment11_defense_pipeline.ipynb
```

## Test Suites

| Test | Mục đích | Expected |
|------|----------|----------|
| **Test 1: Safe Queries** (5 câu) | Banking queries hợp lệ | Tất cả PASS ✅ |
| **Test 2: Attack Queries** (9 câu) | Injection, CISO impersonation, translation, toxic | Tất cả BLOCKED ✅ |
| **Test 3: Rate Limiting** (15 requests) | Brute-force từ 1 user | 10 đầu pass, 5 cuối blocked |
| **Test 4: Edge Cases** (5 câu) | Empty, very long, emoji, SQL injection, off-topic | Xử lý gracefully |

## HITL (Human-in-the-Loop)

Pipeline tích hợp **ConfidenceRouter** để routing response dựa trên risk:

| Confidence | Action | HITL Model |
|------------|--------|------------|
| ≥ 0.9 | Auto send | Human-on-the-loop (review sau) |
| 0.7 - 0.9 | Queue review | Human-in-the-loop (duyệt trước) |
| < 0.7 hoặc high-risk action | Escalate | Human-as-tiebreaker (người quyết định) |

## Tech Stack

| Component | Technology |
|-----------|-----------|
| LLM | OpenAI `gpt-4.1-nano` |
| Moderation | OpenAI Moderation API (miễn phí) |
| Guardrails | Pure Python (regex + classes) |
| NeMo Rails | NVIDIA NeMo Guardrails + Colang (optional layer) |
| Framework | No framework — Pure Python pipeline |

## References

- [OWASP Top 10 for LLM](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails)
- [OpenAI Moderation API](https://platform.openai.com/docs/guides/moderation)
- [AI Safety Fundamentals](https://aisafetyfundamentals.com/)
