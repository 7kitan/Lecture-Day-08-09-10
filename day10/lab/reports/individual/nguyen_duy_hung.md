# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Duy Hưng  
**Vai trò:** Cleaning & Pipeline Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài:** ~550 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**

- `transform/cleaning_rules.py` — Thiết kế và cài đặt toàn bộ 6 luật làm sạch Sprint 2: strip HTML tags, normalize `doc_id` về lowercase, và 5 luật thay thế nội dung chính sách lỗi thời theo từng `doc_id`.
- `quality/expectations.py` — Thêm 2 expectation kỹ thuật mới: `no_html_tags` (halt) và `unique_chunk_id` (halt), đồng bộ tên ID với mục `quality_rules` trong `contracts/data_contract.yaml`.
- `contracts/data_contract.yaml` — Điền thông tin `owner_team`, `alert_channel`, cập nhật `canonical_sources` sang `policy_export_dirty_v2.csv`, bổ sung 5 rule nghiệp vụ và 2 rule kỹ thuật.
- `etl_pipeline.py` — Sửa bug đường dẫn tương đối/tuyệt đối (ValueError `relative_to`) và cập nhật `RAW_DEFAULT` sang file `v2`.

**Kết nối với thành viên khác:** Tôi cung cấp tên rule chính xác (ví dụ `no_stale_hr_leave_20d`) để thành viên Docs Owner dùng trong `data_contract.md` và `pipeline_architecture.md`.

**Bằng chứng:** commit `feat: complete Day 10 Sprint 2 with business rules and technical expectations` trên nhánh `main`, tag `[cleaned: no_stale_it_floor_4]` trong `cleaned_sprint2_final_test.csv`.

---

## 2. Một quyết định kỹ thuật

**Chọn `halt` thay vì `warn` cho `no_html_tags` và `unique_chunk_id`:**

Khi thiết kế 2 expectation mới, tôi cân nhắc giữa severity `warn` (pipeline tiếp tục) và `halt` (dừng ngay). Tôi chọn `halt` cho cả hai vì:

- **`no_html_tags`:** Nếu thẻ HTML (`<b>`, `<i>`) lọt vào ChromaDB, embedding vector sẽ bị nhiễu ký tự đặc biệt — AI có thể trả lời sai hoặc lạc chủ đề. Đây là lỗi không thể chấp nhận trong production.
- **`unique_chunk_id`:** Nếu tồn tại ID trùng, lệnh `upsert` sẽ ghi đè dữ liệu sai, gây ra tình trạng vector cũ bị "đội lốt" ID mới — không thể phát hiện bằng mắt thường.

Dùng `warn` với 2 trường hợp này sẽ che giấu lỗi thay vì ngăn nó từ đầu. Kết quả: run `sprint2_final_test` toàn bộ 8 expectation đều `OK (halt)`.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Chạy lệnh `python etl_pipeline.py run --raw data/raw/policy_export_dirty_v2.csv` bị crash với `ValueError: 'data\raw\policy_export_dirty_v2.csv' is not in the subpath of ...`.

**Metric phát hiện:** Log in ra đủ các bước (raw=31, clean=15, expectations OK) nhưng crash ngay trước khi ghi Manifest — nghĩa là dữ liệu đã được embed vào ChromaDB nhưng **không có file Manifest**, vi phạm yêu cầu `artifacts/manifests/*.json` phải tồn tại để chấm điểm.

**Fix:** Trong `etl_pipeline.py` dòng 109, đổi `raw_path.relative_to(ROOT)` thành `raw_path.resolve().relative_to(ROOT)`. Nguyên nhân: `raw_path` là đường dẫn tương đối (relative) trong khi `ROOT` là tuyệt đối (absolute) do `.resolve()`. Sau fix, manifest `manifest_sprint2_final_test.json` được tạo thành công và `PIPELINE_OK` in ra cuối log.

---

## 4. Bằng chứng trước / sau

**Trước Sprint 2** (`run_id=sprint1`, file `v1` — 10 dòng thô, 6 expectations baseline):
```
raw_records=10  cleaned_records=6  quarantine_records=4
freshness_check=FAIL {"age_hours": 116.718, "sla_hours": 24.0}
```

**Sau Sprint 2** (`run_id=sprint2_final_test`, file `v2` — 31 dòng thô):
```
raw_records=31  cleaned_records=15  quarantine_records=16
expectation[no_html_tags] OK (halt) :: violations=0
expectation[unique_chunk_id] OK (halt) :: total=15
freshness_check=PASS {"age_hours": -1.712, "sla_hours": 24.0}
PIPELINE_OK
```

Delta đáng chú ý: `cleaned_records` tăng từ 6 → 15 (x2.5), `quarantine_records` tăng từ 4 → 16 cho thấy file `v2` chứa nhiều dữ liệu bẩn hơn đáng kể (>50% bị loại).

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ tách các luật thay thế nội dung ra khỏi `cleaning_rules.py` vào một file cấu hình riêng `rules_config.yaml`, để Pipeline đọc luật từ YAML thay vì hard-code trong Python. Điều này giúp khi chính sách thay đổi (ví dụ SLA từ 4 giờ thành 6 giờ), nhóm nghiệp vụ chỉ cần sửa YAML, không cần đụng vào code Python — giảm rủi ro lỗi regression.
