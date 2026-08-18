# Group Report — Lab 18: Production RAG

**Nhóm:** Cá nhân (capvietanh11)
**Ngày:** 2026-08-18

## Thành viên & Phân công

Bài làm cá nhân — 1 người phụ trách toàn bộ M1–M5.

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| capvietanh11 | M1: Chunking (semantic/hierarchical/structure-aware) | ☑ | 13/13 |
| capvietanh11 | M2: Hybrid Search (BM25 + Dense + RRF) | ☑ | 5/5 |
| capvietanh11 | M3: Reranking (CrossEncoder + Flashrank) | ☑ | 5/5 |
| capvietanh11 | M4: Evaluation (RAGAS + failure analysis) | ☑ | 4/4 |
| capvietanh11 | M5: Enrichment (summary/HyQA/contextual/metadata) | ☑ | 10/10 |

Tổng: 37/37 tests pass (`pytest tests/ -v`).

## Kết quả RAGAS

Từ `reports/naive_baseline_report.json` (20 câu hỏi) và `reports/ragas_report.json` (20 câu hỏi), sinh bởi `python main.py`.

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.8708 | 0.7608 | -0.1100 |
| Answer Relevancy | 0.7594 | 0.7160 | -0.0434 |
| Context Precision | 0.9250 | 0.9250 | +0.0000 |
| Context Recall | 0.9083 | 0.8167 | -0.0917 |

## Key Findings

1. **Biggest improvement:** Context Precision giữ nguyên ở mức cao (0.925) dù chunk nhỏ hơn nhiều (child 256 ký tự so với paragraph ~500 ở baseline) — cross-encoder rerank (M3) lọc nhiễu hiệu quả, không có chunk rác lọt vào top-3 dù index có 100 chunk từ hybrid search thay vì retrieval dense-only đơn giản.
2. **Biggest challenge:** Production **thấp hơn** naive baseline trên 3/4 metric — không phải do M2/M3 sai, mà do **M1 chunking quá nhỏ (256 ký tự) cắt đứt bảng Markdown** (xem case #5 trong `failure_analysis.md`: bảng thẩm quyền phê duyệt bị chặt ngang, mất dòng "> 50 triệu → CEO") và **kiến trúc pipeline không tận dụng parent-child**: `src/pipeline.py::build_pipeline()` tạo cả `parents` lẫn `children` từ `chunk_hierarchical()` nhưng chỉ index/trả về `children` — bỏ phí đúng lợi ích "child để match chính xác, parent để trả context đầy đủ" mà M1 được thiết kế để cung cấp.
3. **Surprise finding:** 2/5 câu trong bottom-5 (câu #2, #3) thực ra **answer đúng và context đúng**, nhưng bị RAGAS chấm faithfulness = 0.0 — đây là giới hạn của bộ đánh giá (câu trả lời Yes/No quá ngắn khiến LLM-judge không tách được statement để so khớp), không phải lỗi pipeline. Bài học: không nên tin tuyệt đối vào 1 con số RAGAS mà phải đọc lại context/answer thật trước khi kết luận nguyên nhân.

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):** Naive (paragraph chunk + dense-only) đạt faithfulness 0.87 / recall 0.91; Production (hierarchical + hybrid + rerank + enrichment) lại thấp hơn ở 3/4 metric (0.76 / 0.82) — kết quả phản trực giác so với kỳ vọng "production luôn tốt hơn baseline".
2. **Biggest win — module nào, tại sao:** M3 (reranking) — context_precision giữ nguyên 0.925 dù input có nhiều chunk nhiễu hơn (hybrid search trả nhiều ứng viên hơn dense-only), chứng tỏ cross-encoder lọc đúng.
3. **Case study — 1 failure, Error Tree walkthrough:** Câu "Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?" → answer sai (Kế toán trưởng thay vì CEO) → context sai (bảng phê duyệt bị M1 chunk_hierarchical cắt đứt, mất dòng CEO) → root cause ở M1, không phải M2/M3. Chi tiết đầy đủ trong `analysis/failure_analysis.md`.
4. **Next optimization nếu có thêm 1 giờ:** (a) Route file có bảng/list sang `chunk_structure_aware()` thay vì `chunk_hierarchical()`; (b) sửa `src/pipeline.py` để retrieval dùng child nhưng generation dùng parent text (đúng thiết kế parent-child); (c) thêm metadata filter loại bỏ chunk chính sách đã bị thay thế (v2023) khi có bản v2024.
