# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Vũ Việt Hoàng (2A202602398)  
**Khóa:** K4 - Track 3A  

> **Nguồn số liệu:** `reports/ragas_report.json` và `reports/naive_baseline_report.json`
> (judge = Gemini `gemini-3.5-flash-lite`, 20 câu hỏi). Gói free tier giới hạn 15 request/phút nên nhiều
> job RAGAS bị `429 RateLimitError`; job lỗi được tính 0.0 → **điểm có thể thấp hơn thực tế**,
> nhất là faithfulness.

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.4233 | 0.5083 | +0.0850 |
| Answer Relevancy | 0.5690 | 0.6709 | +0.1019 |
| Context Precision | 0.4500 | 0.5917 | +0.1417 |
| Context Recall | 0.7500 | 0.7167 | -0.0333 |

Production cải thiện 3/4 metric; chỉ context recall giảm nhẹ (child chunk 256 ký tự có thể cắt mất ngữ cảnh).
Metric cao nhất là 0.7167 (≥ 0.70); chưa đạt các ngưỡng bonus.

## Bottom-5 Failures

### #1
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất 20.000.000 VNĐ/tháng; lương thử việc = 85% → 17.000.000 VNĐ/tháng.
- **Got:** "Không tìm thấy."
- **Worst metric:** faithfulness (avg 0.0)
- **Error Tree:** Output sai → Context đúng? **Không** (thiếu chunk bảng lương/thử việc) → Query OK? Có → lỗi ở retrieval/chunking.
- **Root cause:** Cần ghép 2 tài liệu (`bang_luong_2024.md` + `thu_viec.md`); child chunk 256 ký tự không chứa cả hai dữ kiện.
- **Suggested fix:** Trả về parent chunk khi generate, hoặc tăng top-k trước rerank; thêm metadata để lọc theo tài liệu.

### #2
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** v2.0: 120 ngày (chính sách cũ 90 ngày đã bị thay thế).
- **Got:** Nêu cả hai mốc 120 và 90 ngày.
- **Worst metric:** context_recall (0.0; avg 0.25)
- **Error Tree:** Output sai một phần → Context đúng? Lẫn `mat_khau_v1` và `v2` → Query OK? Có → lỗi do phiên bản tài liệu.
- **Root cause:** Hai phiên bản chính sách cùng nằm trong index, không có cách phân biệt bản hiện hành.
- **Suggested fix:** Thêm metadata `version`/`status=current` và lọc khi retrieve; hoặc loại bản đã bị thay thế khỏi index.

### #3
- **Question:** Mua laptop 30 triệu cho nhân viên mới: ai phê duyệt và cần gì từ phòng CNTT?
- **Expected:** Director phê duyệt (5–50 triệu); cần xác nhận cấu hình từ CNTT; ≥3 báo giá vì >10 triệu.
- **Got:** Chỉ nêu được xác nhận của phòng CNTT; nói context không cung cấp người phê duyệt.
- **Worst metric:** faithfulness (0.0; avg 0.3333)
- **Error Tree:** Output thiếu → Context đúng? Thiếu bảng hạn mức phê duyệt → lỗi ở retrieval (multi-hop).
- **Root cause:** Câu hỏi cần 2 chunk (quy trình mua sắm + bảng hạn mức) nhưng chỉ lấy top-3 sau rerank.
- **Suggested fix:** Tăng `RERANK_TOP_K`, hoặc query decomposition tách thành các câu hỏi con.

### #4
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Trên 50 triệu cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** "Tổng Giám đốc (CEO)" — **đúng**.
- **Worst metric:** faithfulness (0.0; avg 0.4534)
- **Error Tree:** Output đúng → điểm 0.0 đáng ngờ → nhiều khả năng là nhiễu của judge/rate limit, không phải lỗi pipeline.
- **Root cause:** Câu trả lời quá ngắn nên RAGAS khó tách statement để kiểm chứng; hoặc job bị 429 và tính 0.0 (chưa xác minh được nguyên nhân nào).
- **Suggested fix:** Prompt yêu cầu trả lời đủ câu; chạy RAGAS với quota cao hơn để loại nhiễu rate limit.

### #5
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG; chỉ được BHXH bắt buộc.
- **Got:** "Không." — **đúng**.
- **Worst metric:** faithfulness (0.0; avg 0.4583)
- **Error Tree:** Output đúng → điểm 0.0 đáng ngờ → cùng nguyên nhân với #4.
- **Root cause:** Như #4 (câu trả lời một từ).
- **Suggested fix:** Như #4.

## Case Study (cho presentation)

**Question chọn phân tích:** "Bao lâu phải đổi mật khẩu một lần?" (#2) — có hai phiên bản chính sách.

**Error Tree walkthrough:**
1. Output đúng? → Một phần: nêu cả 120 ngày (mới) và 90 ngày (cũ).
2. Context đúng? → Không: lẫn `mat_khau_v1` và `mat_khau_v2`.
3. Query rewrite OK? → Có, câu hỏi rõ ràng.
4. Fix ở bước: indexing/retrieval — thêm metadata phiên bản và lọc bản hiện hành.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Trả về parent chunk (2048) thay vì child khi generate để tăng recall cho câu multi-hop.
- Lọc theo metadata/phiên bản tài liệu để tránh lẫn v1/v2.
- Chạy lại RAGAS với quota cao hơn để loại nhiễu rate limit khỏi điểm faithfulness.
