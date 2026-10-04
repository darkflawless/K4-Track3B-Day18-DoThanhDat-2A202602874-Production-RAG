# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Đỗ Thành Đạt  
**Mã số học viên:** 2A202602874  
**Khóa:** K4 - Track 3B  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.4875 | 0.7768 | +0.2893 |
| Answer Relevancy | 0.4184 | 0.5638 | +0.1454 |
| Context Precision | 0.4083 | 0.7083 | +0.3000 |
| Context Recall | 0.4083 | 0.6292 | +0.2208 |

---

## Bottom-5 Failures

### #1
- **Question:** Nhân viên được nghỉ bao nhiêu ngày khi kết hôn?
- **Expected:** Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn, không trừ vào phép năm.
- **Got:** Trả lời chung chung về chế độ nghỉ việc riêng hoặc số ngày không chính xác do context bị phân mảnh.
- **Worst metric:** Faithfulness (0.0)
- **Error Tree:** Output sai thông tin chi tiết → Context retrieved đúng tài liệu chế độ nghỉ phép nhưng chunk bị cắt đứt đoạn bảng biểu → Query match lexical tốt nhưng thiếu context đầy đủ → Fix ở bước: Chunking (Structure-Aware / Table Preservation).
- **Root cause:** Thông tin nghỉ kết hôn nằm trong mục các trường hợp nghỉ việc riêng hưởng nguyên lương dưới dạng danh sách gạch đầu dòng. Basic/paragraph chunking cắt ngang bảng khiến LLM bị mất dữ kiện "3 ngày làm việc".
- **Suggested fix:** Cải tiến Module 1 Chunking với Structure-Aware giữ trọn vẹn section và table/list, không cắt giữa các điều khoản con.

### #2
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** Trả lời cần Giám đốc bộ phận (Director) hoặc không xác định rõ cấp phê duyệt tối cao.
- **Worst metric:** Faithfulness (0.0)
- **Error Tree:** Output chọn nhầm cấp thẩm quyền → Context chứa nhiều khoảng định mức (5-50 triệu và >50 triệu) → Retrieval đưa về cả 2 mốc thẩm quyền nhưng LLM so sánh số học 55 triệu với 50 triệu bị sai → Fix ở bước: Reranking & Numeric Reasoning Prompt.
- **Root cause:** Văn bản mua sắm trang thiết bị quy định nhiều cấp bậc thẩm quyền theo các mốc tiền. LLM bị ảo giác khi đối chiếu con số 55 triệu vào khoảng định mức.
- **Suggested fix:** Tối ưu prompt yêu cầu trích dẫn rõ điều kiện so sánh số liệu (Numeric Chain-of-Thought) và sử dụng Cross-Encoder để đẩy chunk chứa định mức ">50.000.000 VNĐ" lên top 1.

### #3
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng. Lương thử việc = 85% x 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Trả về mức lương chính thức 20.000.000 VNĐ hoặc không áp dụng tỷ lệ 85% thử việc.
- **Worst metric:** Faithfulness (0.0)
- **Error Tree:** Output thiếu bước tính 85% → Context retrieved chỉ lấy được bảng lương Junior mà thiếu chunk quy định 85% lương thử việc → Multi-hop query failure → Fix ở bước: Multi-hop Query Decomposition & Parent-Child Retrieval.
- **Root cause:** Đây là câu hỏi Multi-hop: thông tin dải lương Junior nằm ở file quy chế lương, còn quy định thử việc 85% nằm ở quy chế tuyển dụng/thử việc. Retriever chỉ lấy được 1 trong 2 nguồn context.
- **Suggested fix:** Phân rã câu hỏi thành 2 sub-queries: (1) "Mức trần lương Junior là bao nhiêu?" và (2) "Quy định tỷ lệ lương thời gian thử việc là bao nhiêu %?", sau đó tổng hợp context.

### #4
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Trình bày nguyên văn quy chế tạm ứng và lãi suất 2%/tháng nhưng không tính toán ra số tiền phạt cụ thể.
- **Worst metric:** Answer Relevancy (0.0)
- **Error Tree:** Output không trả lời đúng trọng tâm số tiền phạt → Context đúng về quy định thời hạn 15 ngày và lãi phạt 2%/tháng → LLM từ chối tính toán số học pro-rata → Fix ở bước: Generation Prompting (Calculation CoT).
- **Root cause:** System prompt yêu cầu "Trả lời CHỈ dựa trên context. Nếu không có → nói 'Không tìm thấy.'" khiến LLM ngần ngại thực hiện phép tính số học suy diễn (5 ngày quá hạn x 2%/tháng x 15 triệu).
- **Suggested fix:** Tinh chỉnh prompt cho phép thực hiện các phép tính số học cơ bản dựa trên các tham số có sẵn trong context.

### #5
- **Question:** Nhân viên được nghỉ bao nhiêu ngày phép năm?
- **Expected:** Theo chính sách hiện hành (v2024), nhân viên được nghỉ 15 ngày phép năm có lương. Chính sách cũ (v2023) là 12 ngày nhưng đã bị thay thế.
- **Got:** Trả về cả 12 ngày và 15 ngày hoặc trả lời 12 ngày theo chính sách cũ.
- **Worst metric:** Answer Relevancy (0.0)
- **Error Tree:** Output bị mâu thuẫn phiên bản → Context retrieved cả `nghi_phep_nam_v2023.md` (12 ngày) và `nghi_phep_nam_v2024.md` (15 ngày) → Version Conflict → Fix ở bước: Metadata Filtering & Temporal Ranking.
- **Root cause:** Corpus lưu trữ cả văn bản cũ đã hết hiệu lực (`v2023`) và văn bản hiện hành (`v2024`). Cả 2 tài liệu đều có độ tương đồng ngữ nghĩa cực cao với câu hỏi "nghỉ phép năm".
- **Suggested fix:** Bổ sung trường metadata `status: active | superseded` và `effective_year`. Thêm bước tiền xử lý lọc chỉ lấy văn bản có hiệu lực hiện hành.

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
*"Nhân viên được nghỉ bao nhiêu ngày phép năm?"* (Vấn đề xung đột phiên bản tài liệu - Temporal/Version Conflict).

**Error Tree walkthrough:**
1. **Output đúng?**  
   → **SAI**: Model đưa ra câu trả lời chứa thông tin mâu thuẫn: 12 ngày (v2023) và 15 ngày (v2024), hoặc kết luận 12 ngày theo bản cũ.
2. **Context đúng?**  
   → **MỘT NỬA ĐÚNG**: Retriever lấy về cả 2 văn bản: `nghi_phep_nam_v2023.md` và `nghi_phep_nam_v2024.md`. Cả 2 đều match từ khóa "nghỉ phép năm", nhưng một văn bản đã hết hiệu lực.
3. **Query rewrite OK?**  
   → Query gốc người dùng không nói rõ năm áp dụng, nên Dense & BM25 đều chấm điểm cao cho cả 2 tài liệu.
4. **Fix ở bước:**  
   → **Fix tại Metadata Enrichment (M5) & Hybrid Search Filtering (M2)**: 
     - M5 trích xuất `version`, `effective_date`, `status` vào metadata.
     - M2 áp dụng filter: loại trừ các chunks có `status == "superseded"` hoặc ưu tiên `version` cao nhất khi cùng topic.

**Nếu có thêm 1 giờ, sẽ optimize:**
1. **Temporal Filtering & Recency Boosting:** Tích hợp bộ lọc siêu dữ liệu theo ngày hiệu lực vào Qdrant filter, tự động giảm điểm các văn bản cũ bị thay thế.
2. **Multi-hop Query Decomposition:** Tự động tách các câu hỏi phức tạp (như câu tính lương thử việc Junior) thành các truy vấn đơn lẻ để retriever thu thập đủ context đa nguồn.
3. **Fine-tuned Cross-Encoder Reranker cho tiếng Việt:** Huấn luyện thêm Cross-Encoder trên tập dữ liệu quy chế doanh nghiệp để phân biệt rõ ngữ cảnh mâu thuẫn phiên bản.
