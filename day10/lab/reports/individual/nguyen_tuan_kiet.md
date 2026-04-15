# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Tuấn Kiệt  
**Vai trò:** Team Lead + Data Preparation/Ingestion
**Ngày nộp:** 15/04/2026

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `data_contract.md`, `contracts/data_contract.yaml`: Định nghĩa SLA, Schema và Owner cho pipeline.
- `data/raw/policy_export_dirty_v2.csv`: Xây dựng bộ dữ liệu "bẩn" mẫu (injected failures) từ tài liệu thực tế.
- `etl_pipeline.py`: Điều phối thực thi pipeline, quản lý log và manifest (đã chạy thành công có `manifest_sprint1.json` và `run_run1-v2.log`).
- Phân công công việc cho các thành viên trong team.

**Kết nối với thành viên khác:**

Tôi đã xây dựng bộ dữ liệu `policy_export_dirty_v2.csv` với các failure modes chủ ý (mất ngày, sai phiên bản, trùng lặp) để tạo "đề bài" thực tế cho **Cleaning/Quality Owner** kiểm thử các rule và expectation mới. Ngoài ra, việc tôi định nghĩa SLA và Schema trong `data_contract.yaml` cung cấp tham chiếu tiêu chuẩn cho **Monitoring Owner** thực hiện các lệnh `freshness check`. Toàn bộ manifest và file cleaned từ các run của tôi là đầu vào bắt buộc để **Embed Owner** thực hiện đánh giá hiệu quả retrieval (Before/After) trên bộ golden questions.

**Bằng chứng (commit / comment trong code):**

- commit `5e34b45`: docs/data_contract.md based on data_contract.yaml
- commit `38802d1`: Phân công công việc, add TODO vào các file cho các thành viên search
- commit `1c7fa2e`: Chạy pipeline trên data raw, upload log cho sprint 1
- commit `7166933`: Chuẩn bị data lỗi (sprint 3) dựa trên các rule mới từ sprint 2

---

## 2. Một quyết định kỹ thuật (100–150 từ)

- Tôi đã quyết định thêm vào các loại data bẩn mới, bao gồm HTML tag, trùng lặp doc_id, sai thông tin. Việc test kĩ càng các edge case giúp các thành viên bàn luận và đưa ra expectation mới, phù hợp. Đi cùng với đó là việc quyết định cùng team thiết kế quarantine format theo từng loại lỗi mới, giúp việc debug và đánh giá hiệu quả của từng rule dễ dàng hơn. Trong data mới, các hàng thông tin sai cũng có data đúng rõ ràng (ví dụ: `Thông tin sai B (file xxx nói là A không phải B)`) để hỗ trợ debug rule.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

- Khi chạy pipeline trên dữ liệu `policy_export_dirty_v2.csv` mới, tôi phát hiện lỗi `ValueError: ... is not in the subpath of ...` khiến quy trình ingestion bị dừng đột ngột (run_id: run1-v2). Lỗi này xảy ra khi tôi thực thi lệnh từ terminal với đường dẫn file raw dạng tương đối. Qua kiểm tra stack trace, tôi xác định nguyên nhân là do thư viện `pathlib` không thể xử lý logic `relative_to` khi một bên là đường dẫn tuyệt đối (biến `ROOT`) và một bên chưa được resolve. Tôi đã xử lý triệt để bằng cách bổ sung phương thức `.resolve()` vào `etl_pipeline.py` tại mọi vị trí tính toán lineage cho manifest và log. Sau khi fix, pipeline đã thực thi thành công (status: PIPELINE_OK), đảm bảo dữ liệu được nạp vào hệ thống một cách ổn định bất kể cấu trúc thư mục làm việc của người dùng.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Dưới đây là bằng chứng đối sánh giữa lần chạy dữ liệu bị "nhiễm" thông tin cũ (`inject-bad`) và lần chạy đã được làm sạch (`run1-v2`). Kết quả cho thấy sự khác biệt ở `hits_forbidden` khi hệ thống truy xuất thông tin về kỳ hạn hoàn tiền (refund window).

- **Trước (Run: inject-bad):** `q_refund_window`, ..., `hits_forbidden=yes` (Context vẫn chứa chunk lỗi 14 ngày do bỏ qua bước validate/fix).
- **Sau (Run: run1-v2):** `q_refund_window`, ..., `hits_forbidden=no` (Hệ thống đã fix thành 7 ngày, không còn vết tích thông tin cũ).

Trích xuất từ `bad_eval.csv` vs `good_eval.csv`:

```csv
q_refund_window,...,yes,yes,,3 (inject-bad)
q_refund_window,...,yes,no,,3 (run1-v2)
```

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ xây dựng một bộ dữ liệu `golden_questions_with_answers.csv` chứa các câu hỏi mẫu và câu trả lời chuẩn tương ứng với keyword có thể tìm và test được. Việc này sẽ cho phép tôi định nghĩa rõ ràng các `expected_answers` trong `data_contract.yaml` và thực hiện đánh giá retrieval (Before/After) một cách định lượng hơn, thay vì chỉ dựa vào `hits_forbidden`.
