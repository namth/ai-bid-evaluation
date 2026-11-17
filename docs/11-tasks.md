# 11 — Milestone & Task

Mỗi task vừa một phiên của agent thực thi (≈ ≤ 10 file). DoD chung cho mọi task: `make lint` xanh, test nêu trong task xanh, không sửa test do Claude viết, cập nhật docs nếu lệch.

Lệnh xác minh chạy từ gốc repo; `T=docker compose -f deploy/docker-compose.test.yml` là môi trường test (Postgres + Redis thật).

## M0 — Nền móng
| ID | Việc | Phụ thuộc | DoD | Lệnh xác minh tự động |
|---|---|---|---|---|
| TASK-001 | Khung backend: `uv`, FastAPI app, `core/config.py` (đọc toàn bộ biến docs/10), `core/errors.py` (định dạng lỗi docs/07), `GET /api/v1/health`, ruff/mypy, `Makefile` (lint, test, e2e, eval) | — | Health trả `{db, redis}`; lỗi chuẩn hóa | `make lint && cd backend && uv run pytest tests/unit/test_errors.py tests/unit/test_config.py -q` |
| TASK-002 | `deploy/docker-compose.yml` (6 service, `mem_limit` docs/05) + `docker-compose.test.yml` + `.env.example` + Dockerfile api/worker | 001 | Stack dev lên được; Postgres có `vector` | `uv run --project backend pytest backend/tests/unit/test_compose_limits.py -q` (TC-31) && `$T up -d --wait && curl -fsS localhost:8000/api/v1/health` |
| TASK-003 | Alembic: toàn bộ bảng docs/06, enum, index HNSW, trigger audit chỉ INSERT, trigger khóa gói COMPLETED, role DB app | 002 | `alembic upgrade head` sạch; downgrade được | `cd backend && uv run pytest tests/integration/test_schema.py -q` (TC-04, TC-05, TC-06 phần trigger) |
| TASK-004 | Auth: argon2, JWT cookie, refresh xoay vòng, khóa đăng nhập, rate limit Redis, kiểm `Origin`, `require_role`, users API, seed admin | 003 | Đăng nhập/làm mới/đăng xuất chạy | `cd backend && uv run pytest tests/integration/test_auth.py tests/integration/test_users.py -q` (TC-26, TC-28) |
| TASK-005 | `services/audit.py`, `services/state_machine.py` (BR-01), packages/bidders/assignments API, `get_package_scoped` (BR-18) | 004 | Mọi chuyển trạng thái có audit | `cd backend && uv run pytest tests/integration/test_state_machine.py tests/integration/test_scope.py -q` (TC-07, TC-08) |
| TASK-006 | Khung frontend Next.js: Tailwind, Shadcn, client OpenAPI, layout + thanh bên, SCR-01, SCR-02 (4 trạng thái UI) | 005 | Đăng nhập → danh sách gói | `cd frontend && pnpm lint && pnpm tsc --noEmit && pnpm build` |
| TASK-007 | Prototype HTML tĩnh SCR-02, 04, 05, 06 trong `prototype/` (dữ liệu giả từ file mẫu, đã ẩn danh) | — | Người dùng duyệt (DEC-023) | `test -f prototype/scr-05.html && npx -y html-validate prototype/*.html` |

## M1 — MVP Mẫu 01 end-to-end
| ID | Việc | Phụ thuộc | DoD | Lệnh xác minh tự động |
|---|---|---|---|---|
| TASK-008 | `core/llm_client.py` (chat + instructor, vision, embedding, `LLM_EXTRA_BODY_JSON`), kiểm chiều embedding khi worker khởi động, `tests/fakes/fake_llm.py` (server OpenAI-compatible giả) | 001 | Chỉ một nơi import `openai` | `cd backend && uv run pytest tests/unit/test_llm_client.py -q` (TC-12, TC-25) |
| TASK-009 | Upload tài liệu: magic bytes, size, PDF mật khẩu, lưu `UPLOAD_DIR`, kiểm đĩa, `GET documents/{id}/pdf` (Range), `pages/{n}` | 005 | Upload/tải qua kiểm quyền | `cd backend && uv run pytest tests/integration/test_documents.py -q` (TC-27, TC-08 phần pdf) |
| TASK-010 | Khung pipeline ARQ: `pipeline_steps`, bước idempotent, retry backoff, phát hiện RUNNING treo, `POST start`, `GET steps`, `POST retry`, tự chuyển EXPERT_REVIEW | 008, 009 | BR-08 đủ | `cd backend && uv run pytest tests/integration/test_pipeline_steps.py -q` (TC-16, TC-17) |
| TASK-011 | Ingest PDF cuốn chiếu (PyMuPDF, pdfplumber từng trang), OCR Vision trang scan, chunk + embedding | 010 | RSS đỉnh ≤ 600 MB với 500 trang | `cd backend && uv run pytest tests/integration/test_ingest.py -q` (TC-09, TC-15) |
| TASK-012 | Bộ tiêu chí: API bộ mẫu + bộ của gói, nhân bản sâu, khóa; seed bộ mẫu "Hàng hóa CNTT — Mẫu 01" lấy từ file mẫu | 005 | BR-15 | `cd backend && uv run pytest tests/integration/test_criteria.py -q` (TC-29) |
| TASK-013 | Đánh giá định tính: retrieval hybrid, Agent (output `CriterionAssessment`), `services/verification.py` (BR-03, BR-04, BR-05), lưu evaluations/citations | 011, 012 | Mỗi tiêu chí Mẫu 01 có KQ AI | `cd backend && uv run pytest tests/unit/test_verification.py tests/integration/test_evaluate_qualitative.py -q` (TC-10, TC-13, TC-14, TC-30) |
| TASK-014 | API evaluations + KQ chuyên gia + submit/approve/return | 013 | BR-02, BR-16 | `cd backend && uv run pytest tests/integration/test_review.py -q` (TC-06, TC-19, TC-20) |
| TASK-015 | Frontend SCR-03, SCR-04 (nhà thầu, upload, tiến độ polling, chạy lại bước) | 006, 010 | 4 trạng thái UI | `cd frontend && pnpm tsc --noEmit && pnpm exec playwright test e2e/packages.spec.ts` (TC-34 phần SCR-04) |
| TASK-016 | Frontend SCR-05 chia đôi: react-pdf, nhảy trang, tô sáng quote, bảng Mẫu 01, form KQ chuyên gia | 014, 015 | Click citation nhảy đúng trang | `cd frontend && pnpm exec playwright test e2e/review.spec.ts` (TC-32) |
| TASK-017 | `ExcelReportGenerator` Mẫu 01 + job xuất + zip + `POST exports`/`GET exports/{id}` | 014 | Bố cục khớp mẫu, KẾT LUẬN bằng công thức | `cd backend && uv run pytest tests/unit/test_excel_mau01.py tests/integration/test_export.py -q` (TC-21, TC-22 phần Mẫu 01) |
| TASK-018 | Frontend SCR-06 tối thiểu: Duyệt / Trả về / Xuất | 016, 017 | Luồng 2 cấp chạy trọn | `cd frontend && pnpm exec playwright test e2e/approval.spec.ts` (TC-33) |

## M2 — Mẫu 02 (định lượng)
| ID | Việc | Phụ thuộc | DoD | Lệnh xác minh tự động |
|---|---|---|---|---|
| TASK-019 | `services/calc_tool.py` (AST an toàn, Decimal, `round_vnd`) + `services/numbers.py` parser số VN | 001 | BR-07, BR-10 | `cd backend && uv run pytest tests/unit/test_calc_tool.py tests/unit/test_numbers.py -q` (TC-01, TC-02, TC-03) |
| TASK-020 | Agent định lượng: tool `calculate`/`search_pages`, lưu `tool_calls`, `calculation_steps`, phát lại BR-06 | 013, 019 | `replay_ok` đúng | `cd backend && uv run pytest tests/unit/test_replay.py tests/integration/test_evaluate_quantitative.py -q` (TC-11) |
| TASK-021 | Bước CONVERT: LibreOffice headless (profile tạm, timeout) cho docx/xlsx | 010 | BR-08 lỗi tất định | `cd backend && uv run pytest tests/integration/test_convert.py -q` (TC-18) |
| TASK-022 | Bảng hợp đồng tương tự (pdfplumber), seed tiêu chí Mẫu 02, Excel Mẫu 02 (`SUM`), tab Mẫu 02 ở SCR-05 | 017, 020 | Bố cục khớp sheet "Mẫu số 02" | `cd backend && uv run pytest tests/unit/test_excel_mau02.py -q` (TC-21, TC-22 phần Mẫu 02) |

## M3 — Mẫu 02B, Kho tri thức, tổng hợp
| ID | Việc | Phụ thuộc | DoD | Lệnh xác minh tự động |
|---|---|---|---|---|
| TASK-023 | Kho tri thức: upload, chunk Điều/Khoản, version/SUPERSEDED, RAG trong Agent định tính, `legal_refs`; SCR-07, SCR-08 | 011, 013 | BR-12 | `cd backend && uv run pytest tests/integration/test_kb.py -q` (TC-24) |
| TASK-024 | Mẫu 02B: schema nhân sự + quá trình công tác, seed tiêu chí, Excel 02B, tab 02B ở SCR-05 | 022 | Bố cục khớp sheet "Mẫu số 02B" | `cd backend && uv run pytest tests/unit/test_excel_mau02b.py -q` (TC-22 phần 02B) |
| TASK-025 | Frontend SCR-09, SCR-10 soạn bộ tiêu chí (cây, kéo thả) | 012 | Khóa khi rời DRAFT | `cd frontend && pnpm exec playwright test e2e/criteria.spec.ts` |
| TASK-026 | `GET summary`, `TongHop.xlsx` (COUNTIF/COUNTIFS/MIN/AVERAGE), SCR-06 đầy đủ | 024 | BR-17 | `cd backend && uv run pytest tests/integration/test_summary.py -q` (TC-23) |
| TASK-027 | Frontend SCR-11 người dùng + nhật ký kiểm toán | 004 | 4 trạng thái UI | `cd frontend && pnpm exec playwright test e2e/admin.spec.ts` (TC-34) |

## M4 — Triển khai production
| ID | Việc | Phụ thuộc | DoD | Lệnh xác minh tự động |
|---|---|---|---|---|
| TASK-028 | Caddyfile (TLS, header bảo mật, `/api` → api), compose production, script sao lưu `pg_dump` + `data/` có xoay vòng | 002 | docs/15 đủ quy trình | `docker compose -f deploy/docker-compose.yml config -q && caddy validate --config deploy/Caddyfile` |
| TASK-029 | `backend/eval/`: bộ hồ sơ gán nhãn (ẩn danh), `make eval` sinh báo cáo `docs/qa/reports/` | 024 | Đo TC-35 | `make eval EVAL_DRY_RUN=1` |
| TASK-030 | `backend/tests/test_docs_sync.py`: đỏ khi route/bảng/biến env/DEC/TASK có trong code mà docs chưa có | 026 | Chuẩn docs tự giữ | `cd backend && uv run pytest tests/test_docs_sync.py -q` |

## Tồn đọng (lỗi / lệch code–docs)
_(trống)_
