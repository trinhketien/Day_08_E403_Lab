# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** 2A202600475 - Mai Văn Quân  
**Vai trò trong nhóm:** Eval Owner — Variant & A/B Comparison  
**Ngày nộp:** 2026-04-13  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này?

Tôi đảm nhận vai trò Eval Owner phụ trách phần Variant scorecard và A/B Comparison. Cụ thể, tôi implement hàm `score_context_recall()` (đo retriever có tìm đúng tài liệu expected không — dùng rule-based source matching thay vì LLM) và `score_completeness()` (so sánh câu trả lời với expected answer bằng LLM-as-Judge). Tôi cũng viết hàm `compare_ab()` để so sánh điểm từng câu giữa baseline và variant, tính delta cho 4 metrics, và xuất kết quả ra CSV. Ngoài ra, tôi implement `generate_scorecard_md()` để xuất bảng điểm dạng Markdown có chi tiết per-question notes, và `generate_grading_log()` để tạo file JSON log cho grading. Tôi chạy variant config (Hybrid+Rerank) trên 10 câu, so sánh với dữ liệu baseline từ thành viên 3, và phân tích kết quả per-question.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Tôi hiểu rõ nhất về **A/B Testing trong RAG** — đặc biệt là nguyên tắc "chỉ thay 1 biến mỗi lần". Ban đầu nhóm muốn đổi cùng lúc retrieval mode + chunk size + prompt, nhưng nếu làm vậy sẽ không biết cải tiến đến từ đâu. Kết quả A/B cho thấy Faithfulness tăng +0.40 nhưng Relevance giảm -0.30 — đây là **trade-off** chứ không phải "tốt hơn toàn diện". Nếu thay 3 biến cùng lúc, ta sẽ thấy tổng điểm gần bằng nhau và kết luận "không có cải tiến" — trong khi thực tế Faithfulness đang tốt hơn đáng kể. A/B Testing giúp nhìn rõ từng thay đổi impact metric nào.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Điều ngạc nhiên nhất là Context Recall đạt 5.0/5 ở cả baseline lẫn variant — nghĩa là retriever luôn tìm đúng tài liệu cần thiết, nhưng câu trả lời vẫn có thể sai. Điều này cho thấy lỗi thường nằm ở Generation phase chứ không phải Retrieval — LLM có context đúng nhưng trích xuất thông tin sai hoặc suy diễn ngoài context. Khó khăn kỹ thuật là implement `score_context_recall()`: ban đầu tôi dùng exact string matching để so source name, nhưng test_questions ghi "sla-p1-2026.pdf" còn metadata ghi "support/sla-p1-2026.pdf" → recall = 0. Phải chuyển sang partial matching (tìm basename) mới cho kết quả chính xác.

---

## 4. Phân tích một câu hỏi test (sẽ cập nhật bằng grading_questions sau 17:00)

> **Lưu ý:** Phần dưới đây phân tích câu hỏi từ `test_questions.json` (bộ câu hỏi nội bộ). Sau khi `grading_questions.json` được công bố lúc 17:00, mục này sẽ được cập nhật bằng phân tích câu chính thức.

**Câu hỏi:** q10 — "Nếu cần hoàn tiền khẩn cấp cho khách hàng VIP, quy trình có khác không?"

**Phân tích:**

Đây là câu hỏi kiểm tra khả năng xử lý thông tin **không tồn tại** trong corpus — không có tài liệu nào đề cập đến "hoàn tiền khẩn cấp" hay "khách hàng VIP".

- **Baseline:** Abstain đúng nhưng điểm thấp (9/20). Faithfulness=1 vì judge cho rằng LLM nên trích dẫn policy hoàn tiền thông thường rồi nói "không có quy trình riêng cho VIP" thay vì abstain hoàn toàn.
- **Variant:** Cũng abstain, cùng điểm 9/20. Reranker không giúp ích ở câu này vì vấn đề nằm ở prompt chứ không phải retrieval.
- **Root cause:** Prompt yêu cầu "nếu thiếu context thì abstain" — nhưng câu này có context (policy hoàn tiền thông thường), chỉ thiếu phần VIP riêng. Pipeline cần phân biệt được "hoàn toàn thiếu context" vs "có context bổ trợ nhưng thiếu thông tin cụ thể".
- **Cải thiện:** Sửa prompt thêm hướng dẫn: "Nếu chỉ có thông tin liên quan gián tiếp, hãy trình bày thông tin đó và ghi chú rõ rằng không có quy trình riêng cho trường hợp cụ thể."

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Dựa trên kết quả evaluation, tôi đề xuất hai cải tiến. Thứ nhất, cải thiện grounded prompt để hỗ trợ câu hỏi "partial context" như q10: thay vì chỉ có hai lựa chọn (trả lời hoặc abstain), bổ sung option thứ ba — "trả lời từ thông tin có sẵn và ghi chú rọ không có chính sách riêng cho trường hợp cụ thể". Thứ hai, calibrate LLM-as-Judge bằng cách chấm thủ công 5 câu rồi so sánh với điểm tự động — quan sát cho thấy judge hiện tại đôi khi quá khắt khe với abstain response (q09 Faithfulness=1 trong khi abstain là hành vi đúng).

---

*File: `reports/individual/2A202600475_Mai-Van-Quan.md`*
