# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Nguyễn Tuấn Kiệt  
**Vai trò trong nhóm:** Tech Lead
**Ngày nộp:** 13/04/2026 
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

- Là người phân công công việc chính trong lab, chịu trách nhiệm quản lý thời gian và hoạt động của GitHub repo.
- Implement version đầu của Sprint 1, đối chiếu lại với các thành viên khác trong nhóm để tìm ra implementation nào có lỗi.
- Phối hợp team so sánh kết quả khác nhau cho implementation của Sprint 1 và phân công debug tìm ra nguyên nhân
- Implement alternative cùng lúc với người làm sprint 2 và 3 để góp ý và đối chiếu implementation
- Phối hợp với Eval Owner để chạy test và sửa `eval.py`, là người upload kết quả test cuối cùng lên repo main


_________________

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

- Hiểu rõ hơn về điểm khác biệt giữa dense và sparse retrieval. Tôi hiểu được điểm yếu của architecture RAG cũ (embedding only) - embedding có khả năng tìm nghĩa tốt nhưng có khả năng ảo giác/hallucinate khi yêu cầu là cụm từ hay cụm character chính xác.
- BM25 dựa keyword match. Mạnh ở exact match: tên riêng, mã lỗi, thuật ngữ hiếm và yếu ở việc tìm từ gần nghĩa.
- Hiểu hơn về 2 cách hỗ trợ điểm yếu của embedding only: hybrid ranking với sparse retrieval để tăng thêm weight cho các exact match, và reranking để hỗ trợ tìm kiếm từ tầng thứ 2. Initial retrieval lấy rộng (high recall), reranker model cross-encoder đánh lại relevance.

_________________

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

- Kỳ vọng reranker cải thiện rõ rệt top-k quality, thực tế gần như không đổi. Nguyên nhân chính: candidate set từ retriever đã quá nhiễu hoặc quá sạch, reranker không còn nhiều không gian để cải thiện. Việc dùng LLM để đánh giá output quality cũng khiến test khó chính xác (model bias)
- Debug khó nhất nằm ở inconsistency trong `build_index()` giữa các thành viên. Kết quả từng chunk có thể reproduce (print) nhưng rất tốn thời gian phân tích
- ChromaDB score issue gây hiểu nhầm. Một số pipeline trả negative similarity do dùng cosine distance thay vì similarity, dẫn tới so sánh sai ngưỡng khi evaluate. API của ChromaDB mới, không biết phải dùng metadata cosine cũng gây ra vấn đề trong lúc debug
- Giả thuyết ban đầu: chỉ cần thêm cross-encoder reranker là precision tăng đáng kể. Trong thực tế thì không hơn mấy (+0.20 completeness only). Khó đo được chính xác improvement khi chạy LLM-as-judge.

_________________

---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:**  "ERR-403-AUTH là lỗi gì và cách xử lý?"
**Phân tích:**
ERR-403-AUTH là HTTP 403 biến thể liên quan authentication/authorization. Server đã nhận request nhưng từ chối quyền truy cập do thiếu scope, role sai, token không hợp lệ với resource.

Baseline trả lời đúng hướng kỹ thuật: phân biệt 403 vs 401, nêu nguyên nhân phổ biến như expired token, missing permission, sai ACL. Điểm quan trọng là model vẫn “đoán” thay vì dừng lại khi thiếu context. Điều này tạo ra câu trả lời có vẻ hữu ích nhưng không xác minh được nguồn lỗi cụ thể.

Quan sát đáng chú ý: không có cơ chế fallback rõ ràng “insufficient info”. Model chọn giải thích chung, dẫn đến risk hallucination nhẹ ở phần nguyên nhân.

Điểm thú vị là dạng trả lời này vẫn hữu dụng trong pipeline như HyDE hoặc retrieval expansion, vì nó tạo pseudo-query giàu ngữ nghĩa để tìm tài liệu liên quan. Tuy nhiên trong production debugging, cần thêm guardrail: yêu cầu log hoặc error trace trước khi kết luận nguyên nhân.

___________



_________________

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

- Thử tune chunk size + overlap vì eval cho thấy recall thấp ở query dài, nhiều context bị split sai ranh giới. Mục tiêu: tăng context continuity, giảm missing fact khi retrieval. Có thể test nhiều chunk size khác nhau (350, 400, 450, 500) nếu có thời gian. Thực tế là cả team phải chọn thống nhất nhanh tại các phần khác dài.
- Chạy và test chunker kĩ hơn, đặc biệt là sentence-level chunker. Trong hiện tại implementation của chunker vẫn còn edge case gây ra lỗi cắt giữa câu. Implementation chunker đúng sẽ giúp recall.


_________________

---

*Lưu file này với tên: `reports/individual/[ten_ban].md`*
*Ví dụ: `reports/individual/nguyen_van_a.md`*
