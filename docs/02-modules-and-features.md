# 02 — Phân hệ, tính năng & RBAC

## Phân hệ
| Mã | Phân hệ | Tính năng chính | Màn hình |
|---|---|---|---|
| MOD-AUTH | Xác thực & người dùng | Đăng nhập, refresh, đăng xuất, Admin tạo/khóa tài khoản, đổi vai trò | SCR-01, SCR-11 |
| MOD-PKG | Gói thầu | Tạo/sửa gói (mã, tên, ngày đóng thầu), phân công chuyên viên, nhân bản bộ tiêu chí, máy trạng thái | SCR-02, SCR-03, SCR-04 |
| MOD-DOC | Hồ sơ & ingest | Upload E-HSMT/E-HSDT, convert PDF, đọc cuốn chiếu, OCR Vision, embedding, theo dõi bước + chạy lại | SCR-04 |
| MOD-KB | Kho tri thức | Upload văn bản luật/quy định, metadata, chunk theo Điều/Khoản, embedding, thay bản mới | SCR-07, SCR-08 |
| MOD-CRIT | Bộ tiêu chí | Bộ mẫu (Admin), bộ của gói (Chuyên viên), tiêu chí ngôn ngữ tự nhiên theo Mẫu 01/02/02B | SCR-09, SCR-10 |
| MOD-EVAL | Đánh giá AI | Agent truy xuất + trích xuất + tool tính + phát lại log + quy tắc Cần làm rõ | SCR-04, SCR-05 |
| MOD-REVIEW | Duyệt | KQ chuyên gia, nhận xét, gửi lãnh đạo, duyệt cuối/trả về | SCR-05, SCR-06 |
| MOD-REPORT | Báo cáo | Excel Mẫu 01/02/02B mỗi nhà thầu + tổng hợp, gói zip | SCR-06 |
| MOD-AUDIT | Kiểm toán | Nhật ký chỉ INSERT, xem/lọc | SCR-11 |

## Vai trò và phạm vi dữ liệu
- **Vai trò** (DEC-016): `ADMIN`, `CHUYEN_VIEN`, `LANH_DAO`.
- **Phạm vi dữ liệu** (DEC-017): `CHUYEN_VIEN` chỉ thấy/sửa gói được phân công (`package_assignments`); `ADMIN`, `LANH_DAO` thấy mọi gói. Truy cập gói ngoài phạm vi → `404 PACKAGE_NOT_FOUND` (không lộ sự tồn tại).

## Ma trận RBAC
✓ = được; P = chỉ gói được phân công; — = không.

| Hành động | ADMIN | CHUYEN_VIEN | LANH_DAO |
|---|---|---|---|
| Quản lý người dùng | ✓ | — | — |
| Xem nhật ký kiểm toán | ✓ | — | — |
| Quản lý Kho tri thức (upload/thay bản) | ✓ | — | — |
| Xem Kho tri thức | ✓ | ✓ | ✓ |
| Tạo/sửa bộ tiêu chí **mẫu** | ✓ | — | — |
| Tạo gói thầu | ✓ | ✓ (tự động được phân công) | — |
| Phân công chuyên viên vào gói | ✓ | — | — |
| Sửa gói / bộ tiêu chí của gói (khi Nháp) | ✓ | P | — |
| Upload E-HSMT/E-HSDT, thêm nhà thầu (khi Nháp) | ✓ | P | — |
| Bắt đầu xử lý AI, chạy lại bước | ✓ | P | — |
| Xem gói, kết quả, PDF | ✓ | P | ✓ |
| Ghi KQ chuyên gia + nhận xét | — | P | — |
| Gửi lãnh đạo duyệt | — | P | — |
| Duyệt cuối / trả về | — | — | ✓ |
| Xuất Excel (bản nháp khi chưa Hoàn tất) | ✓ | P | ✓ |

Admin không ghi KQ chuyên gia để tách bạch trách nhiệm chuyên môn. Ánh xạ chi tiết sang endpoint: `docs/07-api-contracts.md`.
