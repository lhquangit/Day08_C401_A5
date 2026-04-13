# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Dương Trung Hiếu  
**Vai trò trong nhóm:** Documentation Owner
**Ngày nộp:** 13/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

Trong dự án Lab 08, tôi đảm nhận vai trò Documentation Owner với trọng tâm là quản lý chất lượng bộ dữ liệu đánh giá và phân tích kết quả đầu ra, hoàn toàn không tham gia vào việc lập trình trực tiếp.

Cụ thể, ở giai đoạn đầu, tôi chịu trách nhiệm rà soát toàn bộ file test_questions.json. Tôi đọc kỹ từng câu hỏi, đối chiếu với tài liệu gốc để kiểm tra và chuẩn hóa lại các trường expected_answer và expected_source, đảm bảo tính hợp lý và độ chính xác tuyệt đối. Tiếp đó, tôi thiết lập các tiêu chí chuyên sâu để so sánh hiệu năng giữa mô hình Baseline và các Variant. Sau khi nhóm chạy thực tế và có file kết quả từ hệ thống, tôi trực tiếp đọc, đối chiếu điểm số của từng câu (scorecard), từ đó viết các nhận xét cuối cùng và tổng hợp báo cáo đánh giá chuyên sâu (group report) để rút ra kết luận về hiệu quả của các phương pháp.

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

Sau lab này, tôi thực sự hiểu sâu sắc về khái niệm Evaluation Loop (Vòng lặp đánh giá) và tầm quan trọng của Ground Truth (Dữ liệu gốc chuẩn) trong việc xây dựng hệ thống RAG.

Ban đầu, tôi thường chỉ tập trung vào việc làm sao để mô hình trả lời mượt mà. Tuy nhiên, khi trực tiếp rà soát bộ test, tôi nhận ra rằng nếu expected_answer hoặc expected_source bị sai lệch dù chỉ một chút, các metric đánh giá tự động (như Faithfulness hay Completeness) sẽ trở nên hoàn toàn vô nghĩa. Việc xây dựng một tập câu hỏi kiểm thử với bộ đáp án kỳ vọng chuẩn xác là "xương sống" để LLM-as-a-judge có thể chấm điểm công tâm, giúp nhóm biết chính xác thuật toán Retrieval đang bị hụt ở đâu để từ đó có hướng tinh chỉnh phù hợp.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

Khó khăn lớn nhất và mất nhiều thời gian nhất đối với tôi là việc thiết lập và chuẩn hóa Tiêu chí so sánh (Comparison Criteria) cho các Variant. Việc định nghĩa thế nào là một câu trả lời "đầy đủ" (Completeness) cực kỳ mơ hồ khi làm việc với ngôn ngữ tự nhiên.

Tôi cũng rất ngạc nhiên khi rà soát kết quả đánh giá của câu gq05. Mặc dù Baseline lấy đúng tài liệu nguồn, câu trả lời vẫn bị đánh giá completeness rất thấp (2/5) do hệ thống suy luận sai về quyền hạn của đối tượng contractor. Điều này làm tôi nhận ra một sự thật thú vị: một file text cung cấp thông tin (source) đúng chưa chắc đã dẫn đến câu trả lời (answer) đúng. Nó đòi hỏi tôi phải quay lại cập nhật expected_answer thật chi tiết, ghi rõ các điều kiện biên để bộ chấm điểm tự động có thể bắt lỗi chính xác hiện tượng này của LLM.

---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:** Ai là người phê duyệt và quy trình như thế nào khi cần cấp quyền tạm thời (khẩn cấp) trong 24 giờ? (Tham chiếu câu gq06).

**Phân tích:**

Đứng từ góc độ người đánh giá đầu ra, gq06 là một trường hợp "kinh điển" minh họa cho sự thay đổi chất lượng giữa các phiên bản.

Ở Baseline: Điểm số trung bình chỉ đạt 2.75/5. Khi kiểm tra đối chiếu nguồn, tôi phát hiện baseline chỉ kéo được 1/2 expected_sources và bỏ sót hoàn toàn file quy định về khoảng thời gian 24 giờ. Vì lỗi thiếu hụt từ bước indexing/retrieval này, model generation bắt buộc phải trả lời chung chung, dẫn đến điểm Completeness bị kéo xuống rất thấp.

Ở Variant 1: Với việc nhóm tích hợp bộ lọc và mở rộng truy vấn, hệ thống đã trúng được 2/2 nguồn kỳ vọng. Kết quả là điểm số của gq06 vọt lên 4.75/5, model trả lời chính xác từng bước quy trình khẩn cấp. Phân tích case này chứng minh rõ ràng với nhóm rằng: việc giải quyết "nút thắt cổ chai" ở khâu truy xuất (Context Recall) mang lại tác động lớn nhất đến chất lượng câu trả lời cuối cùng.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

Nếu có thêm thời gian, tôi sẽ mở rộng độ phức tạp của file test_questions.json bằng cách bổ sung thêm các Adversarial Questions (câu hỏi đánh lừa/gài bẫy). Ví dụ: cố tình hỏi về một chính sách không hề tồn tại trong kho tài liệu để kiểm tra khả năng "Abstain" (từ chối trả lời) của mô hình. Việc làm phong phú thêm bộ Ground Truth này sẽ giúp nhóm đo lường được khả năng chống ảo giác (hallucination) của pipeline một cách toàn diện hơn thay vì chỉ đo lường độ chính xác.

---

*Lưu file này với tên: `reports/individual/[ten_ban].md`*
*Ví dụ: `reports/individual/nguyen_van_a.md`*
