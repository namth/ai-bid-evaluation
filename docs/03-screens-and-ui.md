# 03 — Màn hình & UI

Thiết kế bằng prototype HTML tĩnh trong `prototype/` trước khi code Next.js (DEC-023). UI tiếng Việt, Shadcn/UI, desktop-first (≥ 1280px; SCR-05 cần ≥ 1440px để chia đôi thoải mái).

## Kiểm kê màn hình
| ID | Tên | Route | Vai trò | Design | Code | DoD |
|---|---|---|---|---|---|---|
| SCR-01 | Đăng nhập | `/login` | Tất cả | Chưa | Chưa | Đăng nhập đúng → `/packages`; sai → lỗi chung "Email hoặc mật khẩu không đúng"; bị khóa 429 hiển thị thời gian chờ |
| SCR-02 | Danh sách gói thầu | `/packages` | Tất cả (CV: P) | Chưa | Chưa | Lọc theo trạng thái, tìm theo mã/tên, phân trang 20; badge trạng thái |
| SCR-03 | Tạo/sửa gói thầu | `/packages/new`, `/packages/[id]/edit` | ADMIN, CV | Chưa | Chưa | Form mã, tên, ngày đóng thầu, chọn bộ tiêu chí mẫu để nhân bản, upload E-HSMT; chỉ sửa khi Nháp |
| SCR-04 | Chi tiết gói thầu | `/packages/[id]` | Tất cả (CV: P) | Chưa | Chưa | Danh sách nhà thầu + upload E-HSDT; bảng tiến độ bước pipeline (polling 5s) + nút Chạy lại bước LOI; nút Bắt đầu xử lý AI / Gửi lãnh đạo |
| SCR-05 | Duyệt chia đôi | `/packages/[id]/bidders/[bidderId]/review` | Tất cả (ghi: CV P) | Chưa | Chưa | Trái: PDF viewer; phải: tab Mẫu 01/02/02B; click tiêu chí/citation → viewer nhảy đúng file + trang và tô sáng `exact_quote`; hiển thị `calculation_steps` + kết quả phát lại; chọn KQ chuyên gia + nhận xét, lưu tự động |
| SCR-06 | Tổng hợp & xuất | `/packages/[id]/summary` | Tất cả (CV: P) | Chưa | Chưa | Bảng so sánh nhà thầu (giá dự thầu, KẾT LUẬN từng mẫu, số tiêu chí Không đạt/Cần làm rõ); Lãnh đạo: Duyệt / Trả về (lý do bắt buộc); nút Xuất Excel (zip) |
| SCR-07 | Kho tri thức | `/kb` | Tất cả (ghi: ADMIN) | Chưa | Chưa | Danh sách văn bản (tên, loại, phiên bản, ngày hiệu lực, trạng thái); upload; thay bản mới |
| SCR-08 | Chi tiết văn bản KB | `/kb/[id]` | Tất cả | Chưa | Chưa | Metadata, danh sách chunk theo Điều/Khoản, trạng thái embedding, các phiên bản cũ |
| SCR-09 | Danh sách bộ tiêu chí | `/criteria-sets` | ADMIN (xem: CV) | Chưa | Chưa | Bộ mẫu; nhân bản bộ mẫu |
| SCR-10 | Soạn bộ tiêu chí | `/criteria-sets/[id]` | ADMIN (bộ mẫu), CV P (bộ của gói khi Nháp) | Chưa | Chưa | Cây tiêu chí theo 3 mẫu: mã STT, tiêu đề, yêu cầu ngôn ngữ tự nhiên, loại, bắt buộc; kéo thả thứ tự |
| SCR-11 | Quản trị | `/admin/users`, `/admin/audit` | ADMIN | Chưa | Chưa | Tạo/khóa người dùng, đổi vai trò, phân công gói; nhật ký lọc theo người/thực thể/ngày |

## Điều hướng
```
/login → /packages ─┬─ /packages/new
                    └─ /packages/[id] ─┬─ /edit
                                       ├─ /bidders/[bidderId]/review   (SCR-05)
                                       └─ /summary                      (SCR-06)
Thanh bên: Gói thầu · Kho tri thức · Bộ tiêu chí · Quản trị (chỉ ADMIN)
```

## Bốn trạng thái UI (bắt buộc cho mọi màn danh sách/chi tiết)
| Trạng thái | Quy ước |
|---|---|
| Empty | Minh họa + câu hướng dẫn + nút hành động chính (vd "Chưa có gói thầu — Tạo gói thầu") |
| Skeleton loading | Shadcn `Skeleton` đúng bố cục; không spinner toàn trang |
| Success | Dữ liệu; toast xanh sau thao tác ghi |
| Validation error | Lỗi dưới từng trường (zod, khớp `error.details` từ API); lỗi hệ thống = toast đỏ kèm mã lỗi |

## Màu kết quả (đồng bộ với Excel, docs/12 BR-14)
Đạt = xanh lá; Không đạt = đỏ; Cần làm rõ = vàng/cam; Chưa đánh giá = xám.

## Quy tắc validate form
| Trường | Quy tắc |
|---|---|
| Email | định dạng email, ≤ 255 ký tự, duy nhất |
| Mật khẩu | ≥ 12 ký tự |
| Mã gói thầu | bắt buộc, ≤ 50 ký tự, `[A-Za-z0-9._/-]`, duy nhất |
| Tên gói thầu | bắt buộc, ≤ 500 ký tự |
| Ngày đóng thầu | bắt buộc, ngày hợp lệ |
| Tên nhà thầu | bắt buộc, ≤ 500 ký tự; MST tùy chọn 10 hoặc 13 số (duy nhất trong gói nếu có) |
| Giá dự thầu | tùy chọn, số nguyên đồng ≥ 0 (nhập dạng `1.234.567`) |
| File upload | `.pdf`, `.docx`, `.xlsx`; ≤ 200 MB/file |
| Tiêu chí | mã STT bắt buộc, yêu cầu ≥ 10 ký tự |
| Nhận xét chuyên gia | bắt buộc khi KQ chuyên gia khác KQ AI (DEC-040) |
| Lý do trả về | bắt buộc, ≥ 10 ký tự |
