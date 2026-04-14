# Routing Decisions Log — Lab Day 09

**Nhóm:** ___________  
**Ngày:** 2026-04-14

> Tài liệu này tổng hợp các quyết định routing thực tế từ batch trace mới nhất trong `artifacts/traces/` (`run_20260414_180401.json` đến `run_20260414_180447.json`).
> Lưu ý: `eval_report.json` đếm theo `workers_called`, nên luôn có `synthesis_worker`. Phần dưới đây dùng `supervisor_route` trong trace để phân tích routing thực tế của supervisor.

---

## Routing Decision #1

**Task đầu vào:**
> SLA xử lý ticket P1 là bao lâu?

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `matched incident/SLA keywords: p1, sla, ticket`  
**MCP tools được gọi:** Không có  
**Workers called sequence:** `retrieval_worker -> synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): trả đúng SLA P1 gồm phản hồi ban đầu `15 phút`, xử lý `4 giờ`, và escalation nếu không phản hồi trong `10 phút`
- confidence: `0.95`
- Correct routing? Yes

**Nhận xét:** Đây là case factual, single-document điển hình. Route thẳng sang retrieval là hợp lý vì không cần tool reasoning hay policy inference. Pipeline ngắn, ít rủi ro và answer ra đúng.

---

## Routing Decision #2

**Task đầu vào:**
> Khách hàng có thể yêu cầu hoàn tiền trong bao nhiêu ngày?

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `matched refund factual query: hoàn tiền`  
**MCP tools được gọi:** Không có  
**Workers called sequence:** `retrieval_worker -> synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): trả đúng mốc `7 ngày làm việc` và bám tài liệu refund
- confidence: `0.9`
- Correct routing? Yes

**Nhận xét:** Đây là thay đổi quan trọng so với router cũ. Refund không còn mặc định đi `policy_tool_worker`; chỉ refund eligibility/exception mới đi policy path. Với factual refund query, retrieval-only vừa nhanh hơn vừa ổn định hơn.

---

## Routing Decision #3

**Task đầu vào:**
> ERR-403-AUTH là lỗi gì và cách xử lý?

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `no explicit policy/error keyword → fallback retrieval | risk_high via: err-`  
**MCP tools được gọi:** Không có  
**Workers called sequence:** `retrieval_worker -> synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): `Không đủ thông tin trong tài liệu nội bộ.`
- confidence: `0.3`
- Correct routing? Yes

**Nhận xét:** Supervisor hiện không route `ERR-*` sang `human_review` nữa. Thay vào đó, query được đánh dấu `risk_high=True` nhưng vẫn đi retrieval-first để ưu tiên abstain đúng nếu KB không có dữ liệu. Với batch hiện tại, đây là hành vi hợp lý hơn route HITL placeholder.

---

## Routing Decision #4

**Task đầu vào:**
> Ticket P1 lúc 2am. Cần cấp Level 2 access tạm thời cho contractor để thực hiện emergency fix. Đồng thời cần notify stakeholders theo SLA. Nêu đủ cả hai quy trình.

**Worker được chọn:** `policy_tool_worker`  
**Route reason:** `multi-hop access + incident query: access=access, level 2, contractor | incident=p1, sla, ticket | risk_high via: emergency, 2am`  
**MCP tools được gọi:** `search_kb` x3, `check_access_permission`, `get_ticket_info`  
**Workers called sequence:** `policy_tool_worker -> synthesis_worker`

**Kết quả thực tế:**
- final_answer (ngắn): `Không đủ thông tin trong tài liệu nội bộ.`
- confidence: `0.3`
- Correct routing? Yes, nhưng execution downstream fail

**Nhận xét:** Đây là case routing khó nhất và cũng cho thấy điểm mạnh của supervisor hiện tại. Route sang `policy_tool_worker` là đúng về mặt intent vì câu hỏi vừa cần access approval, vừa cần SLA/notification. Vấn đề không nằm ở router mà nằm ở runtime của policy path: MCP calls fail do dependency `fastapi`, khiến `policy_result` lỗi và synthesis phải abstain.

---

## Tổng kết

### Routing Distribution

Trong batch 12 trace mới nhất, nếu đếm theo `supervisor_route`:

| Worker | Số câu được route | % tổng |
|--------|------------------|--------|
| retrieval_worker | 10 | 83% |
| policy_tool_worker | 2 | 17% |
| human_review | 0 | 0% |

### Routing Accuracy

- Câu route đúng theo intent hiện tại: `12 / 12`
- Câu route sai: `0`
- Câu trigger HITL: `0`
- Câu fail do downstream runtime, không phải do route: `2` câu multi-hop policy path

### Lesson Learned về Routing

1. Keyword router đơn giản nhưng đủ dùng nếu chia rõ giữa `factual retrieval` và `policy/multi-hop reasoning`.
2. Over-routing sang `policy_tool_worker` làm pipeline dễ fail hơn; refund factual và incident factual nên ưu tiên `retrieval_worker`.
3. `risk_high` không đồng nghĩa với `human_review`. Trong batch hiện tại, đánh dấu risk để trace/debug hữu ích hơn là ép route sang HITL placeholder.

### Route Reason Quality

`route_reason` hiện đủ tốt để debug nhanh vì nó ghi rõ nhóm keyword nào đã match, ví dụ `matched incident/SLA keywords` hoặc `multi-hop access + incident query`. Điểm có thể cải tiến thêm là ghi rõ hơn loại query như `refund_factual`, `refund_policy`, `access_policy`, `incident_multi_hop` để việc nhóm lỗi theo category dễ hơn trong evaluator và report.
