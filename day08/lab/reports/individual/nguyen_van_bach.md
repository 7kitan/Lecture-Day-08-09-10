# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Nguyễn Văn Bách
**Vai trò trong nhóm:** Tech Lead
**Ngày nộp:** 2026-04-13
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

Trong buổi lab này, tôi đảm nhận vai trò Tech Lead, chịu trách nhiệm thiết lập nền tảng dự án và kết nối các Sprint. Tôi đã trực tiếp triển khai toàn bộ quy trình Indexing (Sprint 1), bao gồm việc viết hàm tiền xử lý tài liệu để trích xuất metadata và xây dựng cơ chế chunking thông minh dựa trên đề mục (heading-based) để giữ được ngữ cảnh toàn vẹn nhất cho các chính sách thực tế. Ở Sprint 2 và 3, tôi tập trung vào việc tích hợp Hybrid Retrieval (Dense + BM25) và Rerank sử dụng Cross-encoder để tối ưu hóa khả năng tìm kiếm chính xác cho các bộ câu hỏi có chứa mã lỗi kỹ thuật như của IT Helpdesk. Cuối cùng, tôi hỗ trợ nhóm chạy Evaluation và phân tích scorecard để đưa ra các nhận xét về sự cải thiện của Variant so với Baseline.

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

Sau khi trực tiếp thực hiện Lab 8, tôi hiểu sâu sắc hơn về tầm quan trọng của **Evaluation Loop** và **Grounded Prompting**. Trước đây, tôi thường nghĩ chỉ cần retrieval trả về kết quả là LLM sẽ trả lời đúng. Tuy nhiên, qua thực tế, tôi nhận ra rằng nếu prompt không được thiết kế chặt chẽ để "ép" model chỉ trả lời dựa trên context (grounding), model rất dễ tự bịa thêm thông tin dựa trên kiến thức có sẵn (hallucination), đặc biệt là với các câu hỏi về chính sách nội bộ mà model không hề biết trước. Ngoài ra, việc so sánh giữa Dense và Hybrid Retrieval giúp tôi nhận ra rằng Vector Search không phải lúc nào cũng là "viên đạn bạc"; với các từ khóa ngắn và mã lỗi đặc thù, BM25 vẫn đóng vai trò cực kỳ quan trọng trong việc tăng Recall.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

Điều làm tôi mất nhiều thời gian nhất là việc xử lý **metadata matching** trong phần Evaluation. Hệ thống báo lỗi Context Recall bằng 0 dù câu trả lời hoàn toàn đúng. Sau khi debug, tôi phát hiện ra sự sai lệch nhỏ giữa tên file thực tế (dùng dấu gạch dưới `_`) và tên file được kỳ vọng trong test set (dùng dấu gạch ngang `-`). Đây là một bài học thực tế quý giá về sự chuẩn hóa dữ liệu (data normalization) trong các hệ thống RAG thực chiến. Một khó khăn khác là việc load model Cross-encoder cho phần Rerank khá nặng, gây chậm trễ trong quá trình eval, đòi hỏi tôi phải triển khai cơ chế cache model để không phải tải lại mỗi khi gọi hàm, giúp tối ưu hiệu năng đáng kể.

---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:** "Approval Matrix để cấp quyền hệ thống là tài liệu nào?" (q07)

**Phân tích:** 
Câu hỏi này là một trường hợp thú vị vì nó chứa "alias" - tên cũ của tài liệu. Trong tài liệu thực tế, tên của nó đã được đổi thành "Access Control SOP". 
- **Baseline (Dense):** Đã gặp khó khăn trong việc tìm ra mối liên hệ trực tiếp nếu embedding model không đủ mạnh để hiểu ngữ cảnh chuyển đổi tên gọi. Điểm Completeness của Baseline cho câu này khá thấp (2/5) vì model trích dẫn được tài liệu nhưng không giải thích rõ được sự thay đổi tên gọi một cách đầy đủ.
- **Variant (Hybrid + Rerank):** Nhờ cơ chế BM25 trong Hybrid Search, hệ thống đã bắt được keyword "Approval Matrix" xuất hiện ngay trong đoạn mô tả lịch sử phiên bản của tài liệu "Access Control SOP". Sau khi Rerank được áp dụng, chunk văn bản chứa thông tin về việc đổi tên này được đưa lên top đầu, giúp LLM có đủ dữ liệu để trả lời chính xác rằng tài liệu hiện nay là Access Control SOP. Điều này chứng minh rằng việc kết hợp Keyword search là cực kỳ cần thiết cho các kho dữ liệu có sự thay đổi thuật ngữ hoặc sử dụng nhiều bí danh.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

Nếu có thêm thời gian, tôi muốn thử nghiệm thêm kỹ thuật **Query Expansion** (Sử dụng LLM để viết lại câu hỏi thành nhiều dạng khác nhau trước khi retrieve). Kết quả eval cho thấy đối với các câu hỏi phức tạp hoặc có chứa thuật ngữ cũ, Context Recall của chúng ta vẫn có thể cải thiện thêm được 10-15%. Ngoài ra, tôi sẽ tối ưu hóa lại phần chunking để tự động nhận diện các bảng biểu (table parsing), vì các tài liệu SLA thường chứa bảng thời gian phản hồi mà hiện tại cơ chế chunking theo paragraph đang xử lý chưa thực sự hoàn hảo.
