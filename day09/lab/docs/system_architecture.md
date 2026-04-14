# System Architecture — Lab Day 09

**Nhóm:** ___________  
**Ngày:** ___________  
**Version:** 1.0

---

## 1. Tổng quan kiến trúc

> Mô tả ngắn hệ thống của nhóm: chọn pattern gì, gồm những thành phần nào.

**Pattern đã chọn:** Supervisor-Worker  
**Lý do chọn pattern này (thay vì single agent):**

_________________

---

## 2. Sơ đồ Pipeline

> Vẽ sơ đồ pipeline dưới dạng text, Mermaid diagram, hoặc ASCII art.
> Yêu cầu tối thiểu: thể hiện rõ luồng từ input → supervisor → workers → output.

**Ví dụ (ASCII art):**
```
User Request
     │
     ▼
┌──────────────┐
│  Supervisor  │  ← route_reason, risk_high, needs_tool
└──────┬───────┘
       │
   [route_decision]
       │
  ┌────┴────────────────────┐
  │                         │
  ▼                         ▼
Retrieval Worker     Policy Tool Worker
  (evidence)           (policy check + MCP)
  │                         │
  └─────────┬───────────────┘
            │
            ▼
      Synthesis Worker
        (answer + cite)
            │
            ▼
         Output
```

**Sơ đồ thực tế của nhóm:**

```
User Query
    │
    ▼
┌──────────────────────────────┐
│ InternalSupportSupervisor    │
│ (graph.py)                   │
│ - phân tích intent/domain    │
│ - set route_reason           │
│ - set risk_high / needs_tool │
└──────────────┬───────────────┘
               │
               ▼
      [Domain-based Routing]
               │
   ┌───────────┼─────────────────────────────┐
   │           │                             │
   │           │                             │
   ▼           ▼                             ▼
PolicyDomain  IncidentSupportDomain      ITHelpdeskDomain
(refund /     (P1 / SLA / escalation /   (password / VPN /
access / HR)   ticket / on-call)          FAQ / common IT)
   │           │                             │
   └───────────┴──────────────┬──────────────┘
                              │
                              ▼
                 Shared Retrieval Worker
                 (workers/retrieval.py)
                 - lấy evidence từ ChromaDB
                 - trả retrieved_chunks/sources
                              │
                              ▼
                   Policy/Tool Worker
                 (workers/policy_tool.py)
          - xử lý policy / exception / access rule
          - gọi MCP khi cần:
            • search_kb
            • get_ticket_info
            • check_access_permission
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
        Human Review                    Synthesis Worker
      (nếu risk cao /                   (workers/synthesis.py)
       domain không rõ)                - tổng hợp answer
               │                        - cite sources
               └──────────────┬─────────┘
                              │
                              ▼
                    Final Answer + Sources
                       + Confidence + Trace
```

---

## 3. Vai trò từng thành phần

### Supervisor (`graph.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Phân tích câu hỏi đầu vào, xác định domain chính, quyết định route sang worker phù hợp, đồng thời gắn `route_reason`, `risk_high`, `needs_tool` vào shared state |
| **Input** | `task` từ user query và shared `AgentState` |
| **Output** | supervisor_route, route_reason, risk_high, needs_tool |
| **Routing logic** | Query chứa `refund`, `flash sale`, `license`, `access`, `level 3` → ưu tiên `policy_tool_worker`; query chứa `P1`, `SLA`, `ticket`, `escalation` → ưu tiên `retrieval_worker`; query chứa `ERR-` hoặc task rủi ro cao → có thể route `human_review` |
| **HITL condition** | Khi `risk_high=True` và query chứa mã lỗi hoặc trường hợp mơ hồ cần human review |

### Retrieval Worker (`workers/retrieval.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Tìm evidence từ ChromaDB, trả về `retrieved_chunks` và `retrieved_sources` cho các worker phía sau |
| **Embedding model** | Ưu tiên `all-MiniLM-L6-v2` qua Sentence Transformers; fallback sang OpenAI `text-embedding-3-small`; cuối cùng mới fallback random embedding để test |
| **Top-k** | Mặc định `3` (`DEFAULT_TOP_K = 3`) |
| **Stateless?** | Yes |

### Policy Tool Worker (`workers/policy_tool.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Phân tích policy/rule dựa trên retrieved chunks, phát hiện exception, temporal note, và gọi MCP tools khi cần bổ sung thông tin |
| **MCP tools gọi** | `search_kb`, `get_ticket_info`, `check_access_permission` |
| **Exception cases xử lý** | `flash_sale_exception`, `digital_product_exception`, `activated_exception`, và note cho các câu có temporal scoping trước `01/02/2026` |

### Synthesis Worker (`workers/synthesis.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **LLM model** | Mặc định `gpt-4o-mini`; fallback `gemini-1.5-flash` nếu dùng Google API |
| **Temperature** | `0.1` trong nhánh OpenAI |
| **Grounding strategy** | Build context từ `retrieved_chunks` và `policy_result`, sau đó prompt ép model chỉ trả lời từ context, nêu exception trước kết luận, và có citation theo tên file |
| **Abstain condition** | Nếu không có đủ context thì prompt yêu cầu trả lời `Không đủ thông tin trong tài liệu nội bộ`; confidence cũng bị hạ thấp khi thiếu evidence |

### MCP Server (`mcp_server.py`)

| Tool | Input | Output |
|------|-------|--------|
| search_kb | query, top_k | chunks, sources |
| get_ticket_info | ticket_id | ticket details |
| check_access_permission | access_level, requester_role | can_grant, approvers |
| create_ticket | priority, title, description | mock ticket_id, url, created_at |

---

## 4. Shared State Schema

> Liệt kê các fields trong AgentState và ý nghĩa của từng field.

| Field | Type | Mô tả | Ai đọc/ghi |
|-------|------|-------|-----------|
| task | str | Câu hỏi đầu vào | supervisor đọc |
| supervisor_route | str | Worker được chọn | supervisor ghi |
| route_reason | str | Lý do route | supervisor ghi |
| retrieved_chunks | list | Evidence từ retrieval | retrieval ghi, synthesis đọc |
| policy_result | dict | Kết quả kiểm tra policy | policy_tool ghi, synthesis đọc |
| mcp_tools_used | list | Tool calls đã thực hiện | policy_tool ghi |
| final_answer | str | Câu trả lời cuối | synthesis ghi |
| confidence | float | Mức tin cậy | synthesis ghi |
| risk_high | bool | Cờ đánh dấu query có rủi ro cao, cần route cẩn thận hơn | supervisor ghi, human_review đọc |
| needs_tool | bool | Cờ cho biết worker có được/ cần gọi MCP tool hay không | supervisor ghi, policy_tool đọc |
| hitl_triggered | bool | Cho biết pipeline đã đi qua human review chưa | human_review ghi, eval/trace đọc |
| retrieved_sources | list | Danh sách source duy nhất lấy được từ retrieval | retrieval ghi, synthesis/eval đọc |
| sources | list | Nguồn cuối cùng được cite trong answer | synthesis ghi |
| history | list | Nhật ký text theo từng bước trong graph | supervisor và mọi worker ghi |
| workers_called | list | Danh sách worker/node đã được gọi trong một run | supervisor/worker ghi, eval_trace đọc |
| latency_ms | int hoặc null | Thời gian chạy toàn pipeline tính bằng mili-giây | graph ghi, eval_trace đọc |
| run_id | str | ID duy nhất của mỗi run, dùng để lưu trace | graph ghi, save_trace đọc |
| worker_io_logs | list | Log input/output/error của từng worker để phục vụ trace/debug | retrieval/policy_tool/synthesis ghi |

---

## 5. Lý do chọn Supervisor-Worker so với Single Agent (Day 08)

| Tiêu chí | Single Agent (Day 08) | Supervisor-Worker (Day 09) |
|----------|----------------------|--------------------------|
| Debug khi sai | Khó — không rõ lỗi ở đâu | Dễ hơn — test từng worker độc lập |
| Thêm capability mới | Phải sửa toàn prompt | Thêm worker/MCP tool riêng |
| Routing visibility | Không có | Có route_reason trong trace |
| ___________________ | ___________________ | ___________________ |

**Nhóm điền thêm quan sát từ thực tế lab:**

_________________

---

## 6. Giới hạn và điểm cần cải tiến

> Nhóm mô tả những điểm hạn chế của kiến trúc hiện tại.

1. ___________________
2. ___________________
3. ___________________
