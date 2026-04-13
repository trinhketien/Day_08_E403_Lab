# Logs Directory

Thư mục này chứa log khi chạy grading_questions.

## File bắt buộc nộp:
- `grading_run.json` — log chạy pipeline với grading_questions.json

## Format bắt buộc:
```json
[
  {
    "id": "gq01",
    "question": "...",
    "answer": "Câu trả lời của pipeline...",
    "sources": ["support/sla-p1-2026.pdf"],
    "chunks_retrieved": 3,
    "retrieval_mode": "hybrid",
    "timestamp": "2026-04-12T17:23:45"
  }
]
```

> Xem script tạo log trong `SCORING.md` hoặc thêm vào `eval.py`.
