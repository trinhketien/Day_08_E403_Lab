# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** 2A202600461 - Đặng Phan Bảo Huy  
**Vai trò trong nhóm:** Documentation Owner  
**Ngày nộp:** 2026-04-13  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này?

Với vai trò Documentation Owner, tôi chịu trách nhiệm viết toàn bộ tài liệu kỹ thuật và báo cáo của nhóm. Cụ thể: (1) Viết `docs/architecture.md` — tài liệu thiết kế pipeline gồm sơ đồ kiến trúc ASCII, chi tiết từng component (Indexing, Retrieval, Generation, Evaluation), và bảng 5 metadata fields; (2) Viết `docs/tuning-log.md` — nhật ký thử nghiệm A/B gồm bảng per-question comparison, phân tích delta 4 metrics, giải thích nguyên nhân tại sao Faithfulness tăng và Relevance giảm; (3) Biên soạn `reports/group_report.md` — báo cáo nhóm tổng kết pipeline, quyết định kỹ thuật, và ước tính điểm grading. Tôi cũng chuẩn bị 5 tài liệu policy trong `data/docs/` với header metadata chuẩn hóa và soạn bộ 10 test questions với expected answers. Công việc tài liệu của tôi đảm bảo nhóm có đầy đủ deliverables cho phần 25% điểm tài liệu kỹ thuật.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Tôi hiểu rõ nhất về **Grounded Prompt Engineering** — cách viết câu lệnh cho AI sao cho nó chỉ trả lời từ tài liệu và không bịa. Trước lab, tôi nghĩ chỉ cần ghi "hãy trả lời dựa trên context" là đủ. Thực tế prompt cần 4 quy tắc bắt buộc: (1) Evidence-only — chỉ dùng bằng chứng từ context; (2) Abstain — biết nói "không biết"; (3) Citation — gắn nhãn [1], [2]; (4) Short/clear — ngắn gọn dễ chấm. Nếu thiếu quy tắc abstain, LLM sẽ "sáng tạo" câu trả lời nghe hợp lý nhưng hoàn toàn bịa đặt. Ví dụ prompt xấu "Hãy trả lời tự tin, dùng kiến thức của bạn để lấp chỗ trống" — đây là lời mời hallucinate. Sự khác biệt giữa prompt tốt và xấu có thể là 5/5 vs 1/5 Faithfulness.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Khó khăn không ngờ nhất là thiết kế test questions. Tôi nghĩ chỉ cần nghĩ 10 câu hỏi bất kỳ, nhưng thực tế cần phân bổ đều: câu dễ (q02, q05 — đáp án rõ ràng trong 1 chunk), câu trung bình (q01, q06 — cần ghép thông tin từ 2+ chunks), câu khó (q07 — alias matching), câu bẫy (q09 — keyword không tồn tại), và câu edge case (q10 — partial context). Nếu 10 câu đều dễ thì scorecard sẽ 5.0/5 hết — không thấy được điểm yếu pipeline. Điều ngạc nhiên là mỗi câu hỏi test cũng cần expected_sources rõ ràng — nếu ghi sai tên file trong test_questions.json, Context Recall sẽ bằng 0 dù retriever tìm đúng tài liệu.

---

## 4. Phân tích một câu hỏi test (sẽ cập nhật bằng grading_questions sau 17:00)

> **Lưu ý:** Phần dưới đây phân tích câu hỏi từ `test_questions.json` (bộ câu hỏi nội bộ). Sau khi `grading_questions.json` được công bố lúc 17:00, mục này sẽ được cập nhật bằng phân tích câu chính thức.

**Câu hỏi:** q04 — "Sản phẩm kỹ thuật số có được hoàn tiền không?"

**Phân tích:**

Câu hỏi này test khả năng tìm đúng **ngoại lệ** trong policy — thông tin nằm ở Điều 3 (Ngoại lệ) chứ không phải Điều 2 (Điều kiện hoàn tiền).

- **Baseline (Dense):** Trả lời hoàn hảo 20/20. Dense search tìm được cả Điều 2 (score 0.67) lẫn Điều 3 (score 0.57), đưa cả hai vào context. LLM trích dẫn chính xác: "Sản phẩm kỹ thuật số không được hoàn tiền, bao gồm license key và subscription [1]".
- **Variant (Hybrid+Rerank):** Cũng 20/20. BM25 boost chunk Điều 3 lên cao hơn vì chứa keyword "kỹ thuật số" khớp với query. Reranker xác nhận chunk này relevance cao nhất.
- **Bài học tài liệu:** Heading-based chunking giữ "Điều 3: Ngoại lệ" thành 1 chunk nguyên vẹn — nếu cắt cứng, ngoại lệ có thể bị cắt đôi và LLM chỉ trả lời "được hoàn tiền" mà bỏ sót phần ngoại lệ. Đây là ví dụ rõ nhất cho thấy chunking strategy ảnh hưởng trực tiếp đến Completeness.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Dựa trên kết quả evaluation, tôi đề xuất hai cải tiến. Thứ nhất, bổ sung **semantic test categories** vào `test_questions.json` — gắn tag cho mỗi câu (factual, exception, negation, alias, abstain) để scorecard có thể phân tích được pipeline yếu ở thể loại câu hỏi nào. Quan sát từ kết quả hiện tại cho thấy pipeline mạnh ở factual questions (q01–q05, điểm 19–20/20) nhưng yếu ở alias (q07, điểm 8–15/20) và partial-context (q10, điểm 9/20). Thứ hai, mở rộng bộ test lên 20 câu để kết quả A/B có ý nghĩa thống kê cao hơn (n=10 hiện tại chưa đủ để kết luận delta ±0.30 là có ý nghĩa).

---

*File: `reports/individual/2A202600461_Dang-Phan-Bao-Huy.md`*
