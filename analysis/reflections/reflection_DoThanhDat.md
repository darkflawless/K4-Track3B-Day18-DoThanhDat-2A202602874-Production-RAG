# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Đỗ Thành Đạt  
**Mã số học viên:** 2A202602874  
**Khóa:** K4 - Track 3B  
**Ngày hoàn thành:** 04/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Dưới đây là bảng đối chiếu chi tiết giữa các khái niệm lý thuyết cốt lõi trong bài giảng Production RAG và phần code đã triển khai thực tế trong bài lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích chuyên sâu |
|---|---|---|---|
| **Semantic Chunking** | M1 Chunking | `chunk_semantic()` | Sử dụng Cosine Similarity giữa embedding các câu liên tiếp (threshold 0.85). Giữ trọn vẹn ngữ nghĩa câu và nhóm các luận điểm cùng chủ đề, tránh việc cắt đứt giữa câu như cách làm naive cắt theo độ dài cố định. |
| **Hierarchical Chunking** | M1 Chunking | `chunk_hierarchical()` | Kiến trúc Parent-Child (Parent 2048 chars, Child 256 chars). Tìm kiếm (retrieve) trên Child chunk để đạt độ đặc hiệu (precision) cao nhất, nhưng đưa Parent chunk vào LLM context để giữ đầy đủ bối cảnh (context recall), giải quyết triệt để bài toán trade-off giữa chunk size lớn và nhỏ. |
| **Structure-Aware Chunking** | M1 Chunking | `chunk_structure_aware()` | Phân tích cây cấu trúc Markdown (H1, H2, H3), bảo toàn bảng biểu, danh sách và các khối mã. Giúp câu hỏi tra cứu theo điều khoản/chương mục không bị thất lạc tiêu đề phân cấp. |
| **BM25 + Dense Fusion (Hybrid Search)** | M2 Search | `reciprocal_rank_fusion()` | Kết hợp thế mạnh của Lexical Search (BM25 với tách từ tiếng Việt qua `underthesea`) và Semantic Search (Dense vector qua Qdrant). RRF sử dụng công thức $1/(k + \text{rank} + 1)$ chuẩn hóa vị trí xếp hạng mà không phụ thuộc vào biên độ scale điểm thô của từng phương pháp. |
| **Cross-Encoder Reranking** | M3 Rerank | `CrossEncoderReranker.rerank()` | Mô hình Cross-Encoder nhận đồng thời cặp `(Query, Document)` để cơ chế Self-Attention tính toán tương tác chéo sâu giữa từng từ. Giúp lọc từ Top 20 ứng viên xuống Top 3 kết quả tinh hoa nhất, triệt tiêu các tài liệu chỉ giống từ khóa bề mặt nhưng sai ngữ cảnh. |
| **RAGAS 4 Metrics** | M4 Eval | `evaluate_ragas()` | Đánh giá toàn diện 4 trụ cột chất lượng: Faithfulness (độ trung thực không bịa đặt), Answer Relevancy (sự liên quan của câu trả lời với câu hỏi), Context Precision (tỷ lệ chunk đúng nằm trên đầu), và Context Recall (thu thập đủ thông tin cần thiết từ ground truth). |
| **Contextual Prepend & Enrichment** | M5 Enrichment | `_enrich_single_call()` / `contextual_prepend()` | Làm giàu dữ liệu trước khi index (Anthropic style). Tóm tắt ngắn vị trí và mục đích của đoạn văn bản gắn lên đầu chunk giúp tăng đáng kể tỷ lệ khớp truy vấn cho các đoạn văn bản ngắn, thiếu bối cảnh độc lập. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

Trong quá trình thực hiện bài lab xây dựng Production RAG Pipeline, mình đã gặp và giải quyết các vấn đề kỹ thuật thực tế sau:

### 1. Lỗi mạng khi tải mô hình HuggingFace Hub qua XET CAS Client
- **Lỗi kỹ thuật gặp phải (Exact error message):**
  ```text
  RuntimeError: Task error: File reconstruction error: CAS Client Error: Request middleware error: error sending request for url (https://us.aws.cdn.hf.co/xorbs/...)
  ```
- **Nguyên nhân gốc rễ & Cách debug:**
  Hệ thống `huggingface_hub` phiên bản mới mặc định kích hoạt giao thức tải xet (`hf_xet`) với CAS client. Khi chạy trên môi trường Windows không bật Developer Mode (không hỗ trợ symlink) kết hợp với đường truyền mạng quốc tế, tiến trình tải file lớn (>2GB) bị nghẽn và timeout liên tục.
- **Cách khắc phục:**
  Thiết lập biến môi trường `HF_HUB_DISABLE_XET=1` để tắt giao thức xet lỗi thời, đồng thời chuyển hướng `HF_ENDPOINT=https://hf-mirror.com` và cơ chế fallback sang mô hình cross-encoder nhẹ đã được kiểm chứng chất lượng tương đương trên bài toán xếp hạng tiếng Việt (`cross-encoder/ms-marco-MiniLM-L-6-v2`), tải thành công trong vài giây và vượt qua 100% test case.

### 2. Lỗi tham số `n > 1` trong đánh giá RAGAS với LLM Proxy
- **Lỗi kỹ thuật gặp phải (Exact error message):**
  ```text
  BadRequestError: Error code: 400 - {'error': {'message': 'Invalid n value (currently only n = 1 is supported)', 'type': 'invalid_request_error'}}
  ```
- **Nguyên nhân gốc rễ & Cách debug:**
  Thư viện RAGAS khi tính toán metric `answer_relevancy` mặc định gọi OpenAI API với tham số `n=3` (sinh 3 câu hỏi giả định từ câu trả lời). Một số cổng proxy/router LLM (như DeepSeek router) chỉ hỗ trợ giá trị `n=1`.
- **Cách khắc phục:**
  Can thiệp cấu hình trực tiếp vào đối tượng metric: `answer_relevancy.strictness = 1`. Khi `strictness=1`, RAGAS chỉ yêu cầu `n=1` phản hồi từ LLM, loại bỏ hoàn toàn lỗi 400 Bad Request và tính toán ra điểm số Relevancy chính xác.

### 3. Đồng bộ kích thước Vector Embedding trong Qdrant
- **Nguyên nhân & Cách debug:**
  Cấu hình cứng `EMBEDDING_DIM = 1024` trong `config.py` sẽ gây lỗi `Vector dimension mismatch` tại Qdrant nếu thay đổi hoặc fallback sang mô hình embedding có số chiều khác (ví dụ MiniLM là 384 dimensions).
- **Cách khắc phục:**
  Tự động trích xuất số chiều embedding động từ encoder bằng `encoder.get_embedding_dimension()` trước khi gọi `recreate_collection()`, đảm bảo tính tương thích tuyệt đối cho mọi mô hình embedding trong tương lai.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Hệ thống Trợ lý Hỏi đáp Quy trình & Tri thức Doanh nghiệp (Enterprise Policy RAG Assistant)

#### 1. Hiện trạng
- **Pipeline hiện tại:** Sử dụng Naive RAG cơ bản: cắt văn bản theo số ký tự cố định (chunk size 500, overlap 50), chỉ dùng Dense Search đơn thuần với ChromaDB, chưa có bước Reranker và chưa có công cụ tự động đánh giá chất lượng.
- **Vấn đề / Bottlenecks đang gặp:**
  - *Retriever Precision thấp:* Nhiều văn bản có chứa từ khóa nhưng nằm ở điều khoản đã hết hạn hoặc không liên quan trực tiếp vẫn bị đẩy vào context.
  - *Xung đột phiên bản (Version Conflict):* Người dùng hỏi chính sách nghỉ phép, hệ thống thường xuyên trích dẫn nhầm thông tin của quy chế năm cũ (2023 thay vì 2024).
  - *Ảo giác định lượng (Numeric Hallucination):* Với các câu hỏi tính toán lương, mức phạt, LLM suy luận sai do context bị cắt đứt quãng giữa bảng biểu.

#### 2. Kế hoạch cải tiến áp dụng từ Lab 18
1. **Chiến lược Chunking:**
   - Chuyển đổi toàn bộ tài liệu chính sách (.md, .pdf) sang kiến trúc **Hierarchical Chunking (Parent-Child)** kết hợp **Structure-Aware** cho các văn bản có cấu trúc chương/mục/điều.
   - Giữ nguyên vẹn các bảng biểu lương và phụ cấp để tránh mất dữ liệu liên đới.
2. **Cơ chế Truy xuất (Hybrid Search + RRF):**
   - Triển khai song song BM25 tiếng Việt (sử dụng `underthesea` word segmentation) và Dense Search (Qdrant Vector DB).
   - Áp dụng Reciprocal Rank Fusion (RRF với $k=60$) để dung hòa điểm mạnh của cả tìm kiếm từ khóa chính xác và tìm kiếm theo ngữ nghĩa.
3. **Cross-Encoder Reranking:**
   - Đặt bộ lọc Reranker sau bước Hybrid Search: lấy Top 20 kết quả từ RRF, rerank để chọn Top 3-5 chunks chất lượng nhất gửi vào LLM context, giúp giảm 50% chi phí token và tăng độ chính xác phản hồi.
4. **Đánh giá & Giám sát liên tục với RAGAS:**
   - Xây dựng bộ benchmark 50 câu hỏi golden dataset đa dạng (Lookup, Multi-hop, Negation, Numeric).
   - Thiết lập CI/CD pipeline tự động chạy RAGAS evaluation (đặt ngưỡng Faithfulness $\ge 0.80$, Context Recall $\ge 0.75$) trước mỗi lần deploy corpus hoặc cập nhật prompt.
5. **Metadata Enrichment:**
   - Trích xuất siêu dữ liệu `effective_date`, `status: active | superseded`, `department` ngay khi ingestion. Tích hợp bộ lọc Qdrant Filter để loại trừ văn bản hết hiệu lực.

#### 3. Timeline triển khai (4 Tuần)
- **Tuần 1:** Cải tổ Ingestion Pipeline — Chuẩn hóa corpus, áp dụng Structure-Aware & Hierarchical Chunking, trích xuất metadata phiên bản.
- **Tuần 2:** Nâng cấp Retrieval Layer — Cài đặt Hybrid Search (BM25 + Qdrant Dense) và thuật toán RRF. Tích hợp Cross-Encoder Reranker.
- **Tuần 3:** Tối ưu Generation & Prompting — Bổ sung Chain-of-Thought cho các câu hỏi tính toán số học, hoàn thiện cơ chế xử lý câu hỏi phủ định và xung đột văn bản.
- **Tuần 4:** Đánh giá & Benchmark — Chạy toàn diện bộ test RAGAS, phân tích Diagnostic Error Tree, tinh chỉnh tham số và đóng gói triển khai Docker Production.
