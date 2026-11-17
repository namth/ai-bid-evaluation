# 05 — Kiến trúc

## Tổng thể
```
Trình duyệt ──HTTPS──> Caddy ─┬─ /api/*  → api (FastAPI, uvicorn 1 worker)
                              └─ /*      → web (Next.js standalone)
api ──> Postgres 16 + pgvector      api ──enqueue──> Redis ──> worker (ARQ + LibreOffice)
worker ──> Postgres, đĩa data/uploads, LLM/Embedding/Vision (OpenAI-compatible: OpenRouter | Ollama)
```

## Thành phần & giới hạn bộ nhớ (VPS 4GB, DEC-004, DEC-012)
| Container | Image | `mem_limit` | Ghi chú |
|---|---|---|---|
| caddy | caddy:2-alpine | 64m | TLS Let's Encrypt, chỉ publish 80/443 (DEC-032) |
| web | Next.js standalone (node:20-alpine) | 384m | SSR mỏng, gọi API qua Caddy cùng origin |
| api | python:3.11-slim | 512m | FastAPI, không chứa LibreOffice |
| worker | python:3.11-slim + LibreOffice | 1200m | ARQ, `max_jobs=WORKER_MAX_JOBS` (mặc định 1) (DEC-022, DEC-025, DEC-029) |
| postgres | pgvector/pgvector:pg16 | 768m | `shared_buffers=192MB`, `work_mem=8MB`, `maintenance_work_mem=128MB` |
| redis | redis:7-alpine | 128m | `maxmemory 96mb`, `noeviction` (hàng đợi không được mất job) |
| **Tổng** | | **≈ 3.0 GB** | ~1 GB cho OS và page cache |

## Lớp trừu tượng LLM (DEC-002, DEC-003, DEC-026)
`backend/app/core/llm_client.py` là nơi **duy nhất** khởi tạo client:
- `AsyncOpenAI(base_url=LLM_BASE_URL, api_key=LLM_API_KEY)` bọc bởi `instructor` (mode TOOLS, fallback JSON) cho output Pydantic.
- Ba vai trò model đọc từ env: `LLM_MODEL` (trích xuất + agent), `VISION_MODEL` (OCR), `EMBEDDING_MODEL` (+ `EMBEDDING_BASE_URL`/`EMBEDDING_API_KEY` tùy chọn, mặc định dùng chung LLM).
- `LLM_EXTRA_BODY_JSON` (tùy chọn) cho tham số định tuyến OpenRouter (vd ZDR sau này, DEC-033).
- Không module nào khác được import `openai` trực tiếp (kiểm bằng TC-30).

## Pipeline cuốn chiếu (DEC-005, DEC-006, DEC-030, DEC-039)
- Mở PDF bằng `fitz.open(path)`; đọc theo cụm `PDF_BATCH_PAGES` (mặc định 8) trang; với mỗi trang lấy text, nếu `len(text.strip()) < 20` và trang có ảnh → đánh dấu scan. pdfplumber chỉ mở **từng trang** cần lấy bảng. Mỗi cụm commit DB rồi đóng page/giải phóng; không giữ list toàn bộ trang trong RAM.
- OCR: `page.get_pixmap(dpi=OCR_DPI)` (mặc định 150) → PNG bytes → Vision; giải phóng ngay.
- Embedding theo lô `EMBED_BATCH_SIZE` (mặc định 16) chunk.
- Mỗi bước là một bản ghi `pipeline_steps` (khóa duy nhất `step_type + key`); job ARQ đọc trạng thái, bỏ qua bước `DONE` → chạy lại an toàn.

## Đánh giá theo tiêu chí — Agent (DEC-030, DEC-037, DEC-038)
1. **Truy xuất**: hybrid — top-k (`RETRIEVAL_TOP_K`, mặc định 8) theo cosine trên `chunks` của nhà thầu + từ khóa (`ILIKE`/`tsvector simple`) từ tiêu chí; ghép trang lân cận; giới hạn `EVAL_CONTEXT_MAX_CHARS`.
2. **Agent loop** (tối đa `AGENT_MAX_STEPS`, mặc định 8): LLM với 2 tool: `calculate(expression, inputs)` và `search_pages(query)`; tiêu chí định tính thêm ngữ cảnh RAG từ Kho tri thức (bản ACTIVE, DEC-041).
3. **Output** Pydantic `CriterionAssessment`: `result` (`DAT`/`KHONG_DAT`/`UNCERTAIN`), `reason`, `extracted_fields`, `citations[]`, `calculation_steps[]`, `legal_refs[]`.
4. **Hậu kiểm bằng code** (`services/verification.py`): kiểm citation khớp trang (BR-05), phát lại tool (BR-06), áp quy tắc Cần làm rõ (BR-04) → `ai_result` cuối.

## Tool tính toán an toàn (`services/calc_tool.py`)
- Phân tích bằng `ast.parse(mode="eval")`, chỉ cho phép: số, tên biến có trong `inputs`, `+ - * /`, so sánh `< <= > >= == !=`, `and/or/not`, hàm `sum`, `avg`, `min`, `max`, `round_vnd`, `abs`, list literal.
- Mọi số là `Decimal`; kết quả tiền làm tròn bằng `round_vnd` (ROUND_HALF_UP, DEC-036). Không `eval`, không attribute access, không import; độ dài biểu thức ≤ 500 ký tự.

## Bảo mật luồng file
File phục vụ qua `GET /api/v1/documents/{id}/pdf` có kiểm quyền gói (DEC-028); không có static mount `data/`.
