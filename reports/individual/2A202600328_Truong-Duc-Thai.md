# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** 2A202600328 - Trương Đức Thái  
**Vai trò trong nhóm:** Eval Owner — Baseline  
**Ngày nộp:** 2026-04-13  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này?

Tôi đảm nhận vai trò Eval Owner phụ trách phần Baseline trong Sprint 4. Cụ thể, tôi implement các hàm chấm điểm LLM-as-Judge cho 2 metric: `score_faithfulness()` (đánh giá độ trung thực — AI có bịa không) và `score_answer_relevance()` (đánh giá độ liên quan — câu trả lời có đúng trọng tâm câu hỏi không). Mỗi hàm gọi GPT-4o-mini với prompt chuyên biệt, yêu cầu chấm điểm 1-5 và giải thích lý do, output dạng JSON. Tôi cũng viết hàm `_parse_judge_response()` để parse kết quả từ LLM-as-Judge, có xử lý edge case khi LLM trả về format không đúng. Sau đó, tôi chạy `run_scorecard()` với config Dense baseline trên 10 câu hỏi test và xuất kết quả ra `scorecard_baseline.md`. Dữ liệu baseline của tôi là cơ sở để thành viên 4 so sánh A/B.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Concept tôi hiểu rõ nhất là **LLM-as-Judge** — dùng AI để chấm AI. Ban đầu tôi lo rằng GPT-4o-mini chấm chính nó sẽ thiên vị (luôn cho điểm cao). Thực tế kết quả cho thấy judge khá nghiêm khắc: q09 (câu abstain) bị chấm Faithfulness=1 ở baseline vì judge nhận ra "abstain nhưng cách diễn đạt chưa đủ rõ ràng". Tôi cũng hiểu rằng **prompt cho judge phải rất cụ thể** — nếu chỉ hỏi "đánh giá câu trả lời này" thì LLM sẽ cho 4-5 đểm mọi thứ. Phải nêu rõ tiêu chí (1=bịa hoàn toàn, 5=hoàn toàn grounded) và bắt buộc output JSON format thì score mới ổn định.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Khó khăn lớn nhất là thiết kế prompt cho LLM-as-Judge sao cho chấm điểm nhất quán. Lần chạy đầu, tôi dùng prompt tiếng Anh đơn giản "Rate from 1-5", kết quả là judge cho 4-5 điểm gần như tất cả — không phân biệt được câu trả lời tốt và trung bình. Sau khi thêm rubric chi tiết (mô tả cụ thể mỗi mức điểm 1-5) và bắt buộc giải thích lý do trong JSON, phân bố điểm mới hợp lý hơn. Điều ngạc nhiên là q07 (Approval Matrix) bị chấm Faithfulness=1 ở baseline — judge nhận ra câu trả lời "suy diễn" rằng access-control-sop chính là Approval Matrix, trong khi context không hề nói vậy. Đây chính là hallucination tinh vi mà người đọc thường bỏ qua.

---

## 4. Phân tích một câu hỏi test (sẽ cập nhật bằng grading_questions sau 17:00)

> **Lưu ý:** Phần dưới đây phân tích câu hỏi từ `test_questions.json` (bộ câu hỏi nội bộ). Sau khi `grading_questions.json` được công bố lúc 17:00, mục này sẽ được cập nhật bằng phân tích câu chính thức.

**Câu hỏi:** q09 — "ERR-403-AUTH là lỗi gì và cách xử lý?"

**Phân tích:**

Đây là câu hỏi "bẫy" — mã lỗi ERR-403-AUTH không tồn tại trong bất kỳ tài liệu nào trong corpus. Mục đích: test khả năng abstain (từ chối trả lời khi thiếu dữ liệu).

- **Baseline (Dense):** Retriever trả về 3 chunks có score rất thấp (0.35, 0.27, 0.24) — không chunk nào chứa "ERR-403-AUTH". Pipeline trả lời "Không đủ dữ liệu". Tuy nhiên judge chấm Faithfulness=1, Relevance=1. Lý do: baseline vẫn liệt kê 3 sources không liên quan trong response metadata, khiến judge nghi ngờ pipeline đang "cố gắng trả lời nhưng thất bại" thay vì "chủ động abstain".
- **Variant (Hybrid+Rerank):** Cũng abstain, nhưng Faithfulness=5 vì reranker loại hết chunk score thấp → context trống rõ ràng → LLM abstain tự tin hơn.
- **Bài học:** Abstain không chỉ là "nói không biết" — cần abstain một cách rõ ràng, không liệt kê sources nhiễu.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Dựa trên kết quả evaluation, tôi đề xuất bổ sung **confidence threshold** cho retriever: nếu toàn bộ chunks trả về có cosine similarity score dưới 0.4 thì hệ thống tự động abstain mà không cần gọi LLM. Quan sát từ scorecard cho thấy q09 có max retrieval score chỉ đạt 0.35, rõ ràng không có chunk nào thực sự liên quan. Việc phát hiện sớm tại bước retrieval thay vì để LLM tự đoán sẽ giảm chi phí API và loại bỏ rủi ro hallucination từ gốc.

---

*File: `reports/individual/2A202600328_Truong-Duc-Thai.md`*
