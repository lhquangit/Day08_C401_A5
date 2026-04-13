# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Đoàn Sĩ Linh  
**Vai trò trong nhóm:** Retrieval Owner  
**Ngày nộp:** 2026-04-13

---

## 1. Tôi đã làm gì trong lab này? 

Trong dự án này, tôi đảm nhận vai trò Retrieval Owner, chịu trách nhiệm chính cho toàn bộ logic tìm kiếm và tiền xử lý câu hỏi. Ở Sprint 2, tôi xây dựng nền tảng Dense Retrieval kết nối với ChromaDB. Sang Sprint 3, tôi tập trung nâng cấp hệ thống bằng cách triển khai Hybrid Retrieval và Query Transformation. Tôi cũng là người viết các hàm điều hướng (`_choose_query_strategy`) và lọc kết quả (`source filter`) để tối ưu hóa context đưa vào prompt. Ngoài ra, tôi đã xây dựng các script tự động hóa việc chạy so sánh giữa Baseline và các Variant để cung cấp dữ liệu cho Eval Owner làm scorecard. Sự kết hợp giữa logic tìm kiếm của tôi và phần tối ưu hóa Index của đồng đội đã giúp nhóm giải quyết được các câu hỏi multi-hop khó.

---

## 2. Điều tôi hiểu rõ hơn sau lab này 

Sau lab này, tôi hiểu sâu sắc hơn về mô hình "Retrieval Funnel". Thay vì chỉ đơn giản là gọi API và lấy top-k, tôi nhận ra tầm quan trọng của việc "Search rộng - Chọn hẹp". Việc retrieve 10-20 ứng viên (top_k_search) rồi dùng reranking hoặc filtering để chọn ra 3-4 chunk tinh túy nhất (top_k_select) giúp giảm nhiễu (noise) đáng kể cho LLM. Tôi cũng vỡ lẽ ra rằng Hybrid Retrieval không phải lúc nào cũng tốt hơn Dense. Nếu không có bộ lọc hoặc router thông minh, phần Keyword Search (Sparse) đôi khi kéo về những đoạn văn bản chứa nhiều từ khóa nhưng sai ngữ cảnh, làm loãng thông tin. Chính việc thử nghiệm A/B liên tục đã giúp tôi hiểu cách cân bằng giữa Semantic Similarity và Keyword Matching.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn 

Điều làm tôi bất ngờ nhất là Variant "Hybrid + Rerank" đầu tiên của nhóm lại có kết quả tệ hơn Baseline ở một số metric quan trọng. Tôi đã mất khá nhiều thời gian để debug và nhận ra rằng thuật toán RRF nếu không được tinh chỉnh trọng số sẽ dễ bị các chunk chứa nhiều từ khóa lặp lại (keyword stuffing) chiếm ưu thế. Khó khăn lớn nhất là xử lý các câu hỏi về biệt danh hoặc tên cũ của tài liệu (như "Approval Matrix"). Ban đầu tôi định fine-tune embedding nhưng sau đó tôi nhận thấy dùng Query Expansion với alias mapping đơn giản lại mang lại hiệu quả tức thì và ổn định hơn nhiều. Bài học rút ra là: đôi khi một giải pháp rule-based thông minh ở tầng Retrieval lại hiệu quả hơn việc cố gắng làm phức tạp hóa model.

---

## 4. Phân tích một câu hỏi trong scorecard 

**Câu hỏi:** `gq06` — “Lúc 2 giờ sáng xảy ra sự cố P1, on-call engineer cần cấp quyền tạm thời cho một engineer xử lý incident. Quy trình cụ thể như thế nào và quyền này tồn tại bao lâu?”

**Phân tích:**
Đây là câu hỏi cross-document điển hình yêu cầu thông tin từ hai nguồn: `support/sla-p1-2026.pdf` (về sự cố P1) và `it/access-control-sop.md` (về quy trình cấp quyền). Ở bản Baseline (Dense), hệ thống chỉ đạt `Context Recall = 2/5` vì nó chỉ tìm thấy tài liệu SLA mà bỏ lỡ SOP cấp quyền, dẫn đến câu trả lời thiếu ý "thời gian 24 giờ" và " Tech Lead phê duyệt". 
Trong bản Variant của tôi, nhờ cơ chế `Auto Router` và `Query Expansion`, khi nhận thấy từ khóa "cấp quyền", hệ thống đã tự động mở rộng query và ưu tiên tìm kiếm trong `it/access-control-sop.md`. Kết quả là Variant đã retrieve thành công cả 2 nguồn dự kiến, nâng điểm `Completeness` từ 2/5 lên 4/5. Điều này minh chứng rằng với các câu hỏi phức tạp, tầng Retrieval cần phải có khả năng hiểu loại câu hỏi để điều chỉnh chiến thuật tìm kiếm linh hoạt.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? 

Nếu có thêm thời gian, tôi sẽ triển khai một bộ Cross-Encoder thực thụ (như BGE-Reranker) để thay thế cho logic rerank dựa trên mật độ từ khóa hiện tại, giúp việc xếp hạng ứng viên chính xác hơn. Tôi cũng muốn thử nghiệm kỹ thuật HyDE (Hypothetical Document Embeddings) để xử lý các câu hỏi mang tính diễn giải cao, nơi người dùng không sử dụng từ khóa chuyên môn. Cuối cùng, tôi sẽ xây dựng một bảng dashboard trực quan hóa các chunk được chọn để dễ dàng quan sát xem nguồn nào đang bị "over-retrieved" hoặc "under-retrieved".

---
