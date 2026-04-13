# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Nguyễn Văn Bách  
**Vai trò trong nhóm:** Eval Owner  
**Ngày nộp:** 2026-04-13  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này?

Trong dự án IT Helpdesk RAG lần này, tôi đóng vai trò là người chịu trách nhiệm chính cho **Sprint 4: Evaluation & Scorecard**. Công việc của tôi tập trung vào việc hiện thực hóa file eval.py, nơi thiết lập toàn bộ quy trình đánh giá chất lượng của hệ thống RAG qua 10 câu hỏi kiểm thử (grading_questions.json).

Cụ thể, tôi đã:
- Triển khai Logic **LLM-as-Judge** để chấm điểm tự động cho các chỉ số Faithfulness, Answer Relevance và Completeness bằng cách gọi lại LLM (GPT-4o-mini) với các prompt chấm điểm chuyên biệt.
- Xây dựng hàm tính toán **Context Recall** dựa trên việc so sánh tên file nguồn thực tế được trả về so với danh sách nguồn mong đợi (expected_sources).
- Kết nối pipeline từ Sprint 2 và 3 để chạy so sánh **A/B Testing** giữa Baseline (Dense Retrieval) và Variant (Hybrid + Rerank). 
- Công việc của tôi là "chốt chặn" cuối cùng, giúp cả nhóm nhìn thấy con số cụ thể về hiệu quả của các cải tiến kỹ thuật.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Sau lab này, tôi đã thực sự "vỡ lòng" về khái niệm **LLM-as-Judge**. Trước đây tôi luôn tự hỏi làm sao để đánh giá một câu trả lời văn bản tự động mà không cần can thiệp thủ công. Việc viết prompt để LLM tự chấm điểm cho LLM khác (với các tiêu chí rõ ràng như "có bám sát bằng chứng không?") giúp tôi hiểu rằng Evaluation trong RAG không chỉ là đúng/sai mà là sự đo lường về độ tin cậy.

Bên cạnh đó, tôi hiểu sâu sắc hơn về **Context Recall**. Tôi nhận ra rằng ngay cả khi LLM trả lời rất hay (Relevance cao), nhưng nếu nó không tìm đúng tài liệu nguồn (Recall thấp) thì đó vẫn là một rủi ro Hallucination tiềm ẩn. Chỉ số Recall giúp tôi biết chính xác bước Retrieval đã làm tốt hay chưa, tách biệt hoàn toàn với khả năng hành văn của LLM ở bước Generation.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Điều khiến tôi ngạc nhiên nhất là việc xử lý **Insufficient Context** (trường hợp không có dữ liệu trong docs). Ban đầu, tôi giả thuyết rằng nếu mình bảo LLM "say I don't know", nó sẽ luôn được điểm tối đa về Faithfulness. Tuy nhiên, thực tế ở câu `gq07`, Baseline chỉ nhận được 1/5 điểm. 

Lỗi này mất nhiều thời gian debug vì tôi nhận ra prompt chấm điểm của tôi chưa bao quát được trường hợp "Từ chối trả lời đúng". Khi LLM nói "Tôi không biết", giám khảo (Judge) lại chấm là 1 điểm vì cho rằng nó "không cung cấp thông tin hữu ích", trong khi thực tế đó lại là hành vi mong muốn (abstain). Tôi đã phải điều chỉnh logic chấm điểm để ghi nhận sự trung thực của model là một điểm cộng thay vì điểm trừ. Khó khăn khác là việc khớp nối tên file nguồn (source matching) vì định dạng đường dẫn của mỗi người trong nhóm có thể khác nhau, buộc tôi phải dùng regex và normalization để xử lý.

---

## 4. Phân tích một câu hỏi trong scorecard

> [!NOTE] 
> Dưới đây là 2 phương án phân tích, bạn có thể chọn 1 trong 2 cái phù hợp nhất để giữ lại.

### Phương án 1: Phân tích về lỗi "Từ chối trả lời" (Abstain)
**Câu hỏi:** "Công ty sẽ phạt bao nhiêu nếu team IT vi phạm cam kết SLA P1?" (ID: `gq07`)

**Phân tích:**
- **Baseline trả lời:** "Hệ thống hiện tại không có dữ liệu về mức phạt vi phạm SLA P1. Vui lòng liên hệ bộ phận quản lý để biết thêm chi tiết."
- **Điểm số:** Faithfulness: 1/5, Relevance: 1/5, Context Recall: N/A, Completeness: 1/5.

Đây là một trường hợp cực kỳ thú vị. Về mặt kỹ thuật, hệ thống RAG đã làm **đúng nhiệm vụ** của quy tắc "Abstain": vì trong tài liệu `sla-p1-2026.pdf` hoàn toàn không đề cập đến "penalty", model đã từ chối bịa đặt (hallucination). Tuy nhiên, scorecard lại chấm điểm thấp tệ hại.

Tại sao? Lỗi nằm ở **Evaluation Logic**. Bộ tiêu chí `expected_answer` trong file JSON yêu cầu model phải nêu rõ là "Tài liệu hiện có không quy định...". LLM Judge khi so sánh câu trả lời ngắn gọn "Tôi không biết" với một câu trả lời mẫu đầy đủ lý lẽ đã đánh giá thấp tính "Completeness" và "Relevance". Điều này dạy cho tôi bài học rằng: để đánh giá RAG chuẩn, không chỉ cần pipeline tốt mà bộ dữ liệu Ground Truth (Expected Answers) và Prompt dành cho Judge cũng phải cực kỳ tinh tế. Nếu chỉ chăm chăm vào việc tìm "keyword", ta sẽ vô tình trừng phạt những model có tính an toàn cao.

### Phương án 2: Phân tích về khả năng tổng hợp đa tài liệu (Multi-doc Synthesis)
**Câu hỏi:** "Khi làm việc remote, tôi phải dùng VPN và được kết nối trên tối đa bao nhiêu thiết bị?" (ID: `gq02`)

**Phân tích:**
- **Baseline trả lời:** "Khi làm việc từ xa, bạn phải sử dụng VPN Cisco AnyConnect. Bạn được phép kết nối tối đa trên 2 thiết bị cùng lúc."
- **Điểm số:** Faithfulness: 5/5, Relevance: 5/5, Context Recall: 5/5, Completeness: 3/5.

Câu hỏi này kiểm tra kỹ năng khó nhất là **Multi-document Synthesis**. Thông tin về việc "phải dùng VPN" nằm ở file chính sách nhân sự (`hr/leave-policy-2026.pdf`), còn con số "tối đa 2 thiết bị" lại nằm ở file hỗ trợ kỹ thuật (`support/helpdesk-faq.md`). 

Baseline đã thành công trong việc trích xuất được cả hai thông tin từ hai nguồn khác nhau. Tuy nhiên, điểm Completeness chỉ đạt 3/5 vì model đã bỏ lỡ một chi tiết nhỏ nhưng quan trọng trong expected answer: "VPN là bắt buộc theo HR Leave Policy". Mặc dù model đã nêu đúng các ý chính, nhưng việc thiếu citation cụ thể cho từng ý (liên kết ý 1 với file HR, ý 2 với file IT) làm giảm tính minh bạch của câu trả lời. Điều này cho thấy hệ thống cần một "Rerank" tốt hơn để ưu tiên các chunk có metadata rõ ràng hơn, giúp LLM dễ dàng trích dẫn nguồn cho từng phân đoạn thông tin.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Nếu có thêm thời gian, tôi sẽ triển khai **Multi-turn Evaluation**. Hiện tại chúng ta mới đánh giá single-turn (hỏi-đáp một lần), nhưng trong thực tế Helpdesk, khách hàng thường hỏi dồn dập nhiều câu liên quan.

Tôi sẽ thử xây dựng một kịch bản stress-test để xem chỉ số Context Recall thay đổi thế nào khi câu hỏi sau phụ thuộc vào context của câu hỏi trước. Dựa trên kết quả eval hiện tại (Completeness đang ở mức trung bình 3/5), tôi tin rằng việc tối ưu hóa **Recursive Character Text Splitter** (thay vì split cố định) sẽ giúp tăng độ phủ của context, từ đó cải thiện tính đầy đủ của câu trả lời mà không làm tăng độ nhiễu.

---
