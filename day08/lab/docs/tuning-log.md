# Tuning Log — RAG Pipeline (Day 08 Lab)

> Template: Ghi lại mỗi thay đổi và kết quả quan sát được.
> A/B Rule: Chỉ đổi MỘT biến mỗi lần.

---

## Baseline (Sprint 2)

**Ngày:** ___________  
**Config:**
```
retrieval_mode = "dense"
chunk_size = _____ tokens
overlap = _____ tokens
top_k_search = 10
top_k_select = 3
use_rerank = False
llm_model = _____
```

**Scorecard Baseline:**
| Metric | Average Score |
|--------|--------------|
| Faithfulness | 4.00 /5 |
| Answer Relevance | 4.30 /5 |
| Context Recall | 4.44 /5 |
| Completeness | 3.50 /5 |

**Câu hỏi yếu nhất (điểm thấp):**
> TODO: Liệt kê 2-3 câu hỏi có điểm thấp nhất và lý do tại sao.
> Ví dụ: "q07 (Approval Matrix) - context recall = 1/5 vì dense bỏ lỡ alias."

**Giả thuyết nguyên nhân (Error Tree):**
- [ ] Indexing: Chunking cắt giữa điều khoản
- [ ] Indexing: Metadata thiếu effective_date
- [ ] Retrieval: Dense bỏ lỡ exact keyword / alias
- [ ] Retrieval: Top-k quá ít → thiếu evidence
- [ ] Generation: Prompt không đủ grounding
- [ ] Generation: Context quá dài → lost in the middle

---

## Variant 1 (Sprint 3)

**Ngày:** ___________  
**Biến thay đổi:** ___________  
**Lý do chọn biến này:**
> TODO: Giải thích theo evidence từ baseline results.
> Ví dụ: "Chọn hybrid vì q07 (alias query) và q09 (mã lỗi ERR-403) đều thất bại với dense.
> Corpus có cả ngôn ngữ tự nhiên (policy) lẫn tên riêng/mã lỗi (ticket code, SLA label)."

**Config thay đổi:**
```
retrieval_mode = "hybrid"   # hoặc biến khác
# Các tham số còn lại giữ nguyên như baseline
```

**Scorecard Variant 1 (Hybrid + Rerank):**
| Metric | Baseline | Variant 1 | Delta |
|--------|----------|-----------|-------|
| Faithfulness | 4.00 | 4.10 | +0.10 |
| Answer Relevance | 4.30 | 4.20 | -0.10 |
| Context Recall | 4.44 | 4.44 | 0.00 |
| Completeness | 3.50 | 3.50 | 0.00 |

**Nhận xét:**
> TODO: Variant 1 cải thiện ở câu nào? Tại sao?
> Có câu nào kém hơn không? Tại sao?

**Kết luận:**
> TODO: Variant 1 có tốt hơn baseline không?
> Bằng chứng là gì? (điểm số, câu hỏi cụ thể)

---

## Variant 2 (nếu có thời gian)

**Biến thay đổi:** ___________  
**Config:**
```
# TODO
```

**Scorecard Variant 2:**
| Metric | Baseline | Variant 1 | Variant 2 | Best |
|--------|----------|-----------|-----------|------|
| Faithfulness | ? | ? | ? | ? |
| Answer Relevance | ? | ? | ? | ? |
| Context Recall | ? | ? | ? | ? |
| Completeness | ? | ? | ? | ? |

---

## Tóm tắt học được

1. **Lỗi phổ biến nhất trong pipeline này là gì?**
   Thiếu context trong tài liệu gốc cho các trường hợp ngoại lệ (như q09, q10). LLM đôi khi vẫn cố gắng trả lời thay vì nói "Tôi không biết" mặc dù đã có chỉ dẫn (grounding failure).

2. **Biến nào có tác động lớn nhất tới chất lượng?**
   Kỹ thuật `Hybrid Search` và `Rerank` giúp cải thiện tính chính xác khi trích dẫn, mặc dù điểm số cải thiện không quá nhiều trên tập test nhỏ này nhưng giúp lọc nhiễu tốt hơn.

3. **Nếu có thêm 1 giờ, nhóm sẽ thử gì tiếp theo?**
   Thử nghiệm `Query Expansion` (Cải thiện Context Recall cho các câu hỏi chứa từ viết tắt/tên cũ) và tối ưu hóa Prompt để giảm hiện tượng hallucination khi thiếu dữ liệu.
