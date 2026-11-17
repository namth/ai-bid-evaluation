# CONTEXT — dauthau (AI Bid Evaluation System)

**Là gì:** Hệ thống nội bộ đánh giá E-HSDT theo gói thầu. AI đọc hồ sơ 100–500+ trang, trích xuất có trích dẫn, đề xuất **Đạt / Không đạt / Cần làm rõ** theo tiêu chí; Chuyên viên duyệt, Lãnh đạo duyệt cuối; xuất Excel **Mẫu 01, 02, 02B** cho mỗi nhà thầu cùng bảng **TongHop**. Hệ thống không chấm điểm số (DEC-020).

**Vai trò:** ADMIN · CHUYEN_VIEN (chỉ gói được phân công) · LANH_DAO.

**Stack:** FastAPI + SQLAlchemy async + Alembic + Pydantic v2 + instructor · ARQ/Redis · PostgreSQL 16 + pgvector (bge-m3, 1024 chiều) · PyMuPDF + pdfplumber + LibreOffice (docx/xlsx→PDF) · OpenPyXL · Next.js App Router + TS + Tailwind + Shadcn + react-pdf · docker-compose có `mem_limit`, Caddy HTTPS · LLM OpenAI-compatible: OpenRouter (GĐ1) → Ollama (GĐ2) chỉ bằng `.env`.

**Hạ tầng:** VPS 4GB RAM, không GPU. Pipeline đọc cuốn chiếu 8 trang/cụm.

## Bất biến (không bao giờ vi phạm — mỗi cái có test)
| ID | Bất biến | TC |
|---|---|---|
| INV-1 | LLM không tự tính: mọi phép số học/so sánh qua tool tính an toàn; code phát lại log tool, lệch → Cần làm rõ | TC-01, TC-11 |
| INV-2 | Mọi dữ liệu trích xuất có citation `file_name` + `page_number` + `exact_quote` đã xác minh; thiếu → Cần làm rõ | TC-10, TC-13 |
| INV-3 | Tiền = số nguyên đồng (`bigint`), `Decimal` + ROUND_HALF_UP sau mỗi phép tính; không float | TC-02, TC-04 |
| INV-4 | `audit_logs` chỉ INSERT | TC-05 |
| INV-5 | Gói COMPLETED không sửa được | TC-06 |
| INV-6 | Không nạp toàn bộ PDF vào RAM | TC-09 |
| INV-7 | Chỉ `core/llm_client.py` gọi LLM; không hard-code nhà cung cấp | TC-12 |
| INV-8 | CHUYEN_VIEN ngoài phân công → 404 | TC-08 |

## Bản đồ docs
01 tổng quan · 02 phân hệ + RBAC · 03 màn hình SCR-01…11 · 04 luồng · 05 kiến trúc · 06 schema · 07 API · 08 tích hợp · 09 quy chuẩn · 10 env + bảo mật · 11 task TASK-001…030 · 12 quy tắc BR-01…18 · 13 test TC-01…35 · **14 quyết định (thắng mọi chỗ mâu thuẫn)** · 15 vận hành.

Mẫu Excel gốc: `File danh gia mau.xlsx` (chứa dữ liệu cá nhân thật — không commit, DEC-019).
