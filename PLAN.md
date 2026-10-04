# PLAN.md — Lộ trình hoàn thành Lab 18: Production RAG Pipeline

Nguồn: README.md + ASSIGNMENT.md + RUBRIC.md + hiện trạng code (19 TODO còn lại trong `src/m1..m5`, chưa có `reports/`, chưa có reflection cá nhân).

Tổng điểm: 100 + 10 bonus. Tick `[x]` khi hoàn thành từng bước.

---

## 0. Setup môi trường (10 phút)

- [ ] `docker compose up -d` — khởi động Qdrant
- [ ] `python -m venv .venv` + activate
- [ ] `pip install -r requirements.txt`
- [ ] `Copy-Item .env.example .env` (hoặc `cp`) → điền `OPENAI_API_KEY`
- [ ] Pre-download models (MiniLM, bge-m3, bge-reranker-v2-m3)
- [ ] `python naive_baseline.py` chạy thành công

## 1. Module 1 — Chunking (`src/m1_chunking.py`, 20 phút, 12đ)

- [ ] `chunk_semantic()` — nhóm câu theo cosine similarity, trả `list[Chunk]` không rỗng
- [ ] `chunk_hierarchical()` — parent (2048) + child (256), children có `parent_id` hợp lệ
- [ ] `chunk_structure_aware()` — parse markdown headers, giữ `section` trong metadata
- [ ] `pytest tests/test_m1.py -v` pass 100%

## 2. Module 2 — Hybrid Search (`src/m2_search.py`, 20 phút, 12đ)

- [ ] `segment_vietnamese()` — underthesea + thay `_`
- [ ] `BM25Search.index()` + `.search()` — method="bm25"
- [ ] `DenseSearch.index()` + `.search()` — bge-m3 + Qdrant `query_points()`
- [ ] `reciprocal_rank_fusion()` — method="hybrid", score = Σ 1/(k+rank+1)
- [ ] Query "nghỉ phép" trả kết quả liên quan
- [ ] `pytest tests/test_m2.py -v` pass 100%

## 3. Module 3 — Reranking (`src/m3_rerank.py`, 15 phút, 12đ)

- [ ] `CrossEncoderReranker._load_model()` — load bge-reranker-v2-m3
- [ ] `.rerank()` — trả ≤3 `RerankResult`, sort theo `rerank_score` giảm dần
- [ ] Kiểm tra: doc "nghỉ phép" rank cao hơn "VPN"
- [ ] `pytest tests/test_m3.py -v` pass 100%

## 4. Module 4 — RAGAS Eval (`src/m4_eval.py`, 15 phút, 12đ)

- [ ] `evaluate_ragas()` — dict 4 metric keys, wrap try/except
- [ ] `failure_analysis()` — Bottom-N, trả `diagnosis` + `suggested_fix`
- [ ] `pytest tests/test_m4.py -v` pass 100%

## 5. Module 5 — Enrichment (`src/m5_enrichment.py`, 20 phút, 12đ)

- [ ] Chọn mode: Combined `_enrich_single_call()` (khuyến khích, +2 bonus) hoặc 4 hàm riêng
- [ ] `enrich_chunks()` trả `list[EnrichedChunk]`
- [ ] `enriched_text` khác `original_text` khi có API key
- [ ] Fallback hoạt động khi không có API key
- [ ] `pytest tests/test_m5.py -v` pass 100%

## 6. Chạy Pipeline & Evaluation (20 phút, 25đ)

- [ ] `python src/pipeline.py` chạy end-to-end, exit code 0 (10đ)
- [ ] Kiểm tra `reports/ragas_report.json` sinh ra, ≥1 metric ≥0.70 (tốt nhất ≥3) (10đ)
- [ ] Điền bảng so sánh Naive vs Production (Faithfulness, Answer Relevancy, Context Precision, Context Recall) vào ASSIGNMENT.md hoặc ghi chú riêng
- [ ] Mở `reports/ragas_report.json` → tìm bottom-5 worst questions
- [ ] Viết `analysis/failure_analysis.md`: mỗi failure có diagnosis + suggested fix + Error/Diagnostic Tree (5đ)

## 7. Reflection (30 phút, 15đ)

- [ ] Copy `analysis/reflections/reflection_TEMPLATE.md` → `analysis/reflections/reflection_TruongHoangThanhAn.md`
- [ ] Phần 1: Bảng mapping 5 modules (lecture concept → hàm cụ thể → observation) (5đ)
- [ ] Phần 2: Khó khăn gặp phải — lỗi cụ thể + cách debug + kiến thức thiếu (5đ)
- [ ] Phần 3: Action Plan cho project cá nhân — chunking/search/rerank/eval/enrichment + timeline (5đ)

## 8. Bonus (+10 max, tùy chọn)

- [ ] RAGAS Faithfulness ≥ 0.85 (+3)
- [ ] RAGAS tất cả metrics ≥ 0.75 (+3)
- [ ] Enrichment combined mode `_enrich_single_call()` (+2)
- [ ] Latency breakdown report — bảng thời gian từng bước (+2)

## 9. Kiểm tra trước khi nộp

- [ ] `pytest tests/ -v` — 100% pass
- [ ] `ruff check src/` — không lỗi nghiêm trọng
- [ ] Đếm TODO còn lại = 0: `(Select-String -Path src/*.py -Pattern "# TODO").Count`
- [ ] `python main.py` chạy end-to-end, sinh `reports/ragas_report.json`
- [ ] `python check_lab.py` pass toàn bộ

## 10. Nộp bài

- [ ] Đổi tên repo đúng chuẩn: `K4-Track3A-DAY18-TruongHoangThanhAn-2A202602574-ProductionRAG` (đã đúng)
- [ ] Repo để **Public**
- [ ] `git add` + commit + push lên GitHub
- [ ] Nộp link repo lên VLearn LMS trước **23h59 ngày diễn ra lab**
