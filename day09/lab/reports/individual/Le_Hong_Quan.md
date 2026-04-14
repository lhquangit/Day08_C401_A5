# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Lê Hồng Quân  
**Vai trò trong nhóm:** Supervisor Owner, Documentation Owner (Tech Lead)  
**Ngày nộp:** 2026-04-14  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Trong lab Day 09, tôi chịu trách nhiệm chính ở 2 phần: `graph.py` (supervisor orchestration) và nhóm tài liệu kiến trúc/evaluation summary. Ở tầng code, tôi tập trung vào `supervisor_node()` và `route_decision()` trong `day09/lab/graph.py`, đặc biệt là rule phân biệt câu factual với câu policy/multi-hop để tránh over-routing sang `policy_tool_worker`. Ở tầng docs, tôi tổng hợp kết quả kỹ thuật thành `docs/system_architecture.md`, `docs/routing_decisions.md`, `docs/single_vs_multi_comparison.md` và `reports/group_report.md`.

**Module/file tôi chịu trách nhiệm:**
- File chính: `day09/lab/graph.py`, `day09/lab/reports/group_report.md`
- Functions tôi implement: `supervisor_node()`, cập nhật logic `route_reason` và `needs_tool`

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Routing của tôi là điểm vào cho toàn pipeline. Nếu route sai, phần của Worker Owner (`policy_tool.py`, `synthesis.py`) và MCP Owner (`mcp_server.py`) sẽ bị gọi sai ngữ cảnh, khiến answer degrade dù từng module có thể đúng độc lập.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**

Bằng chứng trực tiếp nằm ở `graph.py` và các trace sau khi chạy batch mới: `run_20260414_180408.json`, `run_20260414_180447.json`.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi chọn route factual query về `retrieval_worker`, chỉ route `policy_tool_worker` cho policy eligibility hoặc multi-hop access + incident.

**Lý do:**

Lúc đầu, rule theo keyword rộng (thấy `refund`/`access`) dễ đẩy quá nhiều câu sang policy path. Điều này làm pipeline dài hơn và nhạy cảm hơn với lỗi ở policy-tool path, trong khi nhiều câu chỉ cần retrieval + synthesis là đủ. Vì vậy tôi thêm hai tầng nhận diện trong supervisor:
1) `fact_query_markers` để nhận biết câu factual;  
2) `refund_policy_markers` để tách refund policy question khỏi refund factual question.

Tôi chọn cách này vì nó đơn giản, kiểm soát được, dễ debug qua `route_reason`, và có thể cải tiến dần theo trace mà không cần thêm LLM classifier ở supervisor.

**Trade-off đã chấp nhận:**

Rule-based routing có thể bỏ sót một số câu viết “mơ hồ” nếu không chứa marker đã định nghĩa. Đổi lại, nó chạy nhanh, ổn định, và minh bạch hơn khi debug.

**Bằng chứng từ trace/code:**

```text
graph.py (supervisor_node):
- is_refund_policy and not is_fact_query -> policy_tool_worker
- matched_incident or matched_refund factual -> retrieval_worker

Trace evidence:
- run_20260414_180408.json: refund factual -> retrieval_worker
- run_20260414_180447.json: multi-hop access+incident -> policy_tool_worker
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Over-routing trong supervisor làm tỷ lệ gọi policy path cao hơn cần thiết, kéo theo rủi ro fail ở các câu không cần tool reasoning.

**Symptom (pipeline làm gì sai?):**

Các câu factual đơn giản (ví dụ refund factual hoặc FAQ dạng số liệu) có lúc đi qua nhánh policy/tool thay vì retrieval-first. Kết quả là latency tăng và độ ổn định giảm, trong khi chất lượng câu trả lời không tăng tương xứng.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**

Root cause nằm ở routing logic của supervisor: điều kiện match keyword ban đầu quá rộng, chưa tách rõ factual vs policy question.

**Cách sửa:**

Tôi sửa `supervisor_node` trong `graph.py` theo hướng:
- thêm `fact_query_markers`
- thêm `refund_policy_markers`
- chỉ route `policy_tool_worker` khi là access policy, refund policy, hoặc multi-hop access+incident
- còn lại fallback retrieval

**Bằng chứng trước/sau:**

- Trước: trong `day09/lab/artifacts/grading_run.json` (batch 9 traces lúc 17:52), `policy_tool_worker` chiếm `3/9 (33%)`.
- Sau: trong `day09/lab/artifacts/eval_report.json` (batch 12 traces lúc 18:04), `policy_tool_worker` giảm còn `2/12 (16%)`, `retrieval_worker` tăng lên `10/12 (83%)`.
- Trace sau sửa cho câu factual: `run_20260414_180408.json` route đúng `retrieval_worker`.

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Điểm tôi làm tốt nhất là giữ kiến trúc toàn hệ thống “đọc được và debug được”: route có lý do rõ (`route_reason`), docs phản ánh đúng runtime, và quyết định routing bám evidence từ trace thay vì cảm tính. Ở vai trò leader, tôi cũng làm phần điều phối ưu tiên giữa các nhánh công việc (supervisor, worker, MCP, trace/docs), chốt thứ tự xử lý issue theo mức ảnh hưởng đến grading để tránh cả nhóm tối ưu sai chỗ.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Tôi chưa chốt sớm được bộ acceptance test cho multi-hop (đặc biệt `gq03`, `gq09`) ngay sau khi thay đổi router. Vì vậy có giai đoạn docs đã ổn nhưng policy path vẫn chưa đủ robust.

**Nhóm phụ thuộc vào tôi ở đâu?** _(Phần nào của hệ thống bị block nếu tôi chưa xong?)_

Nhóm phụ thuộc vào tôi ở lớp orchestration và quyết định tích hợp cấp nhóm. Nếu supervisor không ổn định, toàn bộ worker phía sau đều bị gọi sai ngữ cảnh và kết quả eval sẽ nhiễu. Ngoài ra, nếu tôi chưa chốt hướng ưu tiên (fix routing trước hay fix policy path trước), các đầu việc của từng owner dễ chạy lệch pha và mất thời gian.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_

Tôi phụ thuộc vào Worker Owner và MCP Owner để harden policy path (schema output, fallback, synthesis rule) sau khi router đã khoanh đúng class câu hỏi.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ bổ sung `route_tag` chuẩn hóa trong supervisor (ví dụ: `refund_factual`, `refund_policy`, `incident_factual`, `incident_access_multihop`) và ghi tag này vào trace. Lý do: hiện `route_reason` đã có thông tin nhưng còn khó tổng hợp định lượng theo category; việc có `route_tag` sẽ giúp đo `route_accuracy` theo từng nhóm câu nhanh hơn và tìm đúng vùng regression như `gq03/gq09` ngay sau mỗi lần chạy.

---
