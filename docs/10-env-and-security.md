# 10 — Môi trường & Bảo mật

## `.env.example` (giá trị giả)
```dotenv
# Chung
APP_ENV=development              # development | test | production
DOMAIN=dauthau.example.com
ACME_EMAIL=admin@example.com
SECRET_KEY=change-me-64-random-chars
ACCESS_TOKEN_TTL_MIN=15
REFRESH_TOKEN_TTL_DAYS=7
CORS_ORIGINS=http://localhost:3000

# DB / Redis
DATABASE_URL=postgresql+asyncpg://dauthau:change-me@postgres:5432/dauthau
REDIS_URL=redis://redis:6379/0

# LLM (OpenAI-compatible) — DEC-003
LLM_BASE_URL=https://openrouter.ai/api/v1
LLM_API_KEY=sk-or-xxxxxxxx
LLM_MODEL=qwen/qwen-2.5-32b-instruct
LLM_TIMEOUT_S=120
LLM_EXTRA_BODY_JSON=
VISION_MODEL=qwen/qwen-2.5-vl-72b-instruct
EMBEDDING_BASE_URL=
EMBEDDING_API_KEY=
EMBEDDING_MODEL=bge-m3
EMBEDDING_DIM=1024

# Pipeline
WORKER_MAX_JOBS=1
PDF_BATCH_PAGES=8
OCR_DPI=150
EMBED_BATCH_SIZE=16
RETRIEVAL_TOP_K=8
EVAL_CONTEXT_MAX_CHARS=24000
AGENT_MAX_STEPS=8
SOFFICE_TIMEOUT_S=180

# Lưu trữ
UPLOAD_DIR=/data/uploads
EXPORT_DIR=/data/exports
MAX_UPLOAD_MB=200
DISK_MIN_FREE_GB=5

# Seed admin lần đầu
BOOTSTRAP_ADMIN_EMAIL=admin@example.com
BOOTSTRAP_ADMIN_PASSWORD=change-me-strong
```

## Chính sách bảo mật
| Mảng | Quy định |
|---|---|
| Mật khẩu | argon2id; ≥ 12 ký tự; khóa 15 phút sau 5 lần sai liên tiếp |
| Token | Access JWT 15 phút, refresh 7 ngày xoay vòng (hash lưu DB); cookie `httpOnly; Secure; SameSite=Lax`; đổi trạng thái (POST/PUT/PATCH/DELETE) kiểm header `Origin` khớp `DOMAIN` (chống CSRF) |
| Phân quyền | Kiểm ở backend cho mọi endpoint (docs/02, 07); frontend ẩn nút chỉ là UX |
| Rate limit | Đăng nhập 10/phút/IP; API chung 300/phút/người dùng; upload 30/giờ/người dùng (Redis) |
| Upload | Chỉ pdf/docx/xlsx, kiểm **magic bytes** chứ không chỉ đuôi; ≤ 200 MB; lưu tên file sinh ngẫu nhiên, giữ `original_name` trong DB; PDF có mật khẩu → từ chối |
| Đường dẫn | Không ghép path từ input người dùng; tải file chỉ qua `document_id` |
| SSRF | Backend chỉ gọi ra `LLM_BASE_URL`, `EMBEDDING_BASE_URL` từ env; không nhận URL từ người dùng |
| LibreOffice | Chạy với profile tạm riêng, không mạng (`--norestore --nolockcheck`), timeout; macro bị tắt |
| Mạng | Chỉ Caddy publish 80/443; Postgres/Redis trong network nội bộ (DEC-032) |
| Dữ liệu nhạy cảm | E-HSDT chứa giá dự thầu, CCCD, SĐT → không log nội dung; log che email/SĐT; file mẫu thật không commit (DEC-019) |
| Gửi LLM bên thứ ba | **Rủi ro được chấp nhận ở GĐ1** (DEC-033): nội dung gửi OpenRouter không che; giảm rủi ro bằng chuyển Ollama (GĐ2) hoặc bật ZDR qua `LLM_EXTRA_BODY_JSON` |
| Kiểm toán | `audit_logs` chỉ INSERT (DEC-021) |
| Header | Caddy: HSTS, `X-Content-Type-Options`, `Referrer-Policy`, CSP mặc định `self` (cho phép `blob:` cho pdf.js worker) |

## File bí mật — không đọc, không commit
`.env`, `.env.local`, `.env.production`, `deploy/.env`, `data/**`, `samples/**` (trừ README và file đã ẩn danh), khóa SSH, file sao lưu `*.dump`.
