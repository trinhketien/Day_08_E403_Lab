# Group Report — Lab Day 08: RAG Pipeline

**Tên nhóm:** Nhóm 63  
**Thành viên:**
- 2A202600500 - Trịnh Kế Tiến (Tech Lead)
- 2A202600364 - Đoàn Thư Ánh (Retrieval Owner)
- 2A202600328 - Trương Đức Thái (Eval Owner — Baseline)
- 2A202600475 - Mai Văn Quân (Eval Owner — Variant & A/B)
- 2A202600461 - Đặng Phan Bảo Huy (Documentation Owner)

**Ngày nộp:** 2026-04-13

---

## 1. Tóm tắt pipeline nhóm

Pipeline RAG của nhóm xây dựng hệ thống trợ lý nội bộ trả lời câu hỏi chính sách doanh nghiệp dựa trên 5 tài liệu (hoàn tiền, SLA sự cố, quy trình cấp quyền, IT Helpdesk FAQ, chính sách nghỉ phép). Hệ thống thực hiện indexing 29 chunks với metadata 5 trường vào ChromaDB, hỗ trợ Dense/Hybrid/Rerank retrieval, grounded generation bằng GPT-4o-mini với citation [1][2] và khả năng abstain khi lack context.

---

## 2. Quyết định kỹ thuật cấp nhóm

### Chunking
Nhóm sử dụng **heading-based chunking**: cắt tài liệu theo ranh giới `=== Section ===` tự nhiên, sau đó split tiếp theo paragraph (`\n\n`) nếu section quá dài. Overlap ~80 tokens giữa các chunk để không mất ngữ cảnh. Kết quả: 29 chunks từ 5 tài liệu, mỗi chunk giữ đầy đủ 5 metadata fields (source, department, effective_date, access, section). Chiến lược này tốt hơn cắt cứng vì không cắt giữa điều khoản.

### Retrieval Variant
Nhóm chọn **Hybrid (Dense + BM25/RRF) + Cross-Encoder Rerank** vì:
- Baseline Dense thường bỏ lỡ keyword chuyên ngành (P1, mã lỗi)
- BM25 bổ sung khả năng exact keyword matching
- Cross-Encoder lọc bỏ chunk nhiễu trước khi đưa vào prompt

A/B comparison cho thấy: Faithfulness tăng +0.40 (ít bịa hơn), đổi lại Relevance giảm -0.30 (abstain más nhiều). Trade-off hợp lý cho môi trường production.

### Grounding & Abstain
Pipeline xử lý câu hỏi ngoài phạm vi docs bằng grounded prompt 4 quy tắc: evidence-only, abstain, citation, short/clear. Ví dụ cụ thể:
- **q09** "ERR-403-AUTH là lỗi gì?" → Pipeline trả lời "Không đủ dữ liệu trong tài liệu hiện có" (đúng — mã lỗi này không tồn tại trong docs).
- **q10** "Hoàn tiền VIP khác không?" → Abstain đúng (không có policy VIP riêng trong docs).

---

## 3. Kết quả chạy Test Questions (nội bộ)

> **Lưu ý:** Bảng dưới đây là kết quả chạy `test_questions.json` (10 câu nội bộ do nhóm tự thiết kế). Kết quả chạy `grading_questions.json` (công bố 17:00) sẽ được cập nhật vào `logs/grading_run.json` và bổ sung vào mục này.

| ID | Câu hỏi (rút gọn) | Kết quả | Ghi chú |
|---|---|---|---|
| q01 | SLA P1 bao lâu? | **Full** | 15 phút + 4 giờ — trích dẫn từ `sla-p1-2026.pdf` [1] |
| q02 | Hoàn tiền mấy ngày? | **Full** | 7 ngày làm việc — trích dẫn từ `refund-v4.pdf` [1] |
| q03 | Phê duyệt Level 3? | **Full** | LM + IT Admin + IT Security — trích dẫn `access-control-sop` [2] |
| q04 | Kỹ thuật số hoàn tiền? | **Full** | Không — ngoại lệ license key, subscription |
| q05 | Khóa bao nhiêu lần? | **Full** | 5 lần — trích dẫn `helpdesk-faq.md` [1] |
| q06 | Escalation P1? | **Partial** | Đúng 10 phút escalation, thiếu chi tiết VP/CTO chain |
| q07 | Approval Matrix? | **Partial** | Baseline nhận diện đúng source nhưng suy diễn; Variant abstain |
| q08 | Remote mấy ngày? | **Full** | 2 ngày/tuần, phê duyệt qua HR Portal |
| q09 | ERR-403-AUTH? | **Full (Abstain)** | Abstain đúng — mã lỗi không tồn tại trong corpus |
| q10 | VIP hoàn tiền khác? | **Full (Abstain)** | Abstain đúng — không có policy VIP riêng |

**Điểm test nội bộ ước tính:** 8 Full (8×10) + 2 Partial (2×5) = 90/100

**Điểm grading chính thức:** _Chờ cập nhật sau 17:00 khi nhận được `grading_questions.json`_

---

## 4. Điều nhóm học được

1. **Chunking strategy quyết định chất lượng retrieval.** Heading-based chunking giữ nguyên ngữ cảnh điều khoản, không cắt giữa câu như fixed-size chunking. Kết quả: Context Recall đạt 5.0/5 trên toàn bộ 10 câu hỏi — chứng tỏ retriever luôn tìm đúng tài liệu cần thiết.

2. **Hybrid+Rerank tạo trade-off giữa Faithfulness và Completeness.** Faithfulness tăng +0.40 (giảm hallucination) nhưng Relevance giảm -0.30 (abstain nhiều hơn). Trong môi trường sản xuất, nhóm thống nhất ưu tiên không bịa hơn là trả lời đầy đủ.

3. **Evaluation bằng số liệu là bắt buộc, không thể vibe check.** Nếu không có scorecard A/B, nhóm sẽ không phát hiện q07 bị Reranker loại nhầm chunk bổ trợ — vấn đề chỉ hiện ra khi so sánh baseline vs variant trên nhiều câu.

---

*File này được nộp cùng với 5 báo cáo cá nhân.*
