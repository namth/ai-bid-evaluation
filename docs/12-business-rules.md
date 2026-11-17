# 12 — Quy tắc nghiệp vụ

## BR-01 Máy trạng thái gói thầu (DEC-021)
| Từ | Đến | Ai | Guard | Side-effect |
|---|---|---|---|---|
| DRAFT | PROCESSING | A, C(P) | ≥ 1 nhà thầu; mỗi nhà thầu ≥ 1 file EHSDT; bộ tiêu chí có ≥ 1 tiêu chí không phải GROUP | Khóa bộ tiêu chí (`locked_at`); tạo `pipeline_steps` CONVERT; enqueue; audit |
| PROCESSING | EXPERT_REVIEW | hệ thống | Mọi bước của gói ở DONE hoặc LOI | audit (actor null) |
| EXPERT_REVIEW | LEADER_REVIEW | C(P) | Mọi `evaluations` của gói có `expert_result` (BR-16) | audit |
| LEADER_REVIEW | COMPLETED | L | — | `completed_at`; khóa dữ liệu (BR-02); audit |
| LEADER_REVIEW | EXPERT_REVIEW | L | `reason` ≥ 10 ký tự | lưu `return_reason`; audit |
Mọi chuyển khác → `409 INVALID_STATE`. Ở EXPERT_REVIEW vẫn cho Chạy lại bước LOI (gói giữ EXPERT_REVIEW; evaluation chạy lại thì xóa `expert_result` của tiêu chí đó).

## BR-02 Khóa khi Hoàn tất
`COMPLETED` ⇒ không sửa gói, nhà thầu, tài liệu, tiêu chí, evaluations (API 409 + trigger DB). Chỉ xem và xuất Excel chính thức.

## BR-03 Citation bắt buộc (DEC-009)
Mỗi `extracted_fields` có giá trị và mỗi số đầu vào tool phải có ≥ 1 citation (`file_name`, `page_number`, `exact_quote` ≥ 5 ký tự). Thiếu → BR-04.

## BR-04 Quy tắc Cần làm rõ (DEC-035, DEC-038)
`ai_result = CAN_LAM_RO` nếu có **bất kỳ** lý do (ghi vào `uncertain_reasons`):
| Mã lý do | Điều kiện |
|---|---|
| `MISSING_DATA` | Trường bắt buộc của tiêu chí rỗng |
| `MISSING_CITATION` | Vi phạm BR-03 |
| `QUOTE_NOT_FOUND` | Citation không qua BR-05 |
| `NUMBER_UNPARSEABLE` | Số không parse được theo BR-10 |
| `LLM_UNCERTAIN` | LLM trả `UNCERTAIN` |
| `REPLAY_MISMATCH` | BR-06 thất bại |
| `SCHEMA_INVALID` | Output không hợp schema sau 2 lần hỏi lại |
| `STEP_FAILED` | Bước EVALUATE hoặc bước phụ thuộc ở LOI |
Không có lý do nào → `ai_result` = kết luận của Agent (DAT/KHONG_DAT).

## BR-05 Xác minh trích dẫn
Chuẩn hóa cả `exact_quote` và text trang: Unicode NFC, gộp mọi khoảng trắng thành 1 space, bỏ khoảng trắng đầu/cuối, chữ thường. Khớp khi quote chuẩn hóa là chuỗi con của text trang `page_number` (hoặc trang kế tiếp nếu đoạn vắt trang). `quote_verified` lưu kết quả.

## BR-06 Phát lại tool (DEC-038)
Với mỗi evaluation có `kind=QUANTITATIVE`:
1. Chạy lại từng `tool_calls.expression` với `inputs` bằng `calc_tool` → `replay_result`; khác `result` → mismatch.
2. Mọi số xuất hiện trong `calculation_steps[].result` phải bằng một `tool_calls.result`.
3. Bước cuối là phép so sánh trả boolean; `True` ⇔ Agent kết luận DAT.
4. Mỗi biến trong `inputs` khớp một `extracted_fields` có citation.
5. Tiêu chí QUANTITATIVE không có tool call nào → mismatch.
`replay_ok = (không mismatch)`.

## BR-07 Tool tính (DEC-037)
Theo docs/05 mục Tool tính toán an toàn. Biểu thức không hợp lệ → tool trả lỗi cho Agent (Agent được thử lại trong giới hạn `AGENT_MAX_STEPS`), không ném exception ra ngoài.

## BR-08 Bước pipeline (DEC-039)
- Khóa duy nhất `(step_type, step_key)`; chạy lại bước DONE là no-op.
- Lỗi tạm thời (timeout, 429, 5xx, lỗi mạng): retry 3 lần backoff 2s/8s/32s; hết → LOI + `last_error`.
- Lỗi tất định (file hỏng, PDF có mật khẩu, LibreOffice lỗi, Vision trả rỗng): LOI ngay.
- Bước RUNNING quá 30 phút (worker chết) → coi là LOI khi worker khởi động lại.
- Retry thủ công: chỉ bước LOI; đặt PENDING, `attempts=0`.

## BR-09 Bộ nhớ (DEC-005)
Không đọc toàn bộ PDF vào RAM: không `doc.tobytes()`, không list text toàn bộ trang; mỗi cụm `PDF_BATCH_PAGES` trang. Đỉnh RSS worker khi ingest PDF 500 trang ≤ 600 MB (TC-09).

## BR-10 Số kiểu Việt Nam (DEC-036)
- `1.234.567` → 1234567; `723.967.659.425,6666` → Decimal `723967659425.6666`; `1,234,567` (kiểu Anh, ≥ 2 dấu phẩy) → 1234567; bỏ `VND`, `đồng`, `đ`, khoảng trắng.
- Có cả `.` và `,`: dấu xuất hiện **sau cùng** là thập phân.
- Một dấu `.` duy nhất với đúng 3 chữ số sau → nghìn; ngược lại → không parse được (`NUMBER_UNPARSEABLE`), không đoán.
- Chữ "tỷ", "triệu" → nhân 10^9, 10^6.
- Tiền: `round_vnd` = Decimal quantize(1, ROUND_HALF_UP) sau **mỗi** phép tính tiền.

## BR-11 KẾT LUẬN từng mẫu
- KẾT LUẬN tự động = `KHÔNG ĐẠT` nếu có tiêu chí `mandatory` với `ai_result=KHONG_DAT`; ngược lại `CẦN LÀM RÕ` nếu có `CAN_LAM_RO`; ngược lại `ĐẠT`.
- KẾT LUẬN chuyên gia = `KHÔNG ĐẠT` nếu có tiêu chí `mandatory` với `expert_result=KHONG_DAT`; ngược lại `ĐẠT`.
- Tiêu chí `GROUP` (vd 2.1) không đánh giá trực tiếp; KQ hiển thị = tổng hợp con theo cùng quy tắc.
- Tính bằng code (backend) và bằng công thức Excel (BR-13) — hai kết quả phải khớp (TC-21).

## BR-12 Kho tri thức (DEC-013, DEC-041)
- Chunk theo ranh giới "Điều N." / "Khoản"/ đánh số `1.` `a)`; chunk > 1500 ký tự cắt theo câu, chồng 150 ký tự; `meta = {article, clause}`.
- Upload cùng `doc_key` → `version+1`, bản mới ACTIVE, bản cũ SUPERSEDED (cùng transaction).
- RAG chỉ truy xuất chunk của bản ACTIVE.

## BR-13 Xuất Excel (DEC-014, DEC-018)
- Bố cục theo `File danh gia mau.xlsx`: tiêu đề, dòng "Số / Hà Nội, ngày…", header 2 tầng, ô gộp, đánh dấu `x`/`-` ở cột Đạt/Không đạt cho cả "KQ tự động" và "KQ chuyên gia", cột nhận xét, dòng KẾT LUẬN.
- Cột "KQ tự động": `x` vào Đạt hoặc Không đạt; CAN_LAM_RO → cả hai `-` và ô tô vàng + ghi "Cần làm rõ" ở nhận xét hệ thống (thêm cột "Lý do/Trích dẫn hệ thống" ngay trước cột nhận xét chuyên gia, gồm lý do AI + `file — tr. N`).
- KẾT LUẬN dùng **công thức Excel**, vd Mẫu 01: `=IF(COUNTIF(F5:F14,"x")>0,"KHÔNG ĐẠT","ĐẠT")` trên các dòng mandatory.
- Mẫu 02: bảng hợp đồng tương tự + giá trị (ô số định dạng `#,##0`), tổng giá trị bằng `=SUM()`.
- Mẫu 02B: mỗi vị trí → bảng thông tin nhân sự + bảng quá trình công tác.
- TongHop.xlsx: mỗi dòng một nhà thầu: tên, MST, giá dự thầu (số), KẾT LUẬN Mẫu 01/02/02B (tham chiếu giá trị), `=COUNTIF` số Không đạt/Cần làm rõ, cột "Hợp lệ & đáp ứng" `=IF(AND(...="ĐẠT"),"ĐẠT","KHÔNG ĐẠT")`, cột Xếp hạng giá `=IF(<đáp ứng>="ĐẠT",COUNTIFS(<đáp ứng>,"ĐẠT",<giá>,"<"&<giá dòng>)+1,"")` (giá thấp nhất hạng 1), dòng cuối `=MIN`/`=AVERAGE` giá các nhà thầu đáp ứng.
- Trước COMPLETED: tiêu đề có "(BẢN NHÁP)"; tên zip `<mã gói>_<yyyymmdd-hhmm>[_NHAP].zip`.

## BR-14 Màu
Đạt `#C6EFCE`/chữ `#006100`; Không đạt `#FFC7CE`/`#9C0006`; Cần làm rõ `#FFEB9C`/`#9C5700`; viền mảnh toàn bảng.

## BR-15 Bộ tiêu chí (DEC-031)
Tạo gói = nhân bản sâu bộ mẫu (`source_set_id`); sửa bộ mẫu sau đó không ảnh hưởng gói. Bộ của gói khóa khi rời DRAFT.

## BR-16 KQ chuyên gia (DEC-040)
Chỉ ở EXPERT_REVIEW, chỉ C(P). `expert_result ≠ ai_result` (kể cả khi `ai_result = CAN_LAM_RO`) ⇒ `expert_comment` bắt buộc ≥ 10 ký tự (`422 COMMENT_REQUIRED`). Không ghi đè cột AI. Mỗi lần ghi → audit `EXPERT_RESULT_SET` (before/after).

## BR-17 Tổng hợp so sánh
Bảng SCR-06/TongHop: thứ tự theo xếp hạng giá (BR-13) — chỉ xếp hạng nhà thầu "đáp ứng" (cả 3 mẫu KẾT LUẬN chuyên gia ĐẠT); không có điểm số (DEC-020). So sánh giá bằng code `int`, không LLM.

## BR-18 Phạm vi dữ liệu (DEC-017)
Mọi truy vấn gói/nhà thầu/tài liệu/evaluation của CHUYEN_VIEN lọc theo `package_assignments`; ngoài phạm vi → 404.
