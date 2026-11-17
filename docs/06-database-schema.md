# 06 — Lược đồ dữ liệu

PostgreSQL 16 + `pgvector`. Khóa chính `uuid` (`gen_random_uuid()`) trừ `audit_logs` (`bigserial`). Mọi bảng có `created_at timestamptz default now()`; bảng sửa được có `updated_at`. Tiền = `bigint` (đồng); không cột `float`/`real`/`double` nào (DEC-036, TC-04).

## ERD rút gọn
```
users ─< package_assignments >─ packages ─< bidders ─< evaluations >─ criteria >─ criteria_sets
                                   │           │            ├─< citations >─ documents
                                   │           └─< documents ─< pages
                                   │                    └─< chunks (vector)
                                   └─< pipeline_steps     evaluations ─< tool_calls
kb_documents ─ documents(kind=KB) ─< chunks              report_exports, audit_logs, refresh_tokens
```

## Bảng
### users
`id`, `email citext unique`, `password_hash text`, `full_name varchar(255)`, `role` enum `ADMIN|CHUYEN_VIEN|LANH_DAO`, `is_active bool default true`, `failed_logins int default 0`, `locked_until timestamptz null`.

### refresh_tokens
`id`, `user_id → users`, `token_hash text unique`, `expires_at`, `revoked_at null`. Index `(user_id)`.

### packages
`id`, `code varchar(50) unique`, `name varchar(500)`, `closing_date date`, `status` enum `DRAFT|PROCESSING|EXPERT_REVIEW|LEADER_REVIEW|COMPLETED`, `criteria_set_id → criteria_sets null`, `return_reason text null`, `completed_at null`, `created_by → users`. Index `(status)`.

### package_assignments
`package_id → packages`, `user_id → users`; PK `(package_id, user_id)`.

### bidders
`id`, `package_id → packages on delete cascade`, `name varchar(500)`, `tax_code varchar(13) null`, `bid_price_vnd bigint null check (>= 0)`. Unique `(package_id, tax_code)` where `tax_code is not null`.

### documents
`id`, `kind` enum `HSMT|EHSDT|KB`, `package_id null`, `bidder_id null`, `original_name varchar(500)`, `original_path text`, `pdf_path text null`, `sha256 char(64)`, `size_bytes bigint`, `page_count int null`, `status` enum `UPLOADED|PROCESSING|READY|FAILED`. Check: `kind='EHSDT'` ⇒ `bidder_id not null`; `kind='KB'` ⇒ `package_id is null`.

### pages
`id`, `document_id → documents cascade`, `page_number int` (bắt đầu từ 1), `text text`, `is_scanned bool`, `tables jsonb null`. Unique `(document_id, page_number)`.

### chunks
`id`, `document_id → documents cascade`, `page_start int`, `page_end int`, `text text`, `embedding vector(1024)` (DEC-026), `meta jsonb` (KB: `{article, clause}`). Index HNSW `embedding vector_cosine_ops`; index `(document_id)`.

### kb_documents
`id`, `doc_key varchar(100)` (vd `luat-dau-thau-22-2023`), `title varchar(500)`, `kb_type` enum `LAW|DECREE|CIRCULAR|INTERNAL`, `version int`, `effective_date date`, `status` enum `ACTIVE|SUPERSEDED`, `document_id → documents`. Unique `(doc_key, version)`; unique partial `(doc_key) where status='ACTIVE'` (DEC-041).

### criteria_sets
`id`, `name varchar(255)`, `is_template bool`, `source_set_id → criteria_sets null`, `package_id → packages null`, `locked_at null`. Check: `is_template` ⇔ `package_id is null`.

### criteria
`id`, `set_id → criteria_sets cascade`, `form` enum `MAU_01|MAU_02|MAU_02B`, `code varchar(20)` (STT: `2.1.1`), `parent_id → criteria null`, `title text`, `requirement text` (ngôn ngữ tự nhiên, DEC-037), `kind` enum `QUALITATIVE|QUANTITATIVE|GROUP`, `mandatory bool default true`, `sort_order int`. Unique `(set_id, form, code)`.

### evaluations
`id`, `bidder_id → bidders cascade`, `criterion_id → criteria`, `ai_result` enum `DAT|KHONG_DAT|CAN_LAM_RO|PENDING`, `ai_reason text`, `uncertain_reasons text[]`, `extracted_fields jsonb`, `calculation_steps jsonb`, `replay_ok bool null`, `legal_refs jsonb`, `expert_result` enum `DAT|KHONG_DAT null`, `expert_comment text null`, `reviewed_by → users null`, `reviewed_at null`, `model_name varchar(200)`, `prompt_version varchar(20)`. Unique `(bidder_id, criterion_id)`.

### citations
`id`, `evaluation_id → evaluations cascade`, `document_id → documents`, `file_name varchar(500)`, `page_number int`, `exact_quote text`, `quote_verified bool`, `field_path varchar(200) null` (trường nào trong `extracted_fields`).

### tool_calls
`id`, `evaluation_id → evaluations cascade`, `seq int`, `expression text`, `inputs jsonb` (giá trị dạng chuỗi Decimal), `result text`, `replay_result text null`, `replay_match bool null`. Unique `(evaluation_id, seq)`.

### pipeline_steps
`id`, `package_id → packages cascade`, `document_id null`, `bidder_id null`, `criterion_id null`, `step_type` enum `CONVERT|READ_BATCH|OCR_PAGE|EMBED_BATCH|EVALUATE|EXPORT`, `step_key varchar(200)` (vd `<doc>:p9-16`), `status` enum `PENDING|RUNNING|DONE|LOI`, `attempts int default 0`, `last_error text null`, `started_at`, `finished_at`. Unique `(step_type, step_key)`. Index `(package_id, status)`.

### report_exports
`id`, `package_id`, `file_path text`, `is_draft bool`, `created_by → users`.

### audit_logs (chỉ INSERT — DEC-021)
`id bigserial`, `actor_id → users null` (null = hệ thống), `action varchar(50)` (vd `PACKAGE_STATUS_CHANGED`, `EXPERT_RESULT_SET`), `entity varchar(50)`, `entity_id uuid`, `before jsonb`, `after jsonb`, `ip inet null`, `created_at`.
Bảo vệ: trigger `BEFORE UPDATE OR DELETE` raise exception; role ứng dụng chỉ có `INSERT, SELECT` (TC-05).

## Bất biến cấp DB
- `packages.status='COMPLETED'` ⇒ trigger chặn UPDATE `evaluations`, `criteria`, `bidders` thuộc gói (TC-06).
- Không cột số thực cho tiền (TC-04).
