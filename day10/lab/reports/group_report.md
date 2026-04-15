# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Group C401-A03  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyễn Tuấn Kiệt | Tech Lead / Ingestion Owner | kiet.swe@gmail.com |
| Nguyễn Xuân Hoàng | Cleaning & Quality Owner | hoangnx.hust@gmail.com |
| Nguyễn Văn Bách | Eval Owner | vanbachpk1@gmail.com |
| Nguyễn Đức Duy | Retrieval & Embed Owner | ducduynguyen1307@gmail.com |
| Nguyễn Duy Hưng | Documentation Owner | hungngduy2003@gmail.com |
| Trần Trọng Giang | Monitoring & Runbook Owner | giang56b20@gmail.com |

**Ngày nộp:** 2026-04-15  
**Repo:** https://github.com/7kitan/Lecture-Day-08-09-10  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**
Pipeline Day 10 được xây dựng trên mô hình modular ingestion, tập trung vào tính quan sát (observability). Dữ liệu raw từ `policy_export_dirty.csv` (10 records) đi qua tầng `transform` để chuẩn hóa (ngày tháng, mapping doc_id), khử trùng (dedupe) và áp dụng các rule business fix (như window hoàn tiền). Kết quả được kiểm chứng bằng 8 expectations (chứa cả baseline và rule kỹ thuật mới) trước khi publish vào ChromaDB. Pipeline sinh ra log chi tiết và manifest JSON để phục vụ monitoring.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**
`python etl_pipeline.py run --run-id sprint1 && python eval_retrieval.py --out artifacts/eval/before_after_eval.csv`

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV) |
|-------------------------|------------------|-----------------------------|-----------------------|
| Strip HTML Tags | 1 failure (bản bad) | cleaned text (strip noise) | `cleaned_sprint1.csv` |
| HR Leave 20d -> 15d | data chứa 20d | data fix thành 15d | `run_sprint1.log` |
| unique_chunk_id (halt) | N/A | Total metrics=6 | `run_sprint1.log` |
| no_html_tags (halt) | N/A | Violations=0 | `run_sprint1.log` |

**Rule chính (baseline + mở rộng):**

- **Remove HTML Tags:** Tự động phát hiện và loại bỏ thẻ HTML để tránh noise cho embedding model.
- **HR Policy Version Alignment:** Chỉnh sửa các lỗi logic về số ngày phép (20 -> 15) và offboarding time để đồng bộ với policy 2026.
- **IT Floor Correction:** Update vị trí phòng IT từ Tầng 4 về Tầng 3 do thay đổi văn phòng.
- **Expectations Halt:** Setup severity `halt` cho `no_html_tags` và `unique_chunk_id` để ngăn chặn dữ liệu rác/lỗi index được publish.

**Ví dụ 1 lần expectation fail và cách xử lý:**
Khi chạy Sprint 3 với `--no-refund-fix`, expectation `refund_no_stale_14d_window` đã báo FAIL (1 violation). Tuy nhiên, team đã sử dụng `--skip-validate` để demo việc dữ liệu lỗi lọt vào vector store sẽ làm hỏng kết quả retrieval như thế nào (hits_forbidden=yes).

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**
Nhóm cố tình chạy pipeline với cờ `--no-refund-fix --skip-validate`. Tại đây, rule quan trọng giúp sửa lỗi "hoàn tiền 14 ngày" thành "7 ngày" bị bỏ qua, và system cũng bỏ qua bước validate halt để cho phép dữ liệu sai lệch này lọt vào ChromaDB (`run_id=inject-bad`).

**Kết quả định lượng (từ CSV eval):**

| Question ID | Good Run (hits_forbidden) | Bad Run (hits_forbidden) | Ghi chú |
|-------------|----------------------------|---------------------------|---------|
| `q_refund_window` | **no** | **yes** | Bad run chứa cả chunk 14 ngày |
| `q_p1_sla` | no | no | Không bị ảnh hưởng trực tiếp |
| `q_leave_version` | no | no | Đã được lọc bởi rule HR stale |

Việc `hits_forbidden` chuyển sang **yes** là bằng chứng đanh thép cho thấy pipeline đã mất đi tính observability và để lọt dữ liệu stale vào production, ảnh hưởng trực tiếp đến câu trả lời của Agent.

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

Nhóm chọn **SLA = 24 giờ** (cấu hình trong `.env`: `FRESHNESS_SLA_HOURS=24`), đo tại boundary **publish** (sau khi embed xong, manifest ghi `latest_exported_at`).

Khi chạy `run_id=sprint1`, freshness check trả về **FAIL**:

```
freshness_check=FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 116.718, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}
```

**Giải thích:** CSV mẫu có `exported_at = 2026-04-10T08:00:00` — đã cũ ~5 ngày so với thời điểm chạy (15/04). Trong production, FAIL này sẽ trigger alert theo `alert_channel` trong `data_contract.yaml` (`c401-a3-monitoring@gmail.com`). Đây là **hành vi mong đợi** trên data snapshot — nhóm ghi nhận trong `docs/runbook.md` mục Detection và giải thích rằng SLA áp cho "pipeline run" chứ không phải "data snapshot cố định".

| Trạng thái | Ý nghĩa | Hành động |
|------------|---------|----------|
| **PASS** | `age_hours ≤ 24` — dữ liệu tươi | Không cần hành động |
| **WARN** | Manifest thiếu timestamp | Kiểm tra pipeline có ghi `exported_at` không |
| **FAIL** | `age_hours > 24` — dữ liệu cũ quá SLA | Rerun pipeline hoặc cập nhật data source |

---

## 5. Liên hệ Day 09 (50–100 từ)

Việc tách collection `day10_kb` giúp chúng tôi cô lập và kiểm soát chất lượng dữ liệu tốt hơn trước khi đẩy vào pipeline Day 09. Dữ liệu sau khi "Publish" (Embed xong + Manifest ghi nhận) sẽ được service `retrieval.py` của Day 09 truy vấn. Nhờ các rule cleaning mới, Agent Day 09 sẽ không còn bị nhầm lẫn giữa các vị trí phòng IT (Tầng 3 vs 4) hay số ngày phép, từ đó tăng độ tin cậy (Groundness) của hệ thống helpdesk.

---

## 6. Rủi ro còn lại & việc chưa làm

- **Freshness chỉ đo 1 boundary (publish):** Chưa đo tại boundary ingest — nếu ingest xong nhưng embed fail, freshness vẫn báo FAIL nhưng không phân biệt được lỗi ở tầng nào. Cần thêm `ingest_timestamp` vào manifest (Distinction criteria).
- **Chưa có alert tự động:** `freshness_check.py` chỉ chạy manual hoặc cuối pipeline. Cần cron job hoặc webhook để gửi alert thực sự khi FAIL.
- **SLA cứng 24h chưa chắc phù hợp:** Policy PDF đổi 1 lần/tuần → SLA 168h có thể phù hợp hơn; ticket stream cần SLA ngắn hơn (4h). Nên phân biệt SLA theo loại nguồn.
- **Chưa có LLM-judge eval:** Hiện chỉ dùng keyword matching (`must_contain_any`, `must_not_contain`). Mở rộng với LLM-judge sẽ bắt được lỗi ngữ nghĩa tinh vi hơn.
- **Data v2 chưa test đầy đủ:** File `policy_export_dirty_v2.csv` (32 dòng) đã merge nhưng chưa chạy pipeline với `--raw` trỏ tới v2 để kiểm tra các rule mới.
