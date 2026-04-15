# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| Policy Refund v4 | CSV export (Policy DB) | Stale window (14d vs 7d) | expectation[no_stale_refund_window] |
| SLA P1 2026 | CSV export (SLA Tracker) | Duplicate contents | expectation[no_duplicate_chunk_text] |
| IT Helpdesk FAQ | CSV export (Confluence API) | Schema mismatch | quarantine_records > 0 |
| HR Leave Policy | CSV export (HR Portal) | Data lag / Stale version | check_manifest_freshness / FAIL |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | ID ổn định (hash hoặc doc_id + seq) |
| doc_id | string | Có | Khóa logic (vd: policy_refund_v4) |
| chunk_text | string | Có | Nội dung chunk (min_length: 8) |
| effective_date | date | Có | Ngày hiệu lực (ISO YYYY-MM-DD) |
| exported_at | datetime | Có | Thời điểm export từ nguồn |

---

## 3. Quy tắc quarantine vs drop

> Record lỗi đi vào file `quarantine_<id>.csv`. Data Owner định kỳ review và fix nguồn phát (DB/CSV) để replay.

- `unknown_doc_id`: ID không thuộc allowlist.
- `invalid_effective_date_format`: Không parse được ngày.
- `stale_hr_policy_effective_date`: Bản HR cũ (< 2026).
- `duplicate_chunk_text`: Loại trùng nội dung.

---

## 4. Phiên bản & canonical

> `contracts/data_contract.yaml` định nghĩa `canonical_sources` và `policy_versioning`.

- Refund policy chuẩn: v4 (7 ngày làm việc).
- HR policy chuẩn: min effective date ≥ 2026-01-01.
