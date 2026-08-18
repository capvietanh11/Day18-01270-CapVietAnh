# Individual Reflection — Lab 18

**Tên:** Cáp Việt Anh
**Module phụ trách:** M1–M5 (làm toàn bộ pipeline, một mình)

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** Cả 5 module — M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 RAGAS Evaluation, M5 Enrichment.
- **Các hàm/class chính đã viết:**
  - M1: `chunk_semantic()` (sentence embedding + cosine-similarity boundary), `chunk_hierarchical()` (parent/child với `parent_id` ổn định), `chunk_structure_aware()` (split theo heading H1-H3, giữ nguyên bảng/list trong section).
  - M2: `segment_vietnamese()` (underthesea + bỏ dấu `_`), `BM25Search.index/search`, `DenseSearch.index/search` (Qdrant `query_points`), `reciprocal_rank_fusion()` (cộng theo rank, không cộng score thô).
  - M3: `CrossEncoderReranker._load_model/rerank` (dùng `sentence_transformers.CrossEncoder`, không dùng FlagEmbedding), `FlashrankReranker.rerank` (bonus, dùng `id`/`meta` để giữ metadata qua vòng rerank).
  - M4: `evaluate_ragas()` (try/except an toàn, luôn trả đủ 4 key), `failure_analysis()` (Diagnostic Tree map metric tệ nhất → diagnosis/fix).
  - M5: `summarize_chunk`, `generate_hypothesis_questions`, `contextual_prepend`, `extract_metadata`, `_enrich_single_call` — mỗi hàm có nhánh OpenAI + fallback extractive khi không có API key.
- **Số tests pass:** 37/37 (`pytest tests/ -v`).

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Parent-child hierarchical retrieval — ý tưởng "retrieve bằng chunk nhỏ (chính xác) nhưng generate bằng chunk lớn (đủ ngữ cảnh)" chỉ có giá trị nếu pipeline thực sự nối 2 bước đó lại; tự làm mới thấy `src/pipeline.py` (phần "đã implement sẵn") chỉ dùng child mà bỏ hẳn parent, nên lợi ích lý thuyết không hiện ra trong số liệu thật.
- **Điều bất ngờ nhất:** Production pipeline (hybrid + rerank + enrichment) cho RAGAS score **thấp hơn** naive baseline (dense-only + paragraph chunk) trên 3/4 metric. Trực giác ban đầu là "nhiều kỹ thuật hơn = tốt hơn", nhưng chunk quá nhỏ (256 ký tự) lại cắt đứt một bảng Markdown quan trọng (ngưỡng phê duyệt mua sắm), làm mất thông tin trước khi retrieval kịp tìm thấy — chứng minh chunking sai chiến lược có thể phá hỏng toàn bộ downstream dù M2/M3/M4 đều đúng.
- **Kết nối với bài giảng:** Đúng phần RRF (dùng rank, không dùng score thô vì BM25/cosine không cùng thang đo) và phần Diagnostic Tree cho failure analysis — áp dụng trực tiếp không cần chỉnh sửa gì thêm so với scaffold.

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:** RAGAS chấm `faithfulness = 0.0` cho 2 câu trả lời Yes/No hoàn toàn đúng và có context hỗ trợ rõ ràng (ví dụ "Nhân viên thử việc KHÔNG được nghỉ phép năm" — đúng 100% so với ground truth, context trích rõ câu đó). Ban đầu tưởng đây là lỗi ở M2/M3, đọc lại context/answer thật mới nhận ra là giới hạn của bộ chấm (câu trả lời quá ngắn khiến statement-decomposition của RAGAS thất bại), không phải lỗi retrieval.
- **Cách giải quyết:** Không sửa test hay hard-code số liệu để "che" kết quả xấu. Thay vào đó viết script tạm (`inspect_failures.py`) tái sử dụng lại Qdrant collection đã index (scroll payload ra để build lại BM25 mà không phải gọi lại OpenAI enrichment cho 100 chunk) để lấy đúng answer + context thật của 5 câu tệ nhất, rồi đi theo Error Tree (Answer đúng? → Context đúng? → module nào làm mất thông tin?) để phân biệt lỗi thật (M1 cắt bảng) với lỗi đo lường (RAGAS artifact).
- **Thời gian debug:** Vòng chạy `main.py` đầy đủ mất ~19 phút (chủ yếu do M5 enrichment gọi 1 API call/chunk cho 100 chunk ≈ 7 phút, cộng RAGAS LLM-judge 2×). Việc dựng lại pipeline để soi bottom-5 mất thêm ~5 phút nhờ tái dùng Qdrant thay vì enrich lại từ đầu.

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Route chiến lược chunk theo loại nội dung ngay từ đầu — file có bảng/list (`mua_sam.md`, `bang_luong_2024.md`) dùng `chunk_structure_aware()`, file văn xuôi thuần dùng `chunk_hierarchical()`; đồng thời sửa `run_query()` trong `src/pipeline.py` để sau khi rerank chọn được child tốt nhất, fetch text của **parent** (qua `parent_id`) làm context cho LLM thay vì dùng thẳng child.
- **Module nào muốn thử tiếp:** Query rewriting/decomposition cho câu hỏi multi-hop (ví dụ câu "9 năm thâm niên, lương bao nhiêu" cần join 2 tài liệu) — hiện tại top-3 rerank không đủ ngân sách ngữ cảnh cho loại câu hỏi này.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 5 |
| Code quality | 4 |
| Teamwork | N/A (làm cá nhân) |
| Problem solving | 5 |
