# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Cá nhân (capvietanh11)
**Thành viên:** capvietanh11 → M1–M5 (làm toàn bộ pipeline)

---

## RAGAS Scores

Chạy `python main.py` trên `test_set.json` (20 câu hỏi), báo cáo tại `reports/naive_baseline_report.json` và `reports/ragas_report.json`.

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8708 | 0.7608 | -0.1100 |
| Answer Relevancy | 0.7594 | 0.7160 | -0.0434 |
| Context Precision | 0.9250 | 0.9250 | +0.0000 |
| Context Recall | 0.9083 | 0.8167 | -0.0917 |

**Điểm bất ngờ:** Production pipeline (hierarchical chunk + M5 enrichment + hybrid search + rerank) thực ra **thấp hơn** naive baseline (dense-only, paragraph chunk) trên 3/4 metric. Đây không phải lỗi ngẫu nhiên — phân tích bottom-5 bên dưới cho thấy nguyên nhân gốc là kiến trúc, không phải bug lặt vặt trong từng module. Không che giấu bằng cách chỉnh test set hay hard-code; giữ nguyên số thật từ lần chạy để phân tích.

## Bottom-5 Failures

Lấy từ `failure_analysis()` trên `reports/ragas_report.json` → `failures[0..4]` (đã sort avg-of-4-metrics tăng dần). Answer/context thật được lấy lại bằng cách chạy lại `run_query()` trên đúng Qdrant collection đã index (không re-enrich, không đổi pipeline) để soi bằng chứng.

### #1
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** "Không tìm thấy." (faithfulness = 0.0)
- **Worst metric:** faithfulness (nhưng nguyên nhân gốc là context_recall)
- **Error Tree:**
  1. Answer đúng ground truth? → Không — model từ chối trả lời.
  2. Context có chứa bằng chứng cần thiết? → Chỉ **một nửa**: 3 context trả về đều nói về số ngày phép (1 đúng v2024 "15+3=18 ngày", 1 lại là **policy v2023 cũ** "5 năm/1 ngày" bị trộn vào top-3), nhưng **không context nào chứa thông tin lương** (bảng lương P3-P4).
  3. Nếu context thiếu → module nào làm mất thông tin? → **M1 + kiến trúc pipeline**: `chunk_hierarchical()` chia nhỏ mỗi child ≤256 ký tự, và câu hỏi này là multi-hop (cần join 2 tài liệu khác nhau: chính sách nghỉ phép + bảng lương). Retriever+rerank top-3 không đủ "ngân sách" để kéo cả 2 chủ đề vào cùng lúc; đồng thời `search()` (M2/M3) không loại chunk v2023 đã bị thay thế dù metadata có version.
- **Root cause:** Multi-hop question vượt quá top-3 context budget; không có cơ chế lọc theo version (v2023 vs v2024) hay theo document type để ưu tiên chính sách hiện hành.
- **Suggested fix:** (a) Tăng `RERANK_TOP_K` cho câu hỏi multi-hop hoặc thêm query decomposition (tách "ngày phép" và "lương" thành 2 sub-query); (b) thêm metadata filter ưu tiên `status=current`/loại bỏ chunk có "đã bị thay thế" trong text. Kiểm tra lại bằng cách thêm câu hỏi này (hoặc biến thể) vào `test_set.json` và re-run `evaluate_ragas` — context_recall của câu này phải ≥ 0.5 (bắt được cả 2 nguồn) thay vì hiện tại thiếu nguồn lương hoàn toàn.

### #2
- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Expected:** KHÔNG. Nhân viên thử việc KHÔNG được nghỉ phép năm. Nếu cần nghỉ, phải xin nghỉ không lương và được trưởng phòng phê duyệt.
- **Got:** "Nhân viên thử việc **KHÔNG được nghỉ phép năm**." (faithfulness = 0.0)
- **Worst metric:** faithfulness
- **Error Tree:**
  1. Answer đúng ground truth? → **Đúng**, khớp hoàn toàn về nội dung.
  2. Context có chứa bằng chứng? → **Có** — context #1 và #2 đều trích rõ câu "Nhân viên thử việc KHÔNG được nghỉ phép năm".
  3. Nếu context đúng nhưng answer bị chấm sai → đây không phải lỗi retrieval/generation mà là **lỗi/giới hạn của bộ chấm RAGAS**: câu trả lời quá ngắn (1 câu phủ định) khiến LLM-judge của RAGAS không tách được đủ "statement" để so khớp với context, nên chấm faithfulness = 0 dù câu trả lời có căn cứ.
- **Root cause:** Đây là **eval-harness artifact**, không phải lỗi của pipeline retrieval/generation. Câu trả lời quá súc tích (câu phủ định ngắn) làm giảm độ chính xác của phép đo faithfulness qua statement-decomposition.
- **Suggested fix:** Yêu cầu prompt sinh câu trả lời đầy đủ hơn (ví dụ luôn diễn giải lại 1 câu lý do, không chỉ trả lời cụt) để RAGAS decompose statement chính xác hơn; đối chiếu lại bằng cách chạy lại `evaluate_ragas` trên chính câu hỏi này sau khi sửa prompt và kỳ vọng faithfulness tăng dù nội dung answer không đổi nhiều.

### #3
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Got:** "Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI." (faithfulness = 0.0)
- **Worst metric:** faithfulness
- **Error Tree:** Giống hệt #2 — answer đúng, context đúng (context #1 trích rõ "chưa được hưởng gói bảo hiểm sức khỏe PVI"), nhưng bị chấm faithfulness = 0.
- **Root cause:** Cùng eval-harness artifact như #2 (câu trả lời ngắn/phủ định).
- **Suggested fix:** Giống #2 — mở rộng câu trả lời cho các câu hỏi Yes/No để RAGAS statement-decomposition hoạt động ổn định hơn; validate bằng cách so sánh faithfulness trước/sau trên cùng 2 câu hỏi #2, #3.

### #4
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** "Nhân viên sẽ bị phạt 2% trên số tiền 15 triệu VNĐ cho mỗi tháng... Phí bị phạt = 15.000.000 × 2% = 300.000 VNĐ." (faithfulness = 0.25)
- **Worst metric:** faithfulness
- **Error Tree:**
  1. Answer đúng ground truth? → **Sai** — model tính phạt = 300.000 VNĐ (áp nguyên 2% lên toàn bộ khoản tạm ứng), trong khi đáp án đúng là pro-rata theo **5 ngày quá hạn** (~50.000 VNĐ), không phải nguyên 1 tháng.
  2. Context có chứa bằng chứng? → **Có đủ**: context #1 nêu rõ "thanh toán sau 15 ngày sẽ bị tính phí 2%/tháng", context #2 nêu "thời hạn thanh toán 15 ngày". Đủ dữ kiện để tính đúng nếu suy luận đúng (20 - 15 = 5 ngày quá hạn, pro-rata trong tháng).
  3. Context đúng nhưng answer sai → đây là lỗi **generation/reasoning**, không phải retrieval.
- **Root cause:** System prompt hiện tại ("Trả lời CHỈ dựa trên context") không yêu cầu model làm rõ bước tính pro-rata theo số ngày quá hạn; gpt-4o-mini nhân thẳng 2% × 15tr thay vì tính phần trăm theo tỉ lệ 5/30 ngày.
- **Suggested fix:** Thêm hướng dẫn tính toán tường minh trong prompt (ví dụ: "Nếu câu hỏi có tính phí theo %/tháng nhưng thời gian trễ hạn nhỏ hơn 1 tháng, PHẢI tính pro-rata theo số ngày trễ") hoặc few-shot ví dụ tính phí. Kiểm tra lại bằng cách thêm test case tính phí pro-rata riêng và so kết quả số học (50.000 VNĐ) sau khi sửa prompt.

### #5
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** "Cần thêm phê duyệt của Kế toán trưởng." (context_recall = 0.0)
- **Worst metric:** context_recall
- **Error Tree:**
  1. Answer đúng ground truth? → **Sai** hoàn toàn — không nhắc đến CEO.
  2. Context có chứa bằng chứng cần thiết? → **Không** — context #3 trích đúng file `mua_sam.md` nhưng bảng thẩm quyền phê duyệt bị **cắt ngang giữa bảng** ("Dưới 5.000.000 VNĐ | Trưởng phòng (Manager) |" rồi hết, thiếu hẳn dòng ">50.000.000 VNĐ → CEO"). Context #2 lại là quy tắc "Kế toán trưởng" từ **tài liệu tài chính khác** (không phải mua sắm thiết bị), khiến model nhầm.
  3. Context thiếu → module nào làm mất thông tin? → **M1 chunking**: `chunk_hierarchical()` cắt child theo ký tự (≤256) đúng giữa bảng Markdown, làm mất dòng quan trọng nhất (ngưỡng CEO). Đây đúng là rủi ro mà spec M1 đã cảnh báo ("Không vô tình chặt bảng... giữa cấu trúc") — bảng bị chunk_hierarchical chặt đứt vì nó không "structure-aware" như `chunk_structure_aware()`.
- **Root cause:** Pipeline sản xuất (`src/pipeline.py`) dùng `chunk_hierarchical()` cho toàn bộ tài liệu (kể cả các file có bảng markdown), không dùng `chunk_structure_aware()` để giữ bảng nguyên vẹn, nên bảng thẩm quyền phê duyệt bị chặt ngang.
- **Suggested fix:** Với tài liệu có bảng (`mua_sam.md`, `bang_luong_2024.md`, ...), ưu tiên `chunk_structure_aware()` (giữ nguyên bảng theo section) trước khi hierarchical-split phần còn lại; hoặc tăng `HIERARCHICAL_CHILD_SIZE` đủ lớn để chứa trọn bảng nhỏ. Kiểm tra lại bằng cách chạy `chunk_structure_aware()` trên `mua_sam.md`, xác nhận cả 4 dòng bảng phê duyệt nằm trong cùng 1 chunk, rồi re-run câu hỏi #5 — context_recall kỳ vọng lên ≥ 0.5.

## Case Study (cho presentation)

**Question chọn phân tích:** #5 — "Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?"

**Error Tree walkthrough:**
1. Output đúng? → Không, trả lời "Kế toán trưởng" thay vì "CEO".
2. Context đúng? → Không — bảng phê duyệt bị M1's `chunk_hierarchical()` chặt đứt giữa chừng, dòng ">50 triệu → CEO" biến mất khỏi mọi chunk được index.
3. Query rewrite OK? → Không cần rewrite, câu hỏi đã rõ ràng ("thiết bị 55 triệu, ai phê duyệt") — vấn đề không nằm ở M2 retrieval mà nằm ở M1 chunking làm mất dữ liệu trước khi retrieval có cơ hội tìm thấy nó.
4. Fix ở bước: **M1 (chunking)** — đổi chiến lược chunk cho tài liệu có bảng sang `chunk_structure_aware()`, hoặc tăng `child_size` để bảng không bị cắt.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Route theo loại nội dung: file có bảng/list → `chunk_structure_aware()`; file văn xuôi thuần → `chunk_hierarchical()` (dùng đúng thiết kế "child để match, parent để trả context" — hiện `src/pipeline.py` build cả parent lẫn child nhưng chỉ index/trả về child, bỏ phí lợi ích parent-context của M1).
- Thêm metadata filter theo `status`/version (loại chunk "đã bị thay thế") để câu hỏi #1 không bị lẫn giữa v2023/v2024.
- Sửa prompt sinh câu trả lời để có bước tính toán tường minh cho câu hỏi số học (giải quyết #4), và yêu cầu câu trả lời đủ dài để RAGAS statement-decomposition không chấm nhầm 0 điểm cho câu đúng (giải quyết #2, #3).
