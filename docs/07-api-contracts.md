# 07 — Hợp đồng API

Tiền tố `/api/v1` (DEC-034). JSON UTF-8. Xác thực bằng cookie `access_token` (httpOnly) — DEC-027. Tiền trả dạng **chuỗi số nguyên** (`"432300000"`) để tránh mất chính xác ở JS. Ngày `YYYY-MM-DD`, thời điểm ISO 8601 UTC.

## Định dạng lỗi
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "Dữ liệu không hợp lệ", "details": { "code": ["Mã gói thầu đã tồn tại"] } } }
```

## Bảng mã lỗi
| HTTP | code | Khi nào |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Body/query sai |
| 401 | `UNAUTHENTICATED` | Thiếu/hết hạn token |
| 401 | `INVALID_CREDENTIALS` | Sai email/mật khẩu |
| 403 | `FORBIDDEN` | Sai vai trò |
| 404 | `PACKAGE_NOT_FOUND` | Không có hoặc ngoài phạm vi phân công (DEC-017) |
| 404 | `NOT_FOUND` | Tài nguyên khác |
| 409 | `INVALID_STATE` | Thao tác không hợp lệ ở trạng thái hiện tại (BR-01, BR-02) |
| 409 | `DUPLICATE` | Vi phạm unique |
| 409 | `REVIEW_INCOMPLETE` | Gửi lãnh đạo khi còn tiêu chí chưa có KQ chuyên gia |
| 413 | `FILE_TOO_LARGE` | > 200 MB |
| 415 | `UNSUPPORTED_FILE` | Không phải pdf/docx/xlsx |
| 422 | `COMMENT_REQUIRED` | KQ chuyên gia ≠ KQ AI mà thiếu nhận xét (DEC-040) |
| 429 | `RATE_LIMITED` | Vượt rate limit / khóa đăng nhập |
| 507 | `DISK_FULL` | Đĩa còn < `DISK_MIN_FREE_GB` |

## Endpoint
Cột Quyền: A=ADMIN, C=CHUYEN_VIEN (P = chỉ gói được phân công), L=LANH_DAO.

| Method | Path | Quyền | Mô tả |
|---|---|---|---|
| POST | `/auth/login` | công khai | `{email,password}` → 200 + set cookie; 5 lần sai/15 phút → khóa 15 phút |
| POST | `/auth/refresh` | cookie refresh | Xoay vòng refresh token |
| POST | `/auth/logout` | đã đăng nhập | Thu hồi refresh |
| GET | `/auth/me` | đã đăng nhập | `{id,email,full_name,role}` |
| GET/POST | `/users` | A | Danh sách / tạo |
| PATCH | `/users/{id}` | A | Đổi vai trò, khóa/mở, đặt lại mật khẩu |
| GET | `/audit-logs` | A | Lọc `actor_id, entity, from, to`, phân trang |
| GET | `/packages` | A, L, C(P) | Lọc `status, q, page` |
| POST | `/packages` | A, C | Tạo + nhân bản bộ tiêu chí mẫu `template_set_id` |
| GET | `/packages/{id}` | A, L, C(P) | Chi tiết + nhà thầu + tóm tắt tiến độ |
| PATCH | `/packages/{id}` | A, C(P) | Chỉ khi DRAFT |
| PUT | `/packages/{id}/assignments` | A | `{user_ids:[]}` |
| POST | `/packages/{id}/bidders` | A, C(P) | DRAFT |
| PATCH/DELETE | `/packages/{id}/bidders/{bid}` | A, C(P) | DRAFT |
| POST | `/packages/{id}/documents` | A, C(P) | multipart `file`, `kind`, `bidder_id?`; DRAFT |
| GET | `/documents/{id}/pdf` | theo quyền gói / mọi vai trò cho KB | Stream PDF (hỗ trợ Range) |
| GET | `/documents/{id}/pages/{n}` | như trên | Text trang (để tô sáng trích dẫn) |
| POST | `/packages/{id}/start` | A, C(P) | DRAFT → PROCESSING, enqueue |
| GET | `/packages/{id}/steps` | A, L, C(P) | Bước pipeline + trạng thái |
| POST | `/packages/{id}/steps/{stepId}/retry` | A, C(P) | Chỉ bước LOI; reset attempts, enqueue |
| GET | `/packages/{id}/bidders/{bid}/evaluations` | A, L, C(P) | Theo `form`; gồm citations, tool_calls |
| PUT | `/evaluations/{id}/expert` | C(P) | `{expert_result, expert_comment}`; chỉ EXPERT_REVIEW |
| POST | `/packages/{id}/submit` | C(P) | EXPERT_REVIEW → LEADER_REVIEW |
| POST | `/packages/{id}/approve` | L | LEADER_REVIEW → COMPLETED |
| POST | `/packages/{id}/return` | L | `{reason}` → EXPERT_REVIEW |
| GET | `/packages/{id}/summary` | A, L, C(P) | Bảng so sánh nhà thầu |
| POST | `/packages/{id}/exports` | A, L, C(P) | Enqueue job xuất; trả `export_id` |
| GET | `/exports/{id}` | như gói | Trạng thái / tải zip |
| GET/POST | `/criteria-sets` | GET: A, C; POST: A | Bộ mẫu |
| GET/PUT | `/criteria-sets/{id}` | A (mẫu); C(P) (bộ của gói, DRAFT) | Cây tiêu chí |
| GET | `/kb/documents` | A, C, L | Danh sách |
| POST | `/kb/documents` | A | multipart + `doc_key,title,kb_type,effective_date`; cùng `doc_key` → version+1, bản cũ SUPERSEDED |
| GET | `/kb/documents/{id}` | A, C, L | Metadata + chunks + versions |
| GET | `/health` | công khai | `{db, redis}` |

## Payload mẫu
`GET /packages/{id}/bidders/{bid}/evaluations?form=MAU_02`
```json
{ "items": [{
  "id": "4c1e…", "criterion": {"code": "3.2", "title": "Doanh thu bình quân hằng năm", "requirement": "Doanh thu bình quân 3 năm gần nhất ≥ 8.000.000.000 VND", "kind": "QUANTITATIVE"},
  "ai_result": "DAT", "ai_reason": "Doanh thu bình quân 723.967.659.426 ≥ 8.000.000.000",
  "uncertain_reasons": [],
  "extracted_fields": {"doanh_thu_2023": "…", "doanh_thu_2024": "…", "doanh_thu_2025": "…"},
  "calculation_steps": [{"step": 1, "description": "Bình quân 3 năm", "expression": "round_vnd(avg([d23, d24, d25]))", "result": "723967659426"},
                        {"step": 2, "description": "So ngưỡng", "expression": "bq >= 8000000000", "result": "True"}],
  "replay_ok": true,
  "citations": [{"file_name": "HSDT_NangLucTaiChinh.pdf", "page_number": 12, "exact_quote": "Doanh thu năm 2025: …", "quote_verified": true, "document_id": "…"}],
  "expert_result": null, "expert_comment": null
}]}
```
