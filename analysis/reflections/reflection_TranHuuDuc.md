# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Trần Hữu Đức  
**MSSV:** 2A202602459  
**Khóa:** K4 - Track 3A  
**Ngày hoàn thành:** 04/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)
Map từng concept trong lecture vào code bạn vừa viết trong lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích |
|----------------|--------|-------------|--------------------------|
| Semantic chunking | M1 | `chunk_semantic()` | "Ngưỡng cosine threshold 0.85 giúp gom các câu có cùng trường nghĩa thành một đơn vị ngữ nghĩa hoàn chỉnh. Khác với basic chunking (cắt cứng theo độ dài hoặc xuống dòng làm đứt gãy thông tin giữa câu), semantic chunking giữ trọn vẹn ngữ cảnh điều khoản và ý định của người viết." |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | "RRF kết hợp điểm xếp hạng lexical (BM25 bắt chính xác thuật ngữ chuyên ngành tiếng Việt, mã hiệu, tên quy định) và dense vector (bắt ngữ nghĩa mờ và từ đồng nghĩa). Công thức $1/(k + rank + 1)$ với $k=60$ giúp cân bằng thứ hạng mà không bị phụ thuộc vào phân phối điểm số thô khác nhau giữa hai thuật toán." |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | "Đạt latency trung bình ~15-35ms cho 20 documents. Đưa các đoạn tài liệu khớp ngữ nghĩa trực tiếp với câu hỏi (chẳng hạn tài liệu về ngày nghỉ phép so với quy định VPN/mật khẩu) lên top 3, loại bỏ nhiễu trước khi nạp context vào LLM, giúp tăng context precision đáng kể." |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | "Đánh giá toàn diện 4 chỉ số vàng của RAG: Faithfulness (độ trung thực, không ảo giác), Answer Relevancy (mức độ bám sát câu hỏi), Context Precision (tỷ lệ chunks liên quan được xếp trên cùng), Context Recall (tỷ lệ thông tin cần thiết được retrieve). Context Precision trong bài tăng từ 0.8398 lên 0.8955." |
| Contextual embeddings | M5 | `contextual_prepend()` / `_enrich_single_call()` | "Kỹ thuật contextual prepend bổ sung bối cảnh tài liệu nguồn (tên chính sách, chương mục) vào đầu mỗi chunk. Điều này giải quyết bài toán các đoạn văn bản ngắn bị 'mất ngữ cảnh cha' (orphan chunks), giúp vector search và reranker nhận diện chuẩn xác phạm vi tài liệu áp dụng." |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

- **Lỗi kỹ thuật gặp phải (Exact error message):**
  - Lỗi 1: `NameError: name 're' is not defined` trong `src/m4_eval.py` khi chạy fallback evaluation tính điểm token overlap.
  - Lỗi 2: `AssertionError: Semantic chunking should return chunks` và `Child parent_id not in parents` trong `tests/test_m1.py` do cấu trúc phân tách cha-con chưa đồng bộ khóa định danh `parent_id`.
  - Lỗi 3: Qdrant client cảnh báo kết nối gRPC/HTTP timeout và tương thích phiên bản: `Failed to obtain server version. Unable to check client-server compatibility`.
- **Nguyên nhân gốc rễ & Cách debug:**
  - Lỗi 1: Thiếu thư viện built-in `import re` ở đầu file `src/m4_eval.py`. Debug bằng cách đọc traceback của pytest và bổ sung import.
  - Lỗi 2: Trong hàm `chunk_hierarchical`, metadata của parent chunk cần lưu rõ trường `"parent_id": pid` trùng khớp với `parent_id` được gán trên từng child chunk. Đã chuẩn hóa quy trình sinh `pid = f"parent_{i}"` và gán đồng bộ cho cả hai danh sách.
  - Lỗi 3: Client Qdrant tự động chuyển sang chế độ in-memory (`QdrantClient(":memory:")`) khi service Qdrant local chưa sẵn sàng hoặc không thể kết nối cổng 6333, đảm bảo pipeline chạy trơn tru độc lập mà không bị crash giữa chừng.
- **Kiến thức còn thiếu & Cách khắc phục:**
  - Nắm vững cơ chế Reciprocal Rank Fusion (RRF) để kết hợp các danh sách có thang đo điểm khác nhau (BM25 điểm không chặn trên, Cosine điểm [-1, 1]).
  - Hiểu sâu về cách thức xử lý tiếng Việt: `underthesea.word_tokenize` nối từ ghép bằng dấu gạch dưới `_`, nên phải chuyển `replace("_", " ")` để bộ tách từ của BM25 không xem từ ghép và từ đơn là hai token hoàn toàn khác nhau.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

Dựa trên những kỹ thuật đã học và thực hành, lập kế hoạch cụ thể áp dụng vào project của bạn:

### Project: Hệ thống Trợ lý Pháp lý & Tra cứu Quy chế Doanh nghiệp Thông minh (Enterprise Policy Assistant)

#### 1. Hiện trạng
- **Pipeline hiện tại:** Sử dụng Naive RAG cơ bản với recursive character chunking (chunk size 1000, overlap 200), lưu vector trên ChromaDB và truy vấn thuần dense embedding bằng mô hình đa ngôn ngữ tổng quát.
- **Vấn đề / Bottlenecks đang gặp:**
  - Tỷ lệ ảo giác (hallucination) cao khi người dùng hỏi các câu hỏi điều kiện nhiều bước (multi-hop) hoặc so sánh chính sách giữa các năm.
  - Context Precision thấp do văn bản pháp lý có nhiều đoạn định nghĩa chung chung chiếm thứ hạng cao của retrieval.
  - Dữ liệu dạng bảng biểu (bảng lương, ma trận thẩm quyền phê duyệt ngân sách) bị cắt vụn khiến LLM đọc sai số liệu.

#### 2. Kế hoạch cải tiến
1. **Chunking strategy:** Áp dụng kết hợp Structure-Aware Chunking (tách theo từng điều khoản, giữ nguyên cấu trúc bảng biểu markdown) và Hierarchical Chunking (child 256 ký tự cho retrieval độ phân giải cao, trả về parent 2048 ký tự cho LLM đọc đủ ngữ cảnh).
2. **Search retrieval:** Triển khai Hybrid Search (BM25 có phân đoạn từ tiếng Việt kết hợp Dense bge-m3 qua RRF). Điều này bắt buộc để bắt chính xác các số hiệu văn bản (như "Nghị định 13/2023", "MFA", "P3-P4").
3. **Reranking:** Tích hợp Cross-Encoder `BAAI/bge-reranker-v2-m3` để lọc top 20 ứng viên xuống top 3-5 ngữ cảnh tinh túy nhất trước khi sinh lời phản hồi.
4. **Evaluation:** Thiết lập bộ benchmark tự động với 50 câu hỏi golden dataset đa dạng (lookup, temporal change, multi-hop, numeric) và định kỳ chạy RAGAS theo dõi 4 metrics chính trong CI/CD.
5. **Enrichment:** Sử dụng chế độ Combined Single-Call LLM để tự động gán metadata phân loại văn bản (`active/superseded`, `department`, `effective_date`) và contextual prepend tóm tắt vị trí điều khoản.

#### 3. Timeline triển khai
- **Tuần 1:** Tái cấu trúc corpus dữ liệu: chuyển đổi tài liệu quy chế sang Markdown chuẩn hóa; cài đặt module Structure-Aware Chunking và Parent Document Retriever.
- **Tuần 2:** Xây dựng module Hybrid Search (BM25 + Qdrant Vector) kết hợp Cross-Encoder Reranker; tối ưu hóa thời gian phản hồi (latency < 250ms).
- **Tuần 3:** Tích hợp pipeline Enrichment sinh siêu dữ liệu tự động; cấu hình Metadata Filtering theo phòng ban và trạng thái hiệu lực văn bản.
- **Tuần 4:** Đóng gói RAGAS evaluation pipeline vào quy trình kiểm thử tự động; tinh chỉnh prompt template theo Diagnostic Tree để đạt Faithfulness ≥ 0.90 và Answer Relevancy ≥ 0.85 trên toàn bộ hệ thống production.
