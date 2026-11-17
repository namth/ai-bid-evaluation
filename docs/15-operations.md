# 15 — Vận hành

## Môi trường
| Môi trường | Nơi chạy | Compose | Ghi chú |
|---|---|---|---|
| dev | máy lập trình | `deploy/docker-compose.yml` + `docker-compose.override.yml` (hot reload) | LLM OpenRouter hoặc fake |
| test | máy lập trình / CI | `deploy/docker-compose.test.yml` | Postgres `localhost:55432`, Redis `localhost:56379`, LLM fake |
| production | VPS 4GB (DEC-004) | `deploy/docker-compose.yml` | Caddy HTTPS (DEC-032) |

## Triển khai lần đầu
1. VPS Ubuntu LTS, cài Docker + compose plugin; firewall chỉ mở 22, 80, 443; bật swap 2GB (đệm khi LibreOffice convert).
2. Trỏ DNS `DOMAIN` về VPS.
3. `cp deploy/.env.example deploy/.env`, điền giá trị thật (không commit).
4. `docker compose -f deploy/docker-compose.yml up -d --build`.
5. `docker compose exec api alembic upgrade head`.
6. `docker compose exec api python -m app.scripts.bootstrap_admin` (đọc `BOOTSTRAP_ADMIN_*`, chạy một lần, sau đó xóa biến khỏi `.env`).
7. Kiểm tra `https://<DOMAIN>/api/v1/health`.

## Cập nhật phiên bản
`git pull` → `docker compose build` → `docker compose run --rm api alembic upgrade head` → `docker compose up -d`. Migration phá vỡ tương thích phải ghi ở Nhật ký phát hành kèm bước tay.

## Migration chạy tay
_(chưa có)_

## Cấu hình sau deploy
- Admin upload văn bản Kho tri thức (Luật Đấu thầu, Nghị định hướng dẫn, quy định nội bộ).
- Admin tạo bộ tiêu chí mẫu (hoặc dùng seed "Hàng hóa CNTT").
- Kiểm tra `EMBEDDING_DIM` khớp model (worker tự dừng nếu sai — TC-25).

## Sao lưu / Khôi phục
- Hằng ngày 02:00: `pg_dump -Fc` → `/backup/db/dauthau-YYYYMMDD.dump`; `rsync data/` → `/backup/data/`; giữ 7 bản ngày + 4 bản tuần. Khuyến nghị đồng bộ thêm ra ngoài VPS.
- Khôi phục: dừng api/worker → `pg_restore --clean -d dauthau <file>` → khôi phục `data/` → khởi động.
- Theo dõi đĩa: API trả 507 khi trống < `DISK_MIN_FREE_GB`; dọn `EXPORT_DIR` file > 30 ngày.

## Rollback
Checkout tag trước → build → nếu migration mới có downgrade an toàn thì `alembic downgrade -1`, nếu không thì khôi phục bản sao lưu trước khi deploy.

## Chuyển GĐ2 (Ollama)
Xem docs/08 mục "Chuyển GĐ2". Không cần embed lại KB nếu vẫn bge-m3 (DEC-026).

## Nhật ký phát hành
| Ngày | Phiên bản | Nội dung | Bước tay |
|---|---|---|---|
| — | — | — | — |

## Sự cố thường gặp
| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Worker bị OOM kill | LibreOffice convert file lớn | Kiểm swap; giảm `PDF_BATCH_PAGES`; giữ `WORKER_MAX_JOBS=1` |
| Nhiều bước LOI với 429 | Hạn mức OpenRouter | Chờ, bấm Chạy lại bước; cân nhắc model/nhà cung cấp khác qua `.env` |
| Bước RUNNING mãi | Worker chết giữa chừng | Khởi động lại worker → bước chuyển LOI (BR-08) → Chạy lại |
| Nhiều tiêu chí `QUOTE_NOT_FOUND` | Trang scan OCR kém | Tăng `OCR_DPI` lên 200, chạy lại bước OCR |
