# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Dương Trung Hiếu
**Vai trò trong nhóm:** MCP Owner  
**Ngày nộp:** 14/04/2026
**Độ dài yêu cầu:** 500–800 từ

---

> **Lưu ý quan trọng:**
> - Viết ở ngôi **"tôi"**, gắn với chi tiết thật của phần bạn làm
> - Phải có **bằng chứng cụ thể**: tên file, đoạn code, kết quả trace, hoặc commit
> - Nội dung phân tích phải khác hoàn toàn với các thành viên trong nhóm
> - Deadline: Được commit **sau 18:00** (xem SCORING.md)
> - Lưu file với tên: `reports/individual/[ten_ban].md` (VD: `nguyen_van_a.md`)

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Trong Sprint 3 nhằm đạt điểm Bonus +2 với kiến trúc phân tán, tôi đảm nhận vai trò thiết kế và lập trình Model Context Protocol (MCP) Server. Thay vì để LLM Agent chạy tool cục bộ, tôi đưa toàn bộ logic công cụ lên một HTTP REST API độc lập bằng FastAPI.

**Module/file tôi chịu trách nhiệm:**
- File chính: `mcp_server.py`
- Functions tôi implement: `tool_search_kb, tool_get_ticket_info, tool_check_access_permission, tool_create_ticket, cùng hệ thống API Router (/tools/list và /tools/call).`

**Cách công việc của tôi kết nối với phần của thành viên khác:**

Tôi tạo ra "cánh tay" cho bộ não Agent. Cụ thể, khi Graph Agent (do Quang thiết kế) suy luận ra cần gọi tool, Client (do Linh viết trong policy_tool.py) sẽ đóng gói dữ liệu thành JSON và gọi lệnh POST đến localhost:8000/tools/call. Server của tôi sẽ nhận lệnh, gọi function xử lý (có kết hợp module retrieval của Lam), sau đó trả về kết quả JSON để Agent tiếp tục quy trình xử lý ticket.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**

Commit 1c9d0d4, Toàn bộ file mcp_server.py với việc thiết lập uvicorn.run("mcp_server:app", host="0.0.0.0", port=8000).

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi quyết định áp dụng cơ chế "Graceful Degradation / Fallback Mode" kết hợp với HTTP Error Mapping ngay tại hàm Dispatcher (/tools/call), thay vì để exception làm sập (crash) server khi có lỗi.

**Lý do:**

Đặc thù của Multi-Agent là các LLM rất hay "ảo giác" (hallucinate). Chúng có thể truyền thiếu tham số hoặc sai format JSON (Ví dụ: truyền top_k="ba" thay vì integer 3). Nếu tôi để server văng lỗi Python TypeError thông thường, quá trình POST request sẽ sập, kéo theo toàn bộ flow suy luận của Agent thất bại. Tôi chọn bọc logic trong try...except và trả về mã HTTP cụ thể (400 Bad Request cho lỗi type, 404 cho sai tên tool). Nhờ đó, Agent nhận được chuỗi text giải thích lỗi và có cơ hội tự kích hoạt luồng Self-Correction (Tự sửa lỗi) để tạo lại request đúng.

**Trade-off đã chấp nhận:**

Code của server trở nên dài hơn do phải bọc nhiều tầng exception và quản lý Pydantic validation chặt chẽ hơn, làm tăng chút overhead (khoảng 2-3ms) cho mỗi request.

**Bằng chứng từ trace/code:**

```
@app.post("/tools/call")
def dispatch_tool_api(request: ToolCallRequest):
    ...
    try:
        result = tool_fn(**tool_input)
        return {"tool_name": tool_name, "status": "success", "result": result}
    except TypeError as e:
        raise HTTPException(status_code=400, detail=f"Lỗi input: {e}")
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Lỗi thực thi: {e}")
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** LLM Agent bị văng lỗi TypeError: tool_check_access_permission() missing 1 required positional argument: 'is_emergency' khi truy vấn các SOP về phân quyền truy cập thông thường.

**Symptom (pipeline làm gì sai?):**

Khi nhận được câu hỏi không khẩn cấp, LLM quyết định rằng biến is_emergency là không cần thiết, nên nó tạo ra JSON payload chỉ chứa {"access_level": 2, "requester_role": "Developer"}. Khi JSON này gửi tới endpoint /tools/call và unpack **tool_input vào hàm python, server báo lỗi thiếu tham số và trả về lỗi 500, khiến Agent dừng quy trình và không thể trả lời user.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**

Trong Schema khai báo ban đầu, tôi không gán giá trị mặc định cho is_emergency ở cả tầng JSON Schema lẫn tầng tham số hàm Python.

**Cách sửa:**

Tôi cập nhật cấu trúc schema trong TOOL_SCHEMAS, thêm "default": False cho biến is_emergency. Quan trọng nhất, tôi gán explicitly default value ở signature của hàm Python.

**Bằng chứng trước/sau:**

Trước khi sửa:
def tool_check_access_permission(access_level: int, requester_role: str, is_emergency: bool) -> dict:
(Lỗi trace trả về HTTP 500 Internal Server Error)

Sau khi sửa:
def tool_check_access_permission(access_level: int, requester_role: str, is_emergency: bool = False) -> dict:
(Payload thiếu is_emergency vẫn chạy mượt mà, fallback về giá trị False, Agent tiếp tục sinh câu trả lời thành công).

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Tôi đã thiết kế kiến trúc API Server (MCP) rất chuẩn chỉ, bóc tách hoàn toàn logic công cụ (DB search, API gọi Jira) ra khỏi môi trường suy luận của mô hình ngôn ngữ lớn (LangGraph). Code có tính mở rộng cao, dễ dàng bổ sung tool mới vào TOOL_REGISTRY mà không phải đụng tới flow của Agent.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Dữ liệu Jira nội bộ (Tickets) vẫn đang dùng Dictionary thuần túy dạng Mocking (MOCK_TICKETS). Tôi chưa kịp làm bước kết nối tới một cơ sở dữ liệu thực hoặc một Jira Sandbox API để việc truy xuất sinh động hơn.

**Nhóm phụ thuộc vào tôi ở đâu?** _(Phần nào của hệ thống bị block nếu tôi chưa xong?)_

Sỹ Linh (policy_tool.py) và toàn bộ Graph của Quang phụ thuộc 100% vào việc server FastAPI của tôi phải đang chạy ổn định ở cổng 8000. Nếu API sập, Agent bị mù thông tin nội bộ.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_

Tôi cần hàm retrieve_dense của Phạm Thanh Lam chạy mượt mà để import ngược vào logic của tool search_kb, giúp tìm kiếm vector hóa trong tài liệu thực.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ thử viết một cơ chế Authentication (API Key) và Rate Limiting cho mcp_server.py. Hiện tại API mở hoàn toàn trên localhost. Nhìn vào file eval_trace.py khi hệ thống chạy đánh giá hàng loạt câu hỏi đồng thời, lượng POST request dội xuống server cùng lúc là khá lớn. Việc có Rate Limiting sẽ giúp bảo vệ API Database/Knowledge Base không bị quá tải khi nhiều Agents (Multi-Agent) đồng loạt gọi tool cùng một thời điểm trong môi trường Production.

---