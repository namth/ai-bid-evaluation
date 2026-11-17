# 01 — Tổng quan

## Mục tiêu
Hệ thống nội bộ hỗ trợ tổ chuyên gia **đánh giá E-HSDT** của các nhà thầu trong một gói thầu. Hệ thống tự đọc hồ sơ dài 100–500+ trang, trích xuất dữ liệu kèm trích dẫn (file, trang, đoạn nguyên văn), đề xuất kết quả **Đạt / Không đạt / Cần làm rõ** theo từng tiêu chí, cho chuyên viên và lãnh đạo duyệt, rồi xuất biểu mẫu Excel chuẩn (DEC-001, DEC-018, DEC-020).

## Nỗi đau cốt lõi
- Chuyên viên phải đọc tay hàng trăm trang mỗi nhà thầu để điền Mẫu 01/02/02B; dễ sót và tốn nhiều ngày.
- Số liệu tài chính (doanh thu bình quân, tài sản ròng) phải tự tính, dễ nhầm.
- Khó truy lại "số này lấy ở trang nào" khi lãnh đạo hỏi lại.

## Đối tượng sử dụng
| Vai trò | Ai | Dùng để |
|---|---|---|
| Admin | IT / tổ thư ký | Quản trị người dùng, Kho tri thức, bộ tiêu chí mẫu |
| Chuyên viên | Tổ chuyên gia đánh giá | Tạo gói thầu, upload hồ sơ, duyệt từng tiêu chí |
| Lãnh đạo | Ban Giám đốc | Xem tổng hợp so sánh, duyệt cuối hoặc trả về |

## Phạm vi giai đoạn 1 (in scope)
- Gói thầu với 1 E-HSMT, 1 bộ tiêu chí, N nhà thầu (DEC-017); quy mô nhỏ 3–5 nhà thầu/gói (DEC-022).
- E-HSDT dạng PDF (gồm trang scan), `.docx`, `.xlsx` (chuyển sang PDF, DEC-029).
- Kho tri thức luật/quy định nội bộ, RAG luôn dùng bản mới nhất (DEC-041).
- Đánh giá theo 3 biểu mẫu: **Mẫu 01** tính hợp lệ, **Mẫu 02** năng lực & kinh nghiệm, **Mẫu 02B** năng lực kỹ thuật/nhân sự; thêm **báo cáo tổng hợp** so sánh nhà thầu.
- Duyệt 2 cấp, nhật ký kiểm toán, xuất Excel.
- LLM qua OpenRouter (DEC-003).

## Ngoài phạm vi giai đoạn 1
- Chấm điểm số, xếp hạng theo điểm (DEC-020).
- AI tự đề xuất bộ tiêu chí từ E-HSMT (DEC-031).
- SSO, đăng ký tài khoản tự do (DEC-027).
- Đánh giá về giá chi tiết (sửa lỗi số học, hiệu chỉnh sai lệch) — chỉ ghi nhận giá dự thầu.
- MinIO/S3 (DEC-028).

## Phase sau
- GĐ2: chuyển Ollama on-premise bằng `.env` (DEC-003); bật ZDR/che dữ liệu nếu cần (DEC-033).
- Scale-up worker/máy (DEC-022); AI đề xuất tiêu chí; đánh giá giá.
