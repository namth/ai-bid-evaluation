# dauthau — Hệ thống đánh giá E-HSDT nội bộ bằng AI

Đánh giá hồ sơ dự thầu theo Mẫu 01/02/02B, có trích dẫn trang và duyệt 2 cấp.

@CONTEXT.md

## Giai đoạn hiện tại
**Đặc tả xong, chưa có code.** Bước kế tiếp: prototype HTML (TASK-007), rồi M0. Khi code, đi theo `docs/09-conventions.md` và thứ tự `docs/11-tasks.md`.

## Bản đồ thư mục
| Đường dẫn | Vai trò | Được sửa? |
|---|---|---|
| `docs/` | Tài liệu chuẩn duy nhất (DEC-015) | Có — cập nhật khi lệch |
| `docs/14-decisions.md` | Sổ quyết định, thắng mọi mâu thuẫn | Chỉ thêm DEC mới / đánh dấu thay thế |
| `backend/` | FastAPI, worker ARQ, Alembic, tests | Có |
| `frontend/` | Next.js | Có |
| `deploy/` | compose, Caddyfile, `.env.example` | Có |
| `prototype/` | HTML tĩnh SCR-NN | Có |
| `samples/`, `File danh gia mau.xlsx` | Dữ liệu mẫu thật (CCCD, SĐT) | **Chỉ đọc, không commit** |
| `data/` | File upload/xuất lúc chạy | Không |

## Quy ước
- Giao diện và thông báo lỗi bằng tiếng Việt có dấu; mã lỗi `UPPER_SNAKE`.
- Prototype dùng Tailwind CDN và màu kết quả theo docs/03 (Đạt xanh, Không đạt đỏ, Cần làm rõ vàng).
- Accessibility: nút icon có `aria-label`, bảng tiêu chí điều hướng được bằng bàn phím.

## Quy tắc khi code thật
- Đặt tên theo docs/09; API `/api/v1` khớp đúng docs/07; schema khớp docs/06.
- **Không để LLM tự tính** — mọi số học qua `services/calc_tool.py`, phát lại theo BR-06.
- **Mọi trích xuất phải có citation đã xác minh** (BR-03, BR-05).
- **Tiền là `int` đồng**, dùng `Decimal` + `round_vnd`; cấm `float`.
- **Chỉ `core/llm_client.py` import `openai`**; cấu hình model qua env.
- **Không nạp cả PDF vào RAM**; xử lý theo cụm `PDF_BATCH_PAGES`.
- **Chuyển trạng thái gói chỉ qua `services/state_machine.py`**; ghi audit trong cùng transaction; không UPDATE/DELETE `audit_logs`.
- Kiểm quyền và phạm vi gói ở backend (`core/deps.py`).
- Test LLM bằng fake client; không gọi mạng trong `make test`.
- Conventional Commits kèm `[TASK-NNN]`; nhánh `feat/TASK-NNN-...`.

## Lệnh hữu ích (khi đã có code)
```bash
make lint            # ruff + mypy + eslint + tsc
make test            # pytest unit + integration (cần compose test)
make e2e             # Playwright
make eval            # đo độ chính xác model thật (thủ công)
docker compose -f deploy/docker-compose.test.yml up -d --wait
```

## Bảo mật
Không đọc/commit: `.env*` (trừ `.env.example`), `data/`, `samples/` và `File danh gia mau.xlsx`, `*.dump`. Không log nội dung hồ sơ hay prompt.

## Mô hình phân vai Claude ↔ Antigravity

- **Claude Code = Trưởng phòng / Giám đốc sản phẩm:** phân tích, thiết kế kiến trúc, viết đặc tả, viết test, giao việc, nghiệm thu, giữ `docs/`.
- **Antigravity (Gemini) = Kỹ sư thực thi:** viết code sản phẩm theo `brief.md`. Luật ở `.agents/rules/`, quy trình là skill `.agents/skills/thuc-hien-task/`, hook khoá test ở `.agents/hooks.json`. Claude gọi nó qua Antigravity CLI `agy` (chạy ngầm), dự phòng `antigravity-ide chat` hoặc clipboard.
- Giao thức & vòng đời task: `handoff/README.md`, công cụ `scripts/handoff.py` (`doctor` để kiểm tra), cấu hình riêng dự án ở `handoff/config.json`.
- Hạng mục code sản phẩm → mặc định giao qua `/giao-viec`, nghiệm thu qua `/nghiem-thu`. Claude chỉ tự code khi người dùng yêu cầu rõ, hoặc với việc nhỏ (sửa docs, script nội bộ, test).
- **Tiết kiệm token Claude, tận dụng tối đa Antigravity:** Antigravity viết toàn bộ code sản phẩm (task khó kèm blueprint trong brief); Claude chỉ lập kế hoạch (`handoff/templates/plan.md`), viết brief + test, quyết định nghiệm thu. Chạy lệnh/đọc log/tìm file giao subagent `dieu-phoi` (Haiku); giao song song 2 task mỗi tài khoản; Claude chỉ tự viết code lõi khi Antigravity cầu cứu sau 2 lần sửa. Chi tiết: `handoff/README.md`.
- Test trong `tests.lock` thuộc về Claude; Antigravity không được sửa.
- **Điều phối bằng Antigravity (ORCH):** model mặc định `gemini-3.8-flash-high` (hạn mức lớn nhất; model khác chỉ khi thật cần). Agent điều phối dùng `scripts/orchestrate.py` (`digest`/`launch`/`unstick`/`gate`/`merge-low`/`run`): LOW → gate tự động → merge-low; HIGH/nhạy cảm → dừng ở `review_passed` cho Claude duyệt diff. Chi tiết `handoff/README.md`.
- **Luôn theo dõi hiệu quả:** `scripts/metrics.py begin/end/report` (token Claude theo đợt, so sánh mode agy vs self, thống kê unstick/gate_fail/claude_fix…). Đầu mỗi đợt giao việc chạy `begin`, cuối đợt `end`, định kỳ `report` để điều chỉnh mô hình.
- **Task HIGH của dự án này** (Claude duyệt diff cuối): mọi task chạm bất biến INV-1…INV-8 — TASK-003, 004, 005, 008, 011, 013, 014, 019, 020.

## Cách làm việc
- Trả lời bằng tiếng Việt có dấu.
- Trước khi sửa một module: đọc doc liên quan + `docs/14-decisions.md`.
- Code lệch docs → sửa docs cùng lúc hoặc ghi vào `docs/11` mục Tồn đọng; quyết định mới → thêm DEC.
- Không tạo file .md lẻ ngoài bộ chuẩn (DEC-015).
