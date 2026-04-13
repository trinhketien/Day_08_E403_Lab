# Results Directory

Thư mục này chứa scorecard kết quả đánh giá pipeline.

## Files bắt buộc nộp:
- `scorecard_baseline.md` — kết quả đánh giá baseline (dense retrieval)
- `scorecard_variant.md` — kết quả đánh giá variant (hybrid / rerank / query transform)

## Cách tạo:
```python
# Trong eval.py, sau khi implement scoring functions:
baseline_results = run_scorecard(BASELINE_CONFIG)
baseline_md = generate_scorecard_summary(baseline_results, "baseline_dense")
(RESULTS_DIR / "scorecard_baseline.md").write_text(baseline_md, encoding="utf-8")
```
