# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Nguyễn Đức Hải
**Vai trò trong nhóm:** Worker Owner (synthesis.py)
**Ngày nộp:** 14/04/2026
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `workers/synthesis.py`
- Functions tôi trực tiếp implement: `_call_llm` (đảm nhiệm gọi Generative AI), `_estimate_confidence` (LLM-as-judge đánh giá độ tin cậy của câu trả lời) và `_build_context` (tổng hợp tài liệu và cảnh báo chính sách vào cấu trúc prompt).

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Tôi quản lý Node ở điểm hội tụ cuối cùng của quy trình đồ thị (`synthesis_worker_node`). State dictionary từ `graph.py` — chứa `retrieved_chunks` (do `retrieval_worker` nạp) và cờ `policy_result` (do `policy_tool_worker` kiểm tra kèm danh sách exceptions) — sẽ được tôi nhận kết hợp vào prompt. Sau khi tổng hợp, Worker của tôi gắn định dạng cuối cùng cho `final_answer` (kèm theo nguồn trích dẫn) và đẩy lên cho người dùng. Nếu nội dung truyền vào của tôi yếu, mọi quy trình routing trước đó của công cụ sẽ bị lãng phí vì kết quả bị sai lệch (hallucinate).

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**
Trong `synthesis.py`, tôi thiết kế rule khắt khe chặn Hallucination ở Prompt hệ thống đáp ứng theo yêu cầu chấm điểm `SCORING.md`:
```python
SYSTEM_PROMPT = """Bạn là trợ lý IT Helpdesk nội bộ.
Quy tắc nghiêm ngặt:
1. CHỈ trả lời dựa vào context được cung cấp. KHÔNG dùng kiến thức ngoài.
2. Nếu context không đủ để trả lời → nói rõ "Không đủ thông tin trong tài liệu nội bộ".
3. Trích dẫn nguồn cuối mỗi câu quan trọng: [tên_file].
..."""
```

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** 
Sử dụng phương pháp "LLM-as-Judge" làm cơ chế chấm điểm `confidence score` cho câu trả lời ở hàm `_estimate_confidence`, thay vì chỉ tính trung bình điểm cosine-similarity (distance) của quá trình truy vấn tài liệu.

**Lý do:**
Chỉ số `score` vector search chỉ biểu thị "đoạn tài liệu có chứa từ khóa gần giống câu hỏi hay không". Nó không thực sự phản ánh liệu nội dung câu trả lời cuối cùng có logic hay vi phạm chính sách của công ty không. Việc gọi một lượt LLM-as-judge để chấm trực tiếp mức độ tự tin (0.0 - 1.0) sẽ hiểu ngầm được sự mâu thuẫn giữa "Câu hỏi" và "Chính sách ngoại lệ" do `policy_tool` ném ra. Ngoài ra, LLM-as-judge thoả mãn trực tiếp điểm Bonus Sprint 4 trong `SCORING`. 

**Trade-off đã chấp nhận:**
Chấp nhận pipeline sẽ tăng độ trễ (latency) khoảng 700ms - 1s do hệ thống phải chờ thêm 1 block thời gian API gọi LLM chấm điểm bổ sung trước khi trả kết quả cuối cùng.

**Bằng chứng từ trace/code:**
```python
      # Prompt đánh giá độc lập bên trong _estimate_confidence
      prompt = f"""Đánh giá mức độ tự tin (confidence score) từ 0.0 đến 1.0 cho câu trả lời sau dựa trên tài liệu.
      Tài liệu: {context_text}
      Câu trả lời: {answer}
      Chỉ trả về chuỗi JSON định dạng: {{"confidence": 0.85}}"""
      ...
      # Tính Penalty trừ điểm cực mạnh nếu dính policy exception nhằm đảm bảo an toàn
      exception_penalty = 0.05 * len(policy_result.get("exceptions_found", []))
      return round(max(0.1, min(0.95, llm_conf - exception_penalty)), 2)
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Trả lời cho khách hàng là "Được hoàn tiền" dù vi phạm ngoại lệ chính sách (Missing contextual propagation). Bỏ quên logic kiểm duyệt từ `policy_tool`.

**Symptom (pipeline làm gì sai?):**
Trong quá trình test trace qua file `eval_trace.py` với câu hỏi dạng "Khách hàng Flash Sale yêu cầu hoàn tiền", Output luôn cho ra kết quả "Đồng ý hoàn tiền". LLM bị mâu thuẫn context nên trả lời máy móc và quên không ghi kèm cảnh báo ngoại lệ dù `route_reason` đã route trúng policy. 

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):**
Lỗi nằm ở phía worker logic trong hàm `_build_context` của `synthesis.py`. Hàm của tôi khi đó chỉ nhận `retrieved_chunks` để ráp thành string tài liệu, nhưng lại chưa nối đè `exceptions_found` từ object `policy_result` của nhóm bạn làm MCP chuyển đến. LLM bị "mù" context rủi ro chính sách.

**Cách sửa:**
Tôi đã update lại hàm `_build_context`, ép nối chuỗi cảnh báo cứng ngay trước mắt LLM nhằm buộc Prompt nhắc nhở LLM về các ngoại lệ. 
```python
    if policy_result and policy_result.get("exceptions_found"):
        parts.append("\\n=== POLICY EXCEPTIONS ===")
        for ex in policy_result["exceptions_found"]:
            parts.append(f"- {ex.get('rule', '')}")
```

**Bằng chứng trước/sau:**
> Trước: Trace gq02 trả về đáp án: "Khách hàng được hoàn tiền trong vòng 7 ngày [policy_refund_v4.txt]." (Sai hoàn toàn chính sách).
> Sau: Hàm nối đoạn text "=== POLICY EXCEPTIONS ===", Trace now: "Đơn hàng của quý khách thuộc dòng sản phẩm Flash Sale nên không hỗ trợ hoàn tiền theo Điều 3 chính sách [policy_refund_v4.txt]." 

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tổ chức code và thiết kế format Context chặt chẽ linh hoạt. Hệ thống Synthesis của tôi hiện xử lý mượt mà mọi nguồn dữ liệu tổng hợp từ các điểm nghẽn và đưa ra câu trả lời tự nhiên, có nguồn dẫn `[tên_file]`, đáp ứng trọn vẹn barem rubic điểm Sprint 2.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Phần LLM-as-judge hơi dài dòng khiến token bị lạm dụng nếu câu hỏi và response lớn. Cơ chế Regex JSON từ parse API chưa bắt chặt mọi trường hợp lỗi, thỉnh thoảng dễ bị crash nếu LLM chèn Markdown backticks xen giữa text.

**Nhóm phụ thuộc vào tôi ở đâu?**
Sự gắn kết của cả hệ thống Supervisor đều kết thúc tại `synthesis.py`. Dữ liệu Output của Node Synthesis là kết quả cuối mà Graph kết xuất vào biến `final_answer`. Nếu prompt của LLM fail, toàn bộ quá trình Agent Graph đi trước coi như hỏng.

**Phần tôi phụ thuộc vào thành viên khác:**
Tôi cực kỳ phụ thuộc vào Contract của Worker `policy_tool` và chất lượng văn bản của biến `retrieved_chunks` do bạn đảm nhiệm `retrieval` thực hiện. Nếu các object trả về thiếu key `.source` thì kết quả sẽ không có nguồn dẫn chiếu.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Nếu có thêm 2 tiếng, tôi sẽ viết lại toàn bộ `synthesis.py` để sử dụng Structured Outputs với JSON Schema (qua Pydantic) khi gọi LLM cho hàm `analyze_policy` cũng như `_estimate_confidence`. Việc ép cứng schema đầu ra sẽ xóa bỏ điểm yếu Regex của tôi hiện tại, giúp pipeline tin cậy 100% trong việc return đúng cấu trúc JSON, loại trừ hẳn rủi ro "hallucinate text block" và làm flow chấm điểm được nhanh hơn so với chế độ generate text thông thường.
