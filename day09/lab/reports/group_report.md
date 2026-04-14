# Báo Cáo Nhóm — Lab Day 09: Multi-Agent Orchestration

**Tên nhóm:** ___________  
**Thành viên:**


| Tên              | Vai trò                                          | Email                                                     |
| ---------------- | ------------------------------------------------ | --------------------------------------------------------- |
| Lê Hồng Quân     | Tech Lead, Supervisor Owner, Documentation Owner | [hongquanliv@gmail.com](mailto:hongquanliv@gmail.com)     |
| Đoàn Sĩ Linh     | Worker Owner                                     | [doansilinh04@gmail.com](mailto:doansilinh04@gmail.com)   |
| Dương Trung Hiếu | MCP Owner                                        | [duonghieu734@gmail.com](mailto:duonghieu734@gmail.com)   |
| Nguyễn Đức Hải   | Trace Owner                                      | [nguyenhai6586@gmail.com](mailto:nguyenhai6586@gmail.com) |
| Phạm Thanh Lam   | Retrieval Owner                                  | [lamphamaudio@gmail.com](mailto:lamphamaudio@gmail.com)   |


**Ngày nộp:** 2026-04-14  
**Repo:** `https://github.com/lhquangit/Lecture-Day-08-09-10.git`  

---

## 1. Kiến trúc nhóm đã xây dựng

Nhóm xây dựng hệ thống theo pattern supervisor-worker với một shared `AgentState` đi xuyên toàn graph. `supervisor` trong [graph.py](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/graph.py:95) phân tích task rồi ghi `supervisor_route`, `route_reason`, `risk_high`, `needs_tool`. Từ đó graph route sang một worker xử lý chính là `retrieval_worker` hoặc `policy_tool_worker`, sau đó luôn đi qua `synthesis_worker` để sinh `final_answer`, `sources` và `confidence`. `human_review` vẫn tồn tại trong graph như một edge hợp lệ, nhưng ở runtime hiện tại supervisor không còn route mặc định sang nhánh này nữa. Phần retrieval nằm ở [workers/retrieval.py](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/workers/retrieval.py:449), phần policy/tool reasoning ở [workers/policy_tool.py](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/workers/policy_tool.py:423), và phần tổng hợp answer ở [workers/synthesis.py](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/workers/synthesis.py:387). MCP layer được mô phỏng trong [mcp_server.py](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/mcp_server.py:103) với các tool `search_kb`, `get_ticket_info`, `check_access_permission`, `create_ticket`.

**Routing logic cốt lõi:**

Supervisor hiện dùng rule-based keyword routing. Query access/security hoặc multi-hop access + incident/SLA được route sang `policy_tool_worker`; query incident/SLA factual và refund factual được route sang `retrieval_worker`; `ERR-`* chỉ set `risk_high=True` chứ không tự route sang HITL. Ví dụ trong trace [run_20260414_180447.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces/run_20260414_180447.json:1), route reason là `multi-hop access + incident query ... | risk_high via: emergency, 2am`, còn trong [run_20260414_180428.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces/run_20260414_180428.json:1), `ERR-403-AUTH` đi retrieval-first và abstain đúng.

**MCP tools đã tích hợp:**

- `search_kb`: gọi retrieval path dùng chung với `workers/retrieval.py`; ví dụ policy path gọi tool này 3 lần trong [run_20260414_175221.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces_grading_question/run_20260414_175221.json:1).
- `get_ticket_info`: dùng để bổ sung ticket summary cho `incident_access`; xuất hiện trong [run_20260414_175221.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces_grading_question/run_20260414_175221.json:1).
- `check_access_permission`: dùng để suy ra `required_approvers`, `emergency_override`, `notes` cho access rule; xuất hiện trong [run_20260414_175202.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces_grading_question/run_20260414_175202.json:1).

---

## 2. Quyết định kỹ thuật quan trọng nhất

**Quyết định:** Không để mọi câu có keyword policy đều đi `policy_tool_worker`; thay vào đó chia rõ `retrieval factual path` và `policy/multi-hop path`.

**Bối cảnh vấn đề:**

Trong các phiên bản router đầu, các câu có chữ `refund`, `access`, `ERR-`* rất dễ bị đẩy sang policy hoặc HITL path, khiến pipeline dài hơn và dễ phát sinh lỗi runtime không cần thiết. Điều này đặc biệt bất lợi cho các câu fact đơn như “bao nhiêu ngày”, “mấy bước”, “sau 10 phút thì sao”. Nhóm nhận ra rằng mục tiêu của Day 09 không phải biến mọi query thành multi-agent, mà là chỉ orchestration khi thật sự cần tool reasoning hoặc cross-document logic.

**Các phương án đã cân nhắc:**


| Phương án                                                                                                | Ưu điểm                                      | Nhược điểm                                            |
| -------------------------------------------------------------------------------------------------------- | -------------------------------------------- | ----------------------------------------------------- |
| Route rộng: hễ có keyword policy/error là vào `policy_tool_worker` hoặc `human_review`                   | Bao phủ mạnh, dễ nghĩ                        | Over-routing, tăng latency, dễ fail ở policy/MCP path |
| Route hẹp: factual query đi retrieval, chỉ policy eligibility hoặc multi-hop mới đi `policy_tool_worker` | Pipeline ngắn hơn, ổn định hơn, dễ debug hơn | Cần mô tả rule rõ để không bỏ sót case khó            |


**Phương án đã chọn và lý do:**

Nhóm chọn phương án thứ hai. Lý do là batch trace mới nhất cho thấy retrieval-only path đang trả lời tốt các câu đơn như SLA P1, refund days, lockout, remote policy, trong khi policy path vẫn dễ fail ở các case multi-hop nếu tool output thiếu hoặc không đồng nhất schema. Vì vậy, router hiện tại chỉ đẩy sang `policy_tool_worker` khi query là access/security, refund mang tính eligibility/exception, hoặc multi-hop access + incident. Cách này giúp giảm over-orchestration mà vẫn giữ được lợi ích của multi-agent ở các câu phức tạp.

**Bằng chứng từ trace/code:**

```text
[graph.py]
if is_multi_hop:
    route = "policy_tool_worker"
elif is_access_policy:
    route = "policy_tool_worker"
elif is_refund_policy and not is_fact_query:
    route = "policy_tool_worker"
elif matched_incident:
    route = "retrieval_worker"
elif matched_refund:
    route = "retrieval_worker"
```

Ví dụ thật:

- [run_20260414_180408.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces/run_20260414_180408.json:1): `matched refund factual query: hoàn tiền` → `retrieval_worker`
- [run_20260414_180447.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces/run_20260414_180447.json:1): `multi-hop access + incident query ...` → `policy_tool_worker`

---

## 3. Kết quả grading questions

Nhóm hiện có 9 trace công khai trong thư mục `artifacts/traces_grading_question/` cho bộ `grading_questions.json`. Dựa trên các trace này, raw score hiện tại được ước tính khoảng **58 / 96**. Đây là con số ước tính bảo thủ: các câu `gq04`, `gq05`, `gq06`, `gq07`, `gq08`, `gq10` làm khá tốt; `gq03` và `gq09` fail rõ; `gq01` gần đúng nhưng thiếu PagerDuty; `gq02` chưa xuất hiện trong artifact grading hiện tại nên chưa thể tính là pass.

**Câu pipeline xử lý tốt nhất:**

- ID: `gq10` — Lý do tốt: trace [run_20260414_175222.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces_grading_question/run_20260414_175222.json:1) trả đúng ngoại lệ Flash Sale, không bị đánh lừa bởi điều kiện “lỗi nhà sản xuất”, và cite đúng `policy_refund_v4.txt`.

**Câu pipeline fail hoặc partial:**

- ID: `gq09` — Fail ở đâu: câu multi-hop SLA + Level 2 emergency access trả về abstain.  
Root cause: route đúng vào `policy_tool_worker`, nhưng nhánh policy-tool chưa có guard/fallback đủ chặt khi chuỗi tool call trả về output rỗng hoặc lỗi trung gian, nên `policy_result` bị lỗi và pipeline rơi về hướng abstain trong [run_20260414_175221.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces_grading_question/run_20260414_175221.json:1).

**Câu gq07 (abstain):** Nhóm xử lý khá đúng. Trace [run_20260414_175216.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/traces_grading_question/run_20260414_175216.json:1) trả lời rõ “không đủ thông tin trong tài liệu nội bộ” và không bịa mức phạt tài chính.

**Câu gq09 (multi-hop khó nhất):** Trace có 2 workers là `policy_tool_worker -> synthesis_worker`, đúng với kiến trúc mong muốn. Tuy nhiên kết quả cuối vẫn fail do policy path không lấy được tool output hợp lệ, nên `synthesis_worker` rơi về câu trả lời “Không đủ thông tin trong tài liệu nội bộ.”

---

## 4. So sánh Day 08 vs Day 09 — Điều nhóm quan sát được

Metric thay đổi rõ nhất không phải accuracy mà là **routing visibility và khả năng debug**. Day 09 hiện có `supervisor_route`, `route_reason`, `worker_io_logs`, `mcp_tools_used`, trong khi Day 08 không có lớp trace theo node. Về quality, Day 08 variant hiện vẫn mạnh hơn trên scorecard với `Faithfulness 5.0/5`, `Context Recall 5.0/5`, `Completeness 3.9/5` ở [scorecard_variant.md](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day08/lab/results/scorecard_variant.md:20). Ngược lại, Day 09 trên batch trace mới nhất có `avg_confidence = 0.763`, `avg_latency_ms = 4040`, `mcp_usage_rate = 2/12`, `hitl_rate = 0/12` ở [eval_report.json](/Users/quanliver/Projects/AI_Vin_Learner/Lecture-Day-08-09-10/day09/lab/artifacts/eval_report.json:1), nhưng chưa có metric cùng loại để chứng minh thắng về correctness.

Điều bất ngờ nhất khi chuyển từ single sang multi-agent là router của Day 09 thực ra đã khá ổn: trong batch 12 trace mới nhất, `12/12` route đều đúng theo intent hiện tại. Điểm nghẽn không nằm ở supervisor mà ở execution downstream, nhất là policy path. Trường hợp multi-agent chưa giúp ích là các câu fact đơn; với những câu như SLA P1, remote policy hay refund days, retrieval-only path là đủ và nhanh hơn. Day 09 chỉ thật sự phát huy khi policy path ổn định, còn ở trạng thái hiện tại thì orchestration tốt hơn Day 08 nhưng execution vẫn chưa theo kịp.

---

## 5. Phân công và đánh giá nhóm

**Phân công thực tế:**

| Tên | Vai trò | Phần đã làm | Sprint |
|-----|---------|-------------|--------|
| Lê Hồng Quân | Tech Lead, Supervisor Owner, Documentation Owner | `graph.py`, routing logic, state management; review kiến trúc + tổng hợp báo cáo | 1, 4 |
| Đoàn Sĩ Linh | Worker Owner | `workers/policy_tool.py`, `workers/synthesis.py`, `contracts/worker_contracts.yaml` | 2 |
| Phạm Thanh Lam | Retrieval Owner | `workers/retrieval.py` (dense/hybrid + rerank, logging retrieval metadata) | 2 |
| Dương Trung Hiếu | MCP Owner | `mcp_server.py`, integration MCP tools trong policy path | 3 |
| Nguyễn Đức Hải | Trace Owner | `eval_trace.py`, tổng hợp artifacts/trace phục vụ docs và grading | 4 |


**Điều nhóm làm tốt:**

Nhóm chia việc theo pipeline nên khá rõ ranh giới trách nhiệm. Cách tổ chức này đặc biệt hữu ích khi debug, vì mỗi người có thể nhìn đúng lớp mình phụ trách thay vì tất cả cùng sửa một prompt lớn. Việc có contract và trace riêng cho từng worker cũng giúp các quyết định kỹ thuật sau đó nhất quán hơn.

**Điều nhóm làm chưa tốt hoặc gặp vấn đề về phối hợp:**

Điểm chưa tốt nhất là nhóm chưa harden đủ policy path ở tầng xử lý lỗi và dữ liệu trung gian. Router và docs đã được cập nhật đúng logic mới, nhưng policy path vẫn có thể fail khi tool output không đầy đủ hoặc không đồng nhất schema. Điều này cho thấy nhóm đã tiến nhanh ở tầng kiến trúc nhưng chưa kiểm soát tốt độ ổn định end-to-end của toàn pipeline.

**Nếu làm lại, nhóm sẽ thay đổi gì trong cách tổ chức?**

Nhóm sẽ thêm một bước “integration hardening” sớm hơn, ngay sau khi xong MCP layer: kiểm tra schema/guard cho toàn bộ tool output và chạy smoke test riêng cho các câu multi-hop trước khi làm docs/report. Làm vậy sẽ tránh tình huống route đúng nhưng policy_result vẫn vỡ ở giữa pipeline.

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì?

Nếu có thêm 1 ngày, nhóm sẽ ưu tiên 3 cải tiến kỹ thuật để tăng chất lượng hệ thống:

1. Ổn định policy-tool path: chuẩn hóa schema cho tool output và bổ sung fallback theo trạng thái (`success/partial/failed`) để multi-hop access + incident luôn có đường xử lý an toàn ở `gq03/gq09`.
2. Tăng recall cho multi-hop: trong `policy_tool_worker`, bắt buộc source coverage cho cả 2 domain (SLA + access), thiếu domain thì tự retry bằng sub-query rồi merge/rerank trước khi kết luận policy.
3. Siết synthesis theo rule: với `incident_access`, dùng deterministic answer checklist bắt buộc (3 kênh notify gồm PagerDuty, deadline escalation 10 phút -> Senior Engineer, điều kiện Level 2 emergency access với Line Manager + IT Admin on-call) để giảm thiếu ý như `gq01` và drift ở `gq09`.

Sau 3 thay đổi trên, nhóm sẽ chạy regression mini (`gq01`, `gq03`, `gq09`) trước khi re-run full `grading_questions.json` để xác nhận cải thiện thực sự.

---
