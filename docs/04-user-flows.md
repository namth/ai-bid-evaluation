# 04 — Luồng người dùng

## UF-01 Chuẩn bị (Admin)
1. Admin tạo tài khoản Chuyên viên/Lãnh đạo (SCR-11).
2. Admin upload văn bản luật/quy định vào Kho tri thức (SCR-07) → hệ thống chunk theo Điều/Khoản + embedding (BR-12).
3. Admin tạo bộ tiêu chí mẫu, vd "Hàng hóa CNTT", gồm tiêu chí theo Mẫu 01/02/02B viết bằng ngôn ngữ tự nhiên (SCR-10).

## UF-02 Tạo gói thầu (Chuyên viên)
1. SCR-03: nhập mã, tên, ngày đóng thầu; chọn bộ tiêu chí mẫu → hệ thống nhân bản thành bộ riêng của gói (DEC-031); upload E-HSMT (tùy chọn, để tham chiếu).
2. Chỉnh ngưỡng trong bộ tiêu chí của gói theo E-HSMT (SCR-10).
3. SCR-04: thêm nhà thầu (tên, MST, giá dự thầu), upload các file E-HSDT của từng nhà thầu.
4. Bấm **Bắt đầu xử lý AI** → gói chuyển `PROCESSING` (BR-01).

## UF-03 Xử lý AI (hệ thống, chạy nền ARQ)
Với mỗi file E-HSDT, các bước idempotent (BR-08):
1. `CONVERT`: `.docx/.xlsx` → PDF bằng LibreOffice; PDF giữ nguyên.
2. `READ_BATCH` (mỗi 8 trang): PyMuPDF lấy text; pdfplumber lấy bảng; trang không có text layer → đánh dấu scan.
3. `OCR_PAGE`: trang scan → ảnh 150 DPI → Vision API → text.
4. `EMBED_BATCH`: chunk theo trang → embedding bge-m3 → `chunks`.
Sau khi mọi file của nhà thầu xong:
5. `EVALUATE` (mỗi tiêu chí × nhà thầu): truy xuất trang liên quan → Agent trích xuất + gọi tool tính → kiểm citation + phát lại log → KQ AI (BR-03…BR-07).
Khi toàn bộ bước của gói `DONE` hoặc `LOI` → gói chuyển `EXPERT_REVIEW`.

## UF-04 Duyệt của Chuyên viên (SCR-05)
1. Mở từng nhà thầu; bảng phải liệt kê tiêu chí theo Mẫu 01/02/02B với KQ AI, dữ liệu trích xuất, citation.
2. Click tiêu chí/citation → PDF bên trái nhảy đúng trang, tô sáng đoạn trích.
3. Với tiêu chí định lượng: xem `calculation_steps` + trạng thái phát lại.
4. Chọn KQ chuyên gia (Đạt/Không đạt) + nhận xét (bắt buộc nếu khác KQ AI). Lưu tự động, ghi audit.
5. Tiêu chí `LOI`/Cần làm rõ do lỗi kỹ thuật → bấm Chạy lại bước ở SCR-04.
6. Khi mọi tiêu chí của mọi nhà thầu có KQ chuyên gia → **Gửi lãnh đạo duyệt** → `LEADER_REVIEW` (BR-02).

## UF-05 Duyệt cuối (Lãnh đạo, SCR-06)
1. Xem bảng tổng hợp so sánh nhà thầu, có thể mở SCR-05 ở chế độ chỉ đọc.
2. **Duyệt** → `COMPLETED`, khóa toàn bộ kết quả (BR-02); hoặc **Trả về** kèm lý do → `EXPERT_REVIEW`.

## UF-06 Xuất báo cáo
Bấm Xuất Excel (SCR-06) → job ARQ sinh zip gồm `Mau01_<NT>.xlsx`, `Mau02_<NT>.xlsx`, `Mau02B_<NT>.xlsx` cho từng nhà thầu + `TongHop.xlsx` (BR-13). Trước `COMPLETED` file có dòng tiêu đề "BẢN NHÁP".
