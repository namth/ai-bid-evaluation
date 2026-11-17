# 08 — Tích hợp bên thứ ba

| Tích hợp | Dùng cho | Cấu hình | Lỗi & fallback |
|---|---|---|---|
| OpenRouter (GĐ1) / Ollama (GĐ2) — LLM | Agent trích xuất + đánh giá | `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_MODEL` (mặc định `qwen/qwen-2.5-32b-instruct`) | Timeout `LLM_TIMEOUT_S`=120; 429/5xx/timeout retry 3 lần backoff mũ (2s, 8s, 32s) rồi bước LOI (DEC-039) |
| Vision model | OCR trang scan | `VISION_MODEL` (vd `qwen/qwen-2.5-vl-72b-instruct`; Ollama `qwen2.5vl`) | Như trên; OCR lỗi → trang text rỗng, citation trang đó sẽ không khớp → Cần làm rõ |
| Embedding | Chỉ mục E-HSDT + KB | `EMBEDDING_MODEL=bge-m3`, `EMBEDDING_DIM=1024`, `EMBEDDING_BASE_URL?`, `EMBEDDING_API_KEY?` | Khi khởi động, worker gọi thử 1 embedding: số chiều ≠ `EMBEDDING_DIM` → dừng với lỗi rõ (DEC-026) |
| LibreOffice headless | docx/xlsx → PDF | trong image worker; `SOFFICE_TIMEOUT_S`=180 | Timeout/lỗi → bước CONVERT LOI, không retry tự động (lỗi tất định) |
| Let's Encrypt (qua Caddy) | TLS | `DOMAIN`, `ACME_EMAIL` | — |

## Hợp đồng gọi LLM
- Chỉ qua `app/core/llm_client.py` (DEC-002); `instructor` ép output vào model Pydantic; output sai schema → instructor tự hỏi lại tối đa 2 lần, vẫn sai → `CAN_LAM_RO` với lý do `SCHEMA_INVALID`.
- Prompt có `prompt_version` lưu vào `evaluations` để truy vết.
- Không log nội dung prompt/response ra stdout (DEC-033); chỉ log model, số token, thời gian.
- Ghi chú hiệu năng: OpenRouter phải chọn nhà cung cấp hỗ trợ tool calling cho `LLM_MODEL`; nếu model không hỗ trợ tool, instructor chạy mode JSON và Agent mô phỏng tool call bằng trường `tool_requests` (cùng kiểm tra phát lại).

## Chuyển GĐ2 (Ollama)
Chỉ đổi `.env`:
```
LLM_BASE_URL=http://ollama:11434/v1
LLM_API_KEY=ollama
LLM_MODEL=qwen2.5:32b-instruct
VISION_MODEL=qwen2.5vl:7b
EMBEDDING_MODEL=bge-m3
```
Ollama cần máy có GPU/RAM lớn — không chạy trên VPS 4GB.
