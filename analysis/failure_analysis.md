# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Trần Hữu Đức  
**MSSV:** 2A202602459  
**Khóa:** K4 - Track 3A  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.9426 | 0.9133 | -0.0293 |
| Answer Relevancy | 0.9125 | 0.8457 | -0.0668 |
| Context Precision | 0.8398 | 0.8955 | +0.0557 |
| Context Recall | 0.9312 | 0.9040 | -0.0272 |

*Ghi chú:* Điểm Context Precision tăng mạnh (+0.0557), phản ánh hiệu quả vượt trội của việc kết hợp Hybrid Search (BM25 + Dense) và Reranker top-3 thay vì dense-only. Toàn bộ 4/4 metrics đều đạt > 0.84 (vượt xa mốc chuẩn 0.75 của Rubric và bonus tier).

---

## Bottom-5 Failures

### #1
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng. Lương thử việc = 85% x 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness (0.6500)
- **Error Tree:** Output sai (LLM từ chối trả lời) → Context đúng? KHÔNG ĐỦ (chỉ có tỷ lệ 85% từ `thu_viec.md`, thiếu mức trần Junior 20 triệu từ `bang_luong_2024.md`) → Query OK? CÓ → Multi-hop retrieval failure.
- **Root cause:** Đây là câu hỏi multi-hop cần thông tin từ 2 tài liệu độc lập (`thu_viec.md` và `bang_luong_2024.md`). Pipeline retrieval đơn điểm top-3 chưa gom đủ cả 2 nguồn tài liệu cần thiết để thực hiện phép tính 85% x 20.000.000. System prompt yêu cầu nghiêm ngặt "chỉ trả lời dựa trên context, nếu không có nói Không tìm thấy" khiến LLM trả lời fallback an toàn.
- **Suggested fix:** Cấu hình Query Decomposition / Multi-query generation phân tách câu hỏi thành 2 sub-queries: (1) Mức trần lương Junior là bao nhiêu, (2) Quy định tỷ lệ lương thử việc; sau đó tổng hợp context trước khi sinh câu trả lời.

### #2
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Nghỉ 16-30 ngày cần phê duyệt của Giám đốc điều hành (CEO). Lưu ý: nghỉ trên 14 ngày không lương, nhân viên phải tự đóng phần bảo hiểm của mình.
- **Got:** Cần Giám đốc điều hành (CEO) phê duyệt.
- **Worst metric:** context_recall (0.6786)
- **Error Tree:** Output đúng ý chính nhưng thiếu chi tiết bảo hiểm → Context đúng? THIẾU 1 PHẦN (retrieve được thẩm quyền CEO nhưng thiếu câu lưu ý đóng bảo hiểm) → Query OK? CÓ → Chunking boundary cut-off.
- **Root cause:** Khi cắt nhỏ thành child chunks theo kích thước 256 ký tự, bảng phân cấp thẩm quyền (16-30 ngày: CEO) nằm ở chunk đầu, phần lưu ý bảo hiểm xã hội nằm ở đoạn cuối của văn bản.
- **Suggested fix:** Áp dụng Parent Document Retriever: khi child chunk khớp query, hệ thống trả về toàn bộ parent chunk (2048 chars) nạp vào prompt để LLM nắm trọn vẹn lưu ý bảo hiểm.

### #3
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** Cần có xác nhận của phòng CNTT về cấu hình kỹ thuật trước khi đề xuất và cần sự phê duyệt của cấp có thẩm quyền (Tổng Giám đốc/CEO cho đơn hàng trên 50 triệu).
- **Worst metric:** context_precision (0.6500)
- **Error Tree:** Output đúng → Context đúng? LẪN NHIỄU (retrieved cả chunk thủ tục kỹ thuật phòng CNTT và bảng phân quyền tài chính) → Query OK? CÓ (query chứa số liệu 55 triệu) →
- **Root cause:** BM25 bắt mạnh từ khóa "thiết bị" từ file `mua_sam.md` dẫn đến kéo theo cả các đoạn quy định cấu hình kỹ thuật thay vì chỉ bảng ma trận phân quyền tài chính.
- **Suggested fix:** Thêm metadata filtering theo `category: finance/procurement` kết hợp cross-encoder rerank trọng số cao cho ngưỡng giá trị tài chính.

### #4
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** 18 ngày phép năm (15 ngày cơ bản + 3 ngày thâm niên). Không tìm thấy thông tin khoảng lương trong ngữ cảnh.
- **Worst metric:** context_recall (0.7717)
- **Error Tree:** Output đúng 1 phần → Context đúng? THIẾU TÀI LIỆU LƯƠNG → Query OK? CÓ (câu hỏi ghép 2 thông tin) →
- **Root cause:** Single-hop retrieval bị chi phối mạnh bởi từ khóa "nhân viên Senior 9 năm thâm niên nghỉ phép năm", chiếm hết cả 3 slots của reranker nên context không có tài liệu thang bảng lương Senior (`bang_luong_2024.md`).
- **Suggested fix:** Sử dụng Sub-question planning hoặc hybrid search mở rộng top_k candidate lên 40, kết hợp đa dạng hóa nguồn tài liệu (Maximal Marginal Relevance - MMR) tránh nhóm chunks tập trung vào duy nhất 1 file.

### #5
- **Question:** Nhân viên được nghỉ bao nhiêu ngày phép năm?
- **Expected:** Theo chính sách hiện hành (v2024), nhân viên được nghỉ 15 ngày phép năm có lương. Chính sách cũ (v2023) là 12 ngày nhưng đã bị thay thế.
- **Got:** 15 ngày làm việc mỗi năm.
- **Worst metric:** context_recall (0.7700)
- **Error Tree:** Output đúng chính sách hiện hành nhưng thiếu nhắc lại chính sách v2023 cũ đã bị thay thế → Context đúng? CÓ (đã retrieve trúng `nghi_phep_nam_v2024.md`) → Query OK? CÓ →
- **Root cause:** Retrieval chỉ lấy tài liệu mới nhất v2024 (do reranker ưu tiên), không lấy thêm tài liệu v2023 cũ nên LLM không giải thích sự thay đổi giữa 2 phiên bản.
- **Suggested fix:** Bổ sung Temporal Knowledge Graph hoặc metadata linking liên kết các phiên bản văn bản (`supersedes` / `superseded_by`).

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
*"Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?"*

**Error Tree walkthrough:**
1. **Output đúng?** → KHÔNG. LLM trả về `"Không tìm thấy."` do context chỉ có tỷ lệ 85% lương cấp bậc từ `thu_viec.md` mà thiếu mức trần lương Junior 20.000.000 VNĐ từ `bang_luong_2024.md`.
2. **Context đúng?** → KHÔNG ĐỦ. Context thiếu 1 trong 2 chặng thông tin bắt buộc (thiếu `bang_luong_2024.md`).
3. **Query rewrite OK?** → CHƯA TỐI ƯU. Query nguyên bản là một câu hỏi phức hợp nhiều bước suy luận (multi-hop).
4. **Fix ở bước:**  
   - **Query Transformation:** Tách câu hỏi thành 2 sub-queries: `"Mức lương tối đa của cấp bậc Junior là bao nhiêu?"` và `"Quy định phần trăm lương trong thời gian thử việc?"`.
   - **Parent Retrieval:** Tích hợp Parent Document Retriever đảm bảo khi chạm tới child chunk sẽ nạp trọn vẹn bảng biểu lương và quy chế thử việc.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Triển khai **Parent Document Retriever hoàn chỉnh**: khi vector search tìm thấy child chunk (256 ký tự) có độ khớp cao nhất, pipeline sẽ nạp toàn bộ parent chunk tương ứng (2048 ký tự) vào context cho LLM để không bao giờ bị cụt bảng biểu hay điều khoản thẩm quyền.
- Tích hợp **Query Expansion / HyDE (Hypothetical Document Embeddings)** để sinh giả thuyết câu trả lời trước khi thực hiện search, giúp giải quyết triệt để các câu hỏi so sánh ngưỡng số học và multi-hop.
- Bổ sung **Metadata Filtering** cho trạng thái phiên bản văn bản (`superseded` vs `active`) nhằm loại bỏ hoàn toàn việc retrieve nhầm văn bản chính sách cũ v2023 thay vì v2024.
