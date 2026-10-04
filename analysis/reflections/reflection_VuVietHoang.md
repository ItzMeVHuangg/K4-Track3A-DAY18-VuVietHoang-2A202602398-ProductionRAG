# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Vũ Việt Hoàng (2A202602398)  
**Khóa:** K4 - Track 3A  
**Ngày hoàn thành:** 2026-10-04

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích |
|----------------|--------|-------------|--------------------------|
| Semantic chunking | M1 | `chunk_semantic()` | Tách theo cosine similarity giữa câu liền kề (MiniLM, ngưỡng 0.85). Ngưỡng cao → nhiều chunk nhỏ; cần chỉnh theo corpus. Model được cache ở mức module để không load lại mỗi lần gọi. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Parent 2048 / child 256; corpus cho 112 child chunk. Child dùng để retrieve (chính xác), `parent_id` giữ lại để mở rộng context. Câu quá dài bị cắt cứng ở 256 ký tự. |
| Structure-aware chunking | M1 | `chunk_structure_aware()` | Tách theo header `#`–`###`, lưu `section` vào metadata; phù hợp với các file policy markdown. |
| BM25 + Dense fusion | M2 | `segment_vietnamese()`, `reciprocal_rank_fusion()` | Tách từ bằng underthesea rồi thay `_` bằng khoảng trắng để query "nghỉ phép" khớp với corpus. RRF chỉ dùng thứ hạng nên không cần chuẩn hóa điểm BM25 và cosine. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | bge-reranker-v2-m3 chấm lại top-20 → top-3; test xác nhận doc "nghỉ phép" xếp trên doc "VPN". Đổi lại là latency cao hơn rõ rệt trên CPU. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()`, `failure_analysis()` | Judge = Gemini. Production: faith 0.51, relevancy 0.67, precision 0.59, recall 0.72 (naive: 0.42/0.57/0.45/0.75). Job lỗi bị tính 0.0 nên điểm có thể thấp hơn thực tế. |
| Contextual embeddings | M5 | `contextual_prepend()` / `_enrich_single_call()` | Combined mode: 1 call/chunk lấy summary + câu hỏi + context + metadata. Không có key thì dùng fallback trích xuất để pipeline vẫn chạy. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

- **Lỗi kỹ thuật gặp phải (Exact error message):**
  - `pip._vendor.urllib3.exceptions.ReadTimeoutError: HTTPSConnectionPool(host='files.pythonhosted.org', port=443): Read timed out.` khi cài torch/sentence-transformers.
  - `404 models/gemini-2.5-flash-lite is no longer available to new users`; `429 RESOURCE_EXHAUSTED ... free_tier_requests, limit: 15` khi chạy RAGAS; `Missing or invalid Authorization header` do key đặt sai tên biến.
  - `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine` — Docker Desktop chưa chạy.
- **Nguyên nhân gốc rễ & Cách debug:**
  - Lỗi mạng: gói lớn tải chậm → chạy lại với `--default-timeout=120 --retries 10`.
  - Model cũ bị gỡ → đổi sang `gemini-3.5-flash-lite`; key đặt sai tên → đổi thành `GEMINI_API_KEY`; rate limit free tier làm một số job RAGAS tính 0.0 → giảm `max_workers` hoặc dùng key quota cao hơn.
  - Docker: `DenseSearch` đã có fallback sang Qdrant in-memory nên không chặn việc làm lab.
- **Kiến thức còn thiếu & Cách khắc phục:**
  - Cách chọn ngưỡng semantic chunking và kích thước child/parent hợp lý cho tiếng Việt — cần thử nhiều giá trị và đo bằng RAGAS (chưa làm được vì thiếu key).
  - Thời gian chạy trên CPU khá lâu (37 test mất ~23 phút, pipeline ~6 phút do phải tải/nạp bge-m3 và reranker); cần cân nhắc cache index.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Chatbot hỏi đáp tài liệu nội bộ (tiếng Việt)

#### 1. Hiện trạng
- **Pipeline hiện tại:** Chunk theo đoạn + dense search đơn thuần (giống naive baseline).
- **Vấn đề / Bottlenecks đang gặp:** Từ khóa chính xác (mã, số tiền, tên chính sách) bị bỏ sót; câu hỏi nhiều bước thiếu context; tài liệu nhiều phiên bản gây nhiễu.

#### 2. Kế hoạch cải tiến
1. **Chunking strategy:** Hierarchical (child để retrieve, parent để trả lời) — giữ chính xác mà vẫn đủ ngữ cảnh.
2. **Search retrieval:** Hybrid BM25 (đã tách từ tiếng Việt) + dense, hợp nhất bằng RRF.
3. **Reranking:** Cross-encoder bge-reranker-v2-m3 cho top-20 → top-3; cân nhắc FlashRank nếu latency quan trọng.
4. **Evaluation:** RAGAS 4 metric trên bộ test riêng, theo dõi bottom-N bằng Diagnostic Tree.
5. **Enrichment:** Contextual prepend + metadata (phiên bản, loại tài liệu) để lọc.

#### 3. Timeline triển khai
- **Tuần 1:** Dựng chunking + hybrid search, lập bộ test 30–50 câu.
- **Tuần 2:** Thêm reranker và đo RAGAS; sửa theo failure analysis.
- **Tuần 3:** Enrichment, tối ưu latency/cost, viết tài liệu vận hành.
