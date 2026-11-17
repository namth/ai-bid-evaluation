# 09 — Quy chuẩn kỹ thuật

## Cấu trúc thư mục (DEC-042)
```
backend/
  app/
    api/v1/          # router FastAPI, mỗi phân hệ một file: auth.py, packages.py, documents.py, evaluations.py, kb.py, criteria.py, exports.py, users.py
    core/            # config.py (pydantic-settings), security.py, llm_client.py, db.py, errors.py, deps.py (RBAC + phạm vi gói)
    models/          # SQLAlchemy 2.0 declarative, async
    schemas/         # Pydantic v2 (request/response + LLM output)
    services/        # ingest/, eval/ (retrieval, agent, verification), calc_tool.py, excel/, kb/, state_machine.py, audit.py
    workers/         # ARQ: settings.py, tasks.py
  alembic/
  tests/             # unit/, integration/, fakes/ (fake LLM)
  eval/              # bộ đánh giá model thật (make eval)
  pyproject.toml
frontend/
  app/ components/ lib/api/ (sinh từ OpenAPI) lib/ hooks/
deploy/  docker-compose.yml  docker-compose.test.yml  Caddyfile  .env.example
prototype/  samples/  docs/
```

## Đặt tên (DEC-043)
| Lớp | Quy ước |
|---|---|
| Python | module/hàm/biến `snake_case`, class `PascalCase`, hằng `UPPER_SNAKE` |
| Bảng DB | `snake_case` số nhiều; enum giá trị `UPPER_SNAKE` |
| API path | kebab-case số nhiều; JSON field `snake_case` |
| TypeScript | biến/hàm `camelCase`, component `PascalCase`, file component `PascalCase.tsx`, hook `useXxx` |
| Mã lỗi | `UPPER_SNAKE` (docs/07) |
| Biến môi trường | `UPPER_SNAKE` |

## Backend
- SQLAlchemy async, session theo request (`Depends`); service không gọi `commit()` trong vòng lặp lớn trừ pipeline cuốn chiếu (commit mỗi cụm).
- Kiểm quyền tập trung ở `core/deps.py`: `require_role(...)`, `get_package_scoped(id)` (trả 404 khi ngoài phạm vi).
- Mọi thay đổi trạng thái gói qua `services/state_machine.py`; mọi ghi KQ chuyên gia/chuyển trạng thái gọi `services/audit.py` trong **cùng transaction**.
- Tiền: `Decimal` → `int` đồng; cấm `float` cho tiền/% (ruff rule + TC-04).
- Lint: `ruff check`, `ruff format`, `mypy --strict app/services app/core`.

## Frontend
- App Router; server component cho layout, client component cho tương tác.
- State server: TanStack Query; form: react-hook-form + zod; không Redux.
- Client API sinh từ `/openapi.json` (`openapi-typescript`), không viết tay kiểu response.
- PDF viewer: `react-pdf`; nhảy trang bằng `pageNumber`, tô sáng bằng tìm `exact_quote` trong text layer.
- Shadcn/UI; màu kết quả theo docs/03. Accessibility: mọi nút icon có `aria-label`, điều hướng bàn phím trong bảng tiêu chí (↑/↓, Enter mở citation).

## Git
- Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`, `chore:`), tiếng Việt hoặc Anh, kèm mã task: `feat(eval): agent tool tính [TASK-012]`.
- Nhánh `feat/TASK-NNN-mo-ta`; `main` luôn xanh (CI: ruff, mypy, pytest, eslint, tsc, build).
- Không commit: `.env*` (trừ `.env.example`), `data/`, `samples/*` trừ `samples/README.md` và file đã ẩn danh (DEC-019).
