# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** 2A202600364 - Đoàn Thư Ánh  
**Vai trò trong nhóm:** Retrieval Owner  
**Ngày nộp:** 2026-04-13  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này?

Với vai trò Retrieval Owner, tôi chịu trách nhiệm Sprint 2 và Sprint 3 — xây dựng toàn bộ retrieval pipeline và grounded answer function. Cụ thể, ở Sprint 2 tôi implement hàm `retrieve_dense()` sử dụng ChromaDB cosine similarity search để tìm top-10 chunks gần nhất với query, và hàm `call_llm()` tích hợp OpenAI GPT-4o-mini với temperature=0. Ở Sprint 3, tôi implement thêm `retrieve_sparse()` dùng BM25Okapi cho keyword matching, `retrieve_hybrid()` gộp kết quả Dense + BM25 bằng thuật toán RRF (Reciprocal Rank Fusion) với dense_weight=0.6 và sparse_weight=0.4, và `rerank()` dùng Cross-Encoder (`ms-marco-MiniLM-L-6-v2`) để lọc từ top-10 xuống top-3. Công việc của tôi kết nối trực tiếp với index data từ Tech Lead (Sprint 1) và cung cấp hàm `rag_answer()` cho Eval Owner chạy scorecard.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Trước lab tôi nghĩ Dense retrieval (tìm kiếm theo nghĩa) luôn tốt hơn BM25 (tìm theo từ khóa) vì "hiểu ngữ nghĩa". Thực tế cho thấy Dense thất bại khi query chứa keyword chuyên ngành: hỏi "ERR-403-AUTH" thì Dense chỉ trả score 0.35 (rất thấp) vì embedding model không biểu diễn tốt mã lỗi. BM25 ngược lại sẽ tìm chính xác chuỗi ký tự. Hybrid kết hợp cả hai là giải pháp tối ưu — RRF gộp kết quả dựa trên thứ hạng chứ không phải score tuyệt đối, nên hai hệ thống có thang điểm khác nhau vẫn merge được. Tôi cũng hiểu rõ hơn vai trò của Reranker: nó đóng vai trò "phễu" — search rộng top-10 để không bỏ sót, rồi lọc chính xác xuống top-3 để prompt không bị nhiễu.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Khó khăn lớn nhất là thiết kế hàm `build_context_block()` — cách đóng gói chunks vào prompt ảnh hưởng rất lớn đến chất lượng câu trả lời. Ban đầu tôi chỉ nối các chunk lại thành 1 block text dài, kết quả là LLM hay lẫn thông tin giữa các nguồn khác nhau. Sau khi thêm format rõ ràng `[1] source | section | score` cho mỗi chunk, LLM citation chính xác hơn nhiều. Điều ngạc nhiên là hiệu ứng "Lost in the Middle" thực sự xảy ra: khi context dài, LLM bỏ qua chunk ở giữa và chỉ dùng chunk đầu tiên. Giải pháp đơn giản là giới hạn top-3 chunks thay vì top-5.

---

## 4. Phân tích một câu hỏi test (sẽ cập nhật bằng grading_questions sau 17:00)

> **Lưu ý:** Phần dưới đây phân tích câu hỏi từ `test_questions.json` (bộ câu hỏi nội bộ). Sau khi `grading_questions.json` được công bố lúc 17:00, mục này sẽ được cập nhật bằng phân tích câu chính thức.

**Câu hỏi:** q01 — "SLA xử lý ticket P1 là bao lâu?"

**Phân tích:**

- **Baseline (Dense):** Trả lời đúng "15 phút phản hồi ban đầu, 4 giờ xử lý" với citation [1] từ `sla-p1-2026.pdf`. Điểm 19/20 — chỉ trừ nhẹ ở Completeness vì chưa nhắc chi tiết escalation 10 phút.
- **Variant (Hybrid+Rerank):** Trả lời hoàn hảo 20/20. Reranker đẩy chunk chứa bảng SLA chi tiết lên top-1 (thay vì chunk "Công cụ và kênh liên lạc" chiếm vị trí 2 trong baseline). Kết quả là LLM có context chính xác hơn.
- **Lỗi gốc ở baseline:** Retrieval phase — Dense search xếp chunk "Công cụ và kênh liên lạc" (score 0.53) cao hơn chunk chứa bảng SLA chi tiết vì embedding model đặt "P1" gần các chunk khác cũng nhắc P1.
- **Reranker sửa lỗi này** vì Cross-Encoder đọc cả (query + chunk) cùng lúc, nhận ra chunk SLA chi tiết thực sự trả lời câu hỏi tốt hơn.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Dựa trên kết quả evaluation, tôi đề xuất hai cải tiến. Thứ nhất, áp dụng **MMR (Maximal Marginal Relevance)** thay cho top-K selection hiện tại — quan sát từ scorecard cho thấy một số câu hỏi có 3 chunks trùng lặp nội dung (cùng đề cập một điều khoản), làm giảm Completeness. MMR sẽ ưu tiên chunk vừa liên quan vừa đa dạng thông tin. Thứ hai, thử nghiệm điều chỉnh tỷ lệ `dense_weight/sparse_weight` trong RRF từ 0.6/0.4 thành 0.5/0.5 để đánh giá ảnh hưởng của BM25 trong corpus có nhiều thuật ngữ chuyên ngành.

---

*File: `reports/individual/2A202600364_Doan-Thu-Anh.md`*
