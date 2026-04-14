# Single Agent vs Multi-Agent Comparison — Lab Day 09

**Nhóm:** ___________  
**Ngày:** 2026-04-14

> So sánh này dùng số liệu thực tế đang có:
> - Day 08: `results/scorecard_variant.md`
> - Day 09: `artifacts/eval_report.json` và batch 12 trace mới nhất (`run_20260414_180401.json` đến `run_20260414_180447.json`)
>
> Hai hệ chưa dùng cùng một bộ metric, nên vài dòng bên dưới được ghi `N/A` hoặc `proxy` để tránh so sánh sai bản chất.

---

## 1. Metrics Comparison

| Metric | Day 08 (Single Agent) | Day 09 (Multi-Agent) | Delta | Ghi chú |
|--------|----------------------|---------------------|-------|---------|
| Avg confidence | N/A | `0.763` | N/A | Day 08 scorecard không chấm theo confidence |
| Avg latency (ms) | N/A | `4040` | N/A | Day 08 không có số latency trong scorecard |
| Abstain rate (%) | N/A | `25%` | N/A | Day 09: `3/12` trace mới nhất abstain |
| Multi-hop accuracy | N/A | `0/2` (proxy) | N/A | Proxy tính trên 2 trace policy multi-hop mới nhất |
| Faithfulness | `5.0 / 5` | N/A | N/A | Day 09 chưa có metric cùng loại |
| Relevance | `4.8 / 5` | N/A | N/A | Day 09 chưa có metric cùng loại |
| Context Recall | `5.0 / 5` | N/A | N/A | Day 09 chưa có metric cùng loại |
| Completeness | `3.9 / 5` | N/A | N/A | Day 09 chưa có metric cùng loại |
| Routing visibility | ✗ Không có | ✓ Có `supervisor_route`, `route_reason` | N/A | Day 09 dễ debug hơn rõ rệt |
| Debug time (estimate) | N/A | N/A | N/A | Nhóm chưa đo thời gian debug bằng stopwatch |

**Kết luận ngắn:** Day 09 mạnh hơn về observability và phân lớp trách nhiệm, nhưng với batch hiện tại vẫn chưa chứng minh được chất lượng answer tốt hơn Day 08.

---

## 2. Phân tích theo loại câu hỏi

### 2.1 Câu hỏi đơn giản (single-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | Tốt trên scorecard variant | Tốt trên batch retrieval-only hiện tại |
| Latency | N/A | Khoảng 4 giây trung bình toàn batch |
| Observation | Single-agent đủ mạnh cho FAQ/policy fact đơn | Retrieval-only path của Day 09 trả lời ổn các câu như SLA P1, refund days, lockout, remote policy |

**Kết luận:** Với câu đơn tài liệu, multi-agent chưa cho thấy cải thiện chất lượng rõ ràng. Lợi ích chính ở đây là trace dễ đọc hơn, không phải answer tốt hơn. Nếu câu hỏi chỉ cần một nguồn, Day 08 hoặc retrieval-only path của Day 09 đều đủ dùng.

### 2.2 Câu hỏi multi-hop (cross-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | Tốt hơn trong scorecard variant | Đang yếu ở batch hiện tại |
| Routing visible? | ✗ | ✓ |
| Observation | Variant Day 08 retrieve đủ source và trả lời tốt hơn ở các câu access + incident như `gq06` | Day 09 route đúng sang `policy_tool_worker`, nhưng 2/2 trace multi-hop gần nhất fail do MCP runtime dependency rồi abstain |

**Kết luận:** Đây là điểm Day 09 đáng lẽ phải thắng, nhưng hiện chưa thắng. Router hiện đã nhận diện multi-hop tốt hơn, song policy path vẫn chưa ổn định bằng retrieval stack của Day 08. Nói ngắn gọn: orchestration đã tốt hơn, execution chưa theo kịp.

### 2.3 Câu hỏi cần abstain

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Abstain rate | N/A | `25%` |
| Hallucination cases | Thấp theo faithfulness score | Thấp ở retrieval-only, nhưng có abstain do runtime failure ở policy path |
| Observation | Day 08 thường abstain khi thiếu evidence thật | Day 09 có 1 abstain hợp lý (`ERR-403-AUTH`) và 2 abstain không mong muốn do MCP call fail |

**Kết luận:** Abstain của Day 09 hiện “mixed”. Một phần là abstain đúng do thiếu tài liệu, nhưng một phần là abstain do hỏng runtime chứ không phải do model biết tự dừng đúng lúc.

---

## 3. Debuggability Analysis

### Day 08 — Debug workflow

```text
Khi answer sai → phải đọc retrieval + generation flow của cùng một pipeline
Không có supervisor_route hay worker_io_logs theo node
Khó tách ngay lỗi nằm ở retrieval hay generation
```

### Day 09 — Debug workflow

```text
Khi answer sai → đọc trace → xem supervisor_route + route_reason
  → Nếu route sai → sửa supervisor routing logic
  → Nếu policy path fail → xem mcp_tools_used và worker_io_logs
  → Nếu answer drift → xem synthesis generation_mode và policy_result
```

**Câu cụ thể nhóm đã debug:**

- Trace `run_20260414_180447.json` cho thấy router làm đúng, nhưng `policy_tool_worker` fail do toàn bộ MCP calls báo `No module named 'fastapi'`, sau đó `policy_result` thành lỗi và synthesis chỉ còn abstain. Đây là ví dụ điển hình cho thấy Day 09 khoanh vùng bug nhanh hơn Day 08.

**Kết luận:** Về khả năng debug, Day 09 tốt hơn rõ rệt. Dù chưa thắng về quality, nó giúp nhóm biết chính xác bug nằm ở router, retrieval, MCP runtime hay synthesis.

---

## 4. Extensibility Analysis

| Scenario | Day 08 | Day 09 |
|---------|--------|--------|
| Thêm 1 tool/API mới | Phải sửa trực tiếp pipeline hoặc prompt | Thêm MCP tool và gọi từ `policy_tool_worker` |
| Thêm 1 domain mới | Thường phải mở rộng retrieval + prompt chung | Có thể thêm worker/domain path riêng |
| Thay đổi retrieval strategy | Sửa trực tiếp RAG pipeline | Sửa `retrieval_worker` và tái dùng qua `search_kb` |
| A/B test một phần | Khó vì các phần dính chặt nhau | Dễ hơn vì có thể thay router, worker hoặc synthesis độc lập |

**Nhận xét:** Day 09 tốt hơn hẳn về khả năng mở rộng. Việc tách `retrieval_worker`, `policy_tool_worker` và `synthesis_worker` giúp nhóm thay đổi một lớp mà không cần chạm toàn pipeline.

---

## 5. Cost & Latency Trade-off

| Scenario | Day 08 calls | Day 09 calls |
|---------|-------------|-------------|
| Simple query | 1 LLM call | 1 synthesis call, 0 MCP call |
| Complex query | 1 LLM call | 1 synthesis call + nhiều MCP call |
| MCP tool call | N/A | 5 MCP calls trong 2 trace multi-hop mới nhất |

**Nhận xét về cost-benefit:**

- Với query đơn giản, Day 09 chưa mang lại lợi ích chất lượng tương xứng với overhead orchestration.
- Với query phức tạp, Day 09 có tiềm năng cao hơn vì tách tool reasoning ra riêng, nhưng batch hiện tại cho thấy chi phí tăng mà output chưa tương xứng do policy path còn fail.

---

## 6. Kết luận

**Multi-agent tốt hơn single agent ở điểm nào?**

1. Quan sát và debug tốt hơn nhờ có `supervisor_route`, `route_reason`, `worker_io_logs`, `mcp_tools_used`.
2. Dễ mở rộng capability hơn, đặc biệt khi cần thêm tool call hoặc tách domain reasoning.

**Multi-agent kém hơn hoặc chưa tốt bằng ở điểm nào?**

1. Chất lượng answer hiện tại chưa vượt Day 08 ở các câu multi-hop; policy path còn phụ thuộc runtime MCP nên dễ fail.
2. Overhead vận hành cao hơn, trong khi với câu đơn tài liệu thì lợi ích chất lượng chưa rõ.

**Khi nào KHÔNG nên dùng multi-agent?**

Không nên dùng multi-agent cho các FAQ hoặc fact query đơn giản chỉ cần một nguồn tài liệu, ví dụ SLA cơ bản, remote policy, hay refund factual. Khi đó retrieval-only hoặc single-agent RAG gọn hơn và ít điểm hỏng hơn.

**Nếu tiếp tục phát triển hệ thống này, nhóm sẽ thêm gì?**

1. Gỡ phụ thuộc `fastapi/uvicorn` khỏi local MCP dispatch để policy path chạy ổn định trong mọi môi trường.
2. Hoàn thiện deterministic synthesis cho `incident_access` để answer luôn bám structured tool result thay vì rơi về abstain khi MCP lỗi.
3. Bổ sung bộ eval cùng metric với Day 08 để so sánh apples-to-apples thay vì phải dùng proxy.
