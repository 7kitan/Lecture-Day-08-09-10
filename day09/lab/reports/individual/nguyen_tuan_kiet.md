# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Tuấn Kiệt
**Vai trò trong nhóm:**  Worker Owner (synthesis_worker)
**Ngày nộp:** 14/04/2026
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**

- File chính: `workers/synthesis.py`
- Functions tôi implement: `synthesis._call_llm()` (cho OpenAI, Gemini as fallback) và `synthesis.run()`
- Chạy function test từng worker để kiểm tra đối chiếu với implementer

**Cách công việc của tôi kết nối với phần của thành viên khác:**

- Tôi đã làm phần synthesis worker, kết nối với phần retrieval worker và policy tool worker thông qua `supervisor_node`. Synthesis worker có trách nhiệm tổng hợp kết quả từ retrieval worker và policy tool worker để trả lời câu hỏi của user. `synthesis_worker` nhận input từ `supervisor_node`, lấy thông tin trong `AgentState` bao gồm `task`, `retrieved_chunks` (từ retrieval worker), `policy_result` (từ policy tool worker) và trả về output (câu trả lời cuối cùng, tạo ra bằng LLM call) cho `supervisor_node`.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**

- commit `a39033f: phase 5 synthesis` trên branch `main`
- test code cho worker (chạy trong terminal):

```bash
# test retrieval
python -c "
from workers.retrieval import run as retrieval_run
test_state = {"task": "SLA ticket P1 là bao lâu?", "history": []}
result = retrieval_run(test_state)
print(result["retrieved_chunks"])
print(result["worker_io_log"])"

# test policy
python -c "
from workers.policy_tool import run as policy_run
test_state = {"task": "SLA ticket P1 là bao lâu?", "history": []}
result = policy_run(test_state)
print(result["policy_result"])
print(result["worker_io_log"])"

# test synthesis
python -c "
from workers.synthesis import run as synthesis_run
test_state = {"task": "SLA ticket P1 là bao lâu?", "history": []}
result = synthesis_run(test_state)
print(result["answer"])"
```

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Áp dụng cơ chế **Dual-Provider Fallback** (OpenAI ưu tiên, Gemini làm dự phòng) kết hợp với **Strict Type Annotation** (ép kiểu dữ liệu nghiêm ngặt) cho các input và output của worker, dựa trên `basedpyright` type checker.

**Lý do:**

1. **Tính sẵn sàng (Availability):** Trong môi trường lab, việc gọi API thường gặp rủi ro về quota hoặc lỗi xác thực. Fallback cộng thêm in ra lỗi exception giúp pipeline dễ debug hơn.
2. **Tính an toàn của dữ liệu (Type Safety):** Vì `AgentState` là một dictionary dùng chung cho toàn bộ Graph, việc ép kiểu rõ ràng (`list[dict[str, Any]]`) cho các tham số như `retrieved_chunks` giúp phát hiện sớm các lỗi logic nếu worker trước đó trả về sai format, từ đó bảo vệ hàm synthesis khỏi các lỗi crash không đáng có.

**Trade-off đã chấp nhận:**
Việc ép kiểu khiến code dài hơn và tốn thêm thời gian import các module hỗ trợ từ `typing`. Việc `dict` value có thể chứa nhiều kiểu dữ liệu khác nhau khiến việc ép kiểu trở nên phức tạp. Tuy nhiên việc đọc hiểu code và debug sẽ dễ dàng hơn rất nhiều.

**Bằng chứng từ trace/code:**

```python
# Minh chứng cho Strict Typing trong hàm run
def run(state: dict) -> dict:
    task: str = state.get("task", "")
    chunks: list[dict[str, Any]] = state.get("retrieved_chunks", [])
    policy_result: dict[str, Any] = state.get("policy_result", {})

# Minh chứng cho Fallback trong hàm _call_llm
    try:
        # logic OpenAI...
    except Exception as e:
        import traceback
        traceback.print_exc()
        # fallback sang Gemini...
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** `OpenAIError` (Missing API Key) và `NameError` (Missing `Any`).

**Symptom (pipeline làm gì sai?):**
Pipeline bị dừng đột ngột (crash) khi chạy standalone test, trả về lỗi đỏ trong terminal thay vì thông báo lỗi hoặc chuyển sang fallback.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**

1. Do file worker được chạy độc lập, nó không tự động nhận biến môi trường từ `.env` nếu không gọi `load_dotenv()`.
2. Tôi sử dụng kiểu dữ liệu `Any` trong type-hint mà quên không import nó từ module `typing`.

**Cách sửa:**
Tôi đã thêm `from typing import Any` và `from dotenv import load_dotenv` ngay đầu file, đồng thời gọi `load_dotenv()` trước khi khởi tạo client.

**Bằng chứng trước/sau:**

- Trước: Chạy `py workers/synthesis.py` báo lỗi `OpenAIError: The api_key client option must be set...`. và lỗi `type Any is not defined`.
- Sau: Log terminal hiện `OpenAI run complete` và trả về answer có trích dẫn nguồn `[sla_p1_2026.txt]`.

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Tôi rất tập trung vào việc đảm bảo tính đúng đắn của dữ liệu và làm cho code dễ debug hơn, trước khi chạy grading questions. Tôi đã dùng nhiều tool như `basedpyright` để kiểm tra type hint, `traceback` để debug lỗi, và tạo custom command line để test từng phần của pipeline.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Tôi chưa thực sự hiểu rõ về cách hoạt động của `AgentState` và cách nó được truyền giữa các worker. Tôi đã phải đọc lại tài liệu và code của các thành viên khác để hiểu rõ hơn. Cấu trúc dict của `AgentState` khá phức tạp và khó hiểu.

**Nhóm phụ thuộc vào tôi ở đâu?**
Toàn bộ output cuối cùng của hệ thống phụ thuộc vào tôi. Nếu Synthesis worker lỗi, `graph.py` sẽ không thể trả về câu trả lời hoàn thiện cho người dùng.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_
Tôi phụ thuộc vào retrieval worker và policy tool worker để lấy thông tin từ state. Nếu retrieval worker hoặc policy tool worker lỗi, synthesis worker sẽ không thể trả về câu trả lời hoàn thiện cho người dùng.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ dành thời gian annotate type của `AgentState` và các function liên quan. Ví dụ: `list[dict[str, Any]]` có thể được thay thế bằng một dataclass hoặc TypedDict. Việc annotate kĩ cũng có vấn đề ví dụ như `list[dict[str, str | list[str] | dict[str, Any]]]` khá dài và khó đọc. Tôi muốn tìm hiểu thêm về best practice cho type checking của 1 pipeline như thế này. Ngoài type checking thì tôi muốn tìm hiểu thêm về cách automate testing cho pipeline, ví dụ như kiểm tra xem output có chứa các từ khóa quan trọng hay không, hoặc có chứa các trích dẫn nguồn hợp lệ hay không.
