# 13 — Test case & QA

Mức: U = unit, I = integration (Postgres + Redis thật qua `deploy/docker-compose.test.yml`, LLM fake — DEC-044), E = E2E Playwright, M = thủ công/eval. Lệnh: `make test` (U+I), `make e2e`, `make eval`.

## Bất biến (ưu tiên cao nhất)
| TC | Bất biến | Mức | Kịch bản → Kỳ vọng |
|---|---|---|---|
| TC-01 | INV-1 tool tính an toàn | U | `__import__('os')`, `x.__class__`, `open()`, lambda, biểu thức > 500 ký tự → lỗi `CALC_REJECTED`; `avg([1,2])`, `a >= b` → OK |
| TC-02 | INV-3 tiền Decimal/làm tròn | U | `round_vnd(avg([2171902978277, …]))` đúng ROUND_HALF_UP; `0.5 → 1`, `2.5 → 3`; không trả float |
| TC-03 | BR-10 parser số VN | U | bảng ≥ 15 ca: `24.843.500.000`, `723.967.659.425,6666`, `8 tỷ`, `1,234,567`, `1.5` → UNPARSEABLE, `432.300.000 VND` |
| TC-04 | INV-3 không float cho tiền | I | Introspect schema: không cột `real/double precision`; grep `float(` trong `app/services` = 0 |
| TC-05 | INV-4 audit chỉ INSERT | I | `UPDATE/DELETE audit_logs` bằng role app → lỗi |
| TC-06 | INV-5 Hoàn tất bất biến | I | Gói COMPLETED: PUT expert / PATCH gói / thêm file → 409; UPDATE trực tiếp `evaluations` → trigger lỗi |
| TC-07 | BR-01 máy trạng thái | I | Mọi cặp chuyển hợp lệ OK + audit; mọi cặp không hợp lệ → 409; guard thiếu file → 409 |
| TC-08 | INV-8 phạm vi dữ liệu | I | CV không được phân công: GET/PUT gói, evaluation, `documents/{id}/pdf` → 404; LĐ GET OK, PUT expert → 403 |
| TC-09 | INV-6 cuốn chiếu | I | Ingest PDF sinh 500 trang (fixture) đo RSS đỉnh worker ≤ 600 MB; số lần commit = ceil(500/8) |
| TC-10 | INV-2 citation bắt buộc | U | Fake LLM trả field không citation → `CAN_LAM_RO` + `MISSING_CITATION` |
| TC-11 | INV-1 phát lại tool | U | 5 ca BR-06 (kết quả sai, số không từ tool, bool cuối ngược kết luận, không tool call, input không citation) → `REPLAY_MISMATCH`; ca đúng → `replay_ok=true`, DAT |
| TC-12 | INV-7 không hard-code provider | U | Không module nào ngoài `core/llm_client.py` import `openai`; đổi `LLM_BASE_URL` sang URL fake → request đi đúng URL, model đúng `LLM_MODEL` |

## Chức năng
| TC | Nội dung | Mức | Kỳ vọng |
|---|---|---|---|
| TC-13 | BR-05 xác minh quote | U | Khác khoảng trắng/NFD/hoa-thường → verified; sai một từ → không; vắt trang → verified |
| TC-14 | BR-04 `LLM_UNCERTAIN`, `SCHEMA_INVALID`, `MISSING_DATA`, `NUMBER_UNPARSEABLE` | U | Mỗi lý do → CAN_LAM_RO với đúng mã |
| TC-15 | Trang scan → OCR | I | PDF 3 trang (1 ảnh) → Vision fake gọi đúng 1 lần, `is_scanned=true` |
| TC-16 | BR-08 idempotent | I | Chạy job 2 lần → không nhân đôi pages/chunks; bước DONE bỏ qua |
| TC-17 | BR-08 retry | I | Fake LLM 429 ×2 rồi OK → DONE (attempts=3); 429 ×4 → LOI; retry thủ công → PENDING → DONE |
| TC-18 | Convert docx/xlsx | I | docx 3 trang → PDF ≥ 1 trang, pages có text; file hỏng → CONVERT LOI |
| TC-19 | BR-16 KQ chuyên gia | I | Khác AI không nhận xét → 422 `COMMENT_REQUIRED`; có → 200 + audit before/after |
| TC-20 | Gửi lãnh đạo thiếu KQ | I | → 409 `REVIEW_INCOMPLETE` |
| TC-21 | BR-11 KẾT LUẬN code = Excel | I | Sinh Excel, tính lại công thức bằng LibreOffice headless (`--convert-to xlsx` recalculation) → ô KẾT LUẬN = kết quả backend |
| TC-22 | BR-13 bố cục mẫu | U | Header, ô gộp, tiêu đề 3 sheet khớp `File danh gia mau.xlsx` (so bằng openpyxl trên bản ẩn danh `samples/mau-an-danh.xlsx`) |
| TC-23 | BR-13/17 TongHop | I | 4 nhà thầu (1 không đáp ứng) → xếp hạng 1–3 theo giá, nhà thầu không đáp ứng để trống; công thức tồn tại (ô bắt đầu bằng `=`) |
| TC-24 | BR-12 KB phiên bản | I | Upload cùng `doc_key` 2 lần → v1 SUPERSEDED, v2 ACTIVE; retrieval chỉ trả chunk v2 |
| TC-25 | Embedding sai chiều | U | Fake trả 768 chiều → worker dừng khởi động với lỗi rõ |
| TC-26 | Đăng nhập | I | 5 lần sai → 429 15 phút; refresh xoay vòng, refresh cũ dùng lại → 401 |
| TC-27 | Upload an toàn | I | `.exe` đổi đuôi `.pdf` → 415; 201 MB → 413; PDF mật khẩu → 415; tên file `../../x` lưu an toàn |
| TC-28 | Cookie & CSRF | I | Cookie có `HttpOnly; Secure; SameSite=Lax`; POST với `Origin` lạ → 403 |
| TC-29 | BR-15 nhân bản tiêu chí | I | Sửa bộ mẫu sau khi tạo gói → bộ gói không đổi; PUT bộ gói khi PROCESSING → 409 |
| TC-30 | Không log nội dung | I | Chạy evaluate với fake LLM có chuỗi CCCD → log stdout không chứa chuỗi đó |
| TC-31 | `mem_limit` | U | Parse `deploy/docker-compose.yml`: mọi service có `mem_limit`, tổng ≤ 3.1 GB |
| TC-32 | E2E SCR-05 | E | Click citation tr. 12 → viewer hiện trang 12, đoạn trích được tô sáng; chọn KQ chuyên gia → lưu |
| TC-33 | E2E luồng 2 cấp | E | CV gửi → LĐ trả về (lý do) → CV sửa, gửi → LĐ duyệt → nút sửa biến mất, xuất zip không có "_NHAP" |
| TC-34 | E2E 4 trạng thái UI | E | SCR-02/04/07: empty, skeleton (chặn mạng), success, lỗi validate |
| TC-35 | Độ chính xác model | M | `make eval` trên bộ hồ sơ mẫu gán nhãn: KQ AI khớp nhãn ≥ 85% tiêu chí Mẫu 01, ≥ 75% Mẫu 02/02B; 100% citation `quote_verified` của KQ DAT/KHONG_DAT |

## Tiêu chuẩn bàn giao QA
- `make test` xanh, coverage `app/services` ≥ 85%; mọi TC-01…TC-12 xanh.
- `make e2e` xanh cho milestone đang bàn giao.
- `make eval` chạy tay có báo cáo trong `docs/qa/reports/QA-REPORT-YYYYMMDD-HHMM.md`.
- Checklist thủ công theo milestone ở `docs/qa/checklist-<milestone>.md`.
