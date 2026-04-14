# Routing Decisions Log — Lab Day 09

**Nhóm:** Nhóm C401 - A3
**Ngày:** 14/04/2026

> **Hướng dẫn:** Ghi lại ít nhất **3 quyết định routing** thực tế từ trace của nhóm.
> Không ghi giả định — phải từ trace thật (`artifacts/traces/`).
> 
> Mỗi entry phải có: task đầu vào → worker được chọn → route_reason → kết quả thực tế.

---

## Routing Decision #1

**Task đầu vào:**
> Ticket P1 được tạo lúc 22:47. Đúng theo SLA, ai nhận thông báo đầu tiên và qua kênh nào? Deadline escalation là mấy giờ?

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `default route`  
**MCP tools được gọi:** None  
**Workers called sequence:** `retrieval_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): [PLACEHOLDER] Câu trả lời được tổng hợp từ 1 chunks.
- confidence: 0.75
- Correct routing? Yes

**Nhận xét:** Quyết định đúng. Vì task liên quan đến SLA và quy trình escalation cho ticket P1, supervisor đã ưu tiên route vào retrieval_worker để truy xuất file sla_p1_2026.txt.

---

## Routing Decision #2

**Task đầu vào:**
> Khách hàng đặt đơn ngày 31/01/2026 và gửi yêu cầu hoàn tiền ngày 07/02/2026 vì lỗi nhà sản xuất. Sản phẩm chưa kích hoạt, không phải Flash Sale, không phải kỹ thuật số. Chính sách nào áp dụng và có được hoàn tiền không?

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `task contains policy/access keyword`  
**MCP tools được gọi:** None  
**Workers called sequence:** `policy_tool_worker`, `retrieval_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): [PLACEHOLDER] Câu trả lời được tổng hợp từ 1 chunks.
- confidence: 0.75
- Correct routing? Yes

**Nhận xét:** Quyết định đúng. Supervisor nhận diện được keyword "hoàn tiền" và route vào policy_tool_worker để kiểm tra các quy định đặc biệt trước khi thực hiện retrieval.

---

## Routing Decision #3

**Task đầu vào:**
> Sự cố P1 xảy ra lúc 2am. Đồng thời cần cấp Level 2 access tạm thời cho contractor để thực hiện emergency fix. Hãy nêu đầy đủ: (1) các bước SLA P1 notification phải làm ngay, và (2) điều kiện để cấp Level 2 emergency access.

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `task contains policy/access keyword | risk_high flagged`  
**MCP tools được gọi:** None  
**Workers called sequence:** `policy_tool_worker`, `retrieval_worker`, `synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): [PLACEHOLDER] Câu trả lời được tổng hợp từ 1 chunks.
- confidence: 0.75
- Correct routing? Yes

**Nhận xét:** Đây là câu hỏi multi-hop khó. Supervisor đã nhận diện được cả keyword về quyền truy cập ("access") và mức độ rủi ro cao ("emergency", "2am"), nên đã kích hoạt flag `risk_high`.

---

## Tổng kết

### Routing Distribution

| Worker | Số câu được route | % tổng |
|--------|------------------|--------|
| retrieval_worker | 5 | 50% |
| policy_tool_worker | 5 | 50% |
| human_review | 0 | 0% |

### Routing Accuracy

- Câu route đúng: 10 / 10
- Câu route sai (đã sửa bằng cách nào?): 0
- Câu trigger HITL: 0

### Lesson Learned về Routing

> Quyết định kỹ thuật quan trọng nhất nhóm đưa ra về routing logic là gì?  
> (VD: dùng keyword matching vs LLM classifier, threshold confidence cho HITL, v.v.)

1. **Keyword Matching mang lại hiệu năng cao:** Việc sử dụng bộ từ khóa giúp giảm độ trễ (latency) xuống mức gần như bằng 0 và tiết kiệm chi phí API call so với việc dùng LLM Classifier cho mọi câu hỏi.
2. **Quản lý rủi ro qua Error Codes:** Việc nắn dòng các task có mã lỗi "err-" sang Human Review là một quyết định an toàn và hiệu quả để xử lý các trường hợp hệ thống chưa được train.

### Route Reason Quality

Các `route_reason` hiện tại (như "contains policy keyword") đã cung cấp đủ thông tin cơ bản để biết tại sao Supervisor đưa ra quyết định. Tuy nhiên, để cải thiện khả năng debug, nhóm dự kiến sẽ bổ sung thêm các "matched keywords" cụ thể vào lý do (ví dụ: "matched: refund, flash_sale") để dễ dàng điều chỉnh bộ từ khóa khi cần thiết.
