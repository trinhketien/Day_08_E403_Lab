# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Trịnh Kế Tiến - 2A202600500
**Vai trò trong nhóm:** Tech Lead  
**Ngày nộp:** 2026-04-13  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này?

Với vai trò Tech Lead, tôi chịu trách nhiệm thiết kế kiến trúc tổng thể pipeline RAG và đưa ra các quyết định kỹ thuật cấp nhóm. Cụ thể, tôi implement toàn bộ Sprint 1 (indexing pipeline): viết hàm `preprocess_document()` để extract 5 metadata fields từ header tài liệu, thiết kế chiến lược heading-based chunking với overlap ~80 tokens, tích hợp Sentence Transformers embedding model (`paraphrase-multilingual-MiniLM-L12-v2`) chạy local, và lưu 29 chunks vào ChromaDB với cosine similarity. Tôi cũng quyết định kiến trúc Singleton pattern cho embedding model để tránh load lại mỗi lần gọi, và chọn cấu trúc prompt grounded 4 quy tắc (evidence-only, abstain, citation, short/clear) làm chuẩn cho cả nhóm. Công việc của tôi tạo nền tảng data layer cho Retrieval Owner xây Sprint 2-3 và Eval Owner chạy Sprint 4.

---

## 2. Điều tôi hiểu rõ hơn sau lab này

Trước lab, tôi biết RAG là "tìm tài liệu rồi đưa cho AI đọc", nhưng chưa hiểu tại sao chunking lại quan trọng đến vậy. Sau khi implement, tôi nhận ra chunking là bottleneck (nút thắt cổ chai) của cả pipeline: nếu cắt cứng theo số ký tự, chunk có thể chứa nửa đầu Điều 2 và nửa cuối Điều 1 — retriever tìm đúng chunk nhưng LLM vẫn trả lời thiếu vì context bị cắt. Heading-based chunking giải quyết vấn đề này bằng cách tôn trọng cấu trúc tự nhiên của tài liệu. Một insight khác: metadata không chỉ để "trang trí" mà thực sự giúp lọc kết quả — ví dụ trường `effective_date` có thể loại bỏ policy đã hết hiệu lực, tránh việc AI trích dẫn thông tin cũ.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn

Điều ngạc nhiên nhất là lỗi `UnicodeEncodeError` trên Windows khi in ký tự ✓ và ✗ ra console. Tôi mất thời gian debug vì lỗi không nằm ở logic code mà ở encoding mặc định `cp1258` của Windows tiếng Việt. Giải pháp đơn giản: set `PYTHONIOENCODING=utf-8` trước khi chạy. Giả thuyết ban đầu của tôi là "Python 3 đã xử lý Unicode tốt rồi" — thực tế terminal Windows vẫn dùng code page legacy. Ngoài ra, tôi cũng ngạc nhiên khi thấy model embedding local (MiniLM-L12) cho kết quả retrieval khá tốt (Context Recall 5.0/5) dù chỉ có 384 dimensions — không nhất thiết phải dùng OpenAI embedding tốn tiền.

---

## 4. Phân tích một câu hỏi test (sẽ cập nhật bằng grading_questions sau 17:00)

> **Lưu ý:** Phần dưới đây phân tích câu hỏi từ `test_questions.json` (bộ câu hỏi nội bộ). Sau khi `grading_questions.json` được công bố lúc 17:00, mục này sẽ được cập nhật bằng phân tích câu chính thức.

**Câu hỏi:** q07 — "Approval Matrix để cấp quyền hệ thống là tài liệu nào?"

**Phân tích:**

Câu hỏi này thú vị vì nó test khả năng xử lý **alias** (tên gọi khác) trong tài liệu. "Approval Matrix" là tên thông dụng mà nhân viên hay gọi, nhưng tài liệu chính thức tên là "Access Control SOP" (`access-control-sop.md`).

- **Baseline (Dense):** Trả lời partial — nhận ra tài liệu `access-control-sop.md` có liên quan nhưng không khẳng định chắc chắn đó chính là "Approval Matrix". Điểm Faithfulness=1 (bị đánh thấp vì judge cho rằng đang suy diễn ngoài context).
- **Variant (Hybrid+Rerank):** Abstain hoàn toàn — Cross-Encoder Reranker cho score thấp vì query "Approval Matrix" không match trực tiếp với content "Access Control SOP". Điểm Relevance=1.
- **Lỗi nằm ở:** Retrieval phase — cả Dense lẫn BM25 đều không tìm được chunk chứa cụm "Approval Matrix" vì tài liệu không có cụm từ này.
- **Cải thiện:** Cần thêm Query Expansion để mở rộng "Approval Matrix" thành các alias — hoặc thêm metadata field `aliases` vào chunk.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì?

Dựa trên kết quả evaluation, tôi đề xuất hai cải tiến cụ thể. Thứ nhất, implement **Query Expansion** bằng cách sử dụng LLM sinh ra 2-3 alias cho mỗi query trước bước retrieval — kết quả scorecard cho thấy q07 thất bại do "Approval Matrix" không ánh xạ được sang "Access Control SOP" trong embedding space. Thứ hai, bổ sung metadata field `aliases` trong hàm `preprocess_document()` để gắn tên gọi thông dụng vào mỗi chunk, giúp BM25 match chính xác hơn khi người dùng sử dụng tên không chính thức.

---

## 6. Điểm cộng (Bonus): Giao diện RAG Web UI

Vì vai trò của tôi là Tech Lead, sau khi hoàn thành backend pipeline theo yêu cầu, tôi đã chủ động thiết kế thêm một **giao diện Web (Chat UI mang phong cách Google Gemini)** ứng dụng cho RAG. Việc có UI giúp demo pipeline System rõ nét và thân thiện hơn là xem log text khô khan trên Terminal. 

Dưới đây là một số hình ảnh demo:

### Giao diện Welcome & Chat
![Chat UI Demo](../../docs/screenshots/01_welcome.png)
![Chat Result](../../docs/screenshots/02_chat_answer.png) 

### Chunk Inspector - Kiểm tra Source
![Chunks Details](../../docs/screenshots/03_chunks_abstain.png)

---

*File: `reports/individual/2A202600500_Trinh-Ke-Tien.md`*
