# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-<MSSV>`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: `latency_ms <= 3000ms` ở mức P95
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` duy trì trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ lâu hơn bình thường để nhận câu trả lời từ AI
- Ba bước kiểm tra đầu tiên:
  1. Mở Panel Latency trên Dashboard xác định khoảng thời gian P95 tăng đột biến.
  2. Lọc `data/logs.jsonl` tìm `correlation_id` của các request có `latency_ms` > 3000ms.
  3. Tra cứu `correlation_id` trên Langfuse Cloud để soi chi tiết span `retrieval` hoặc `generation` gây chậm.
- Mitigation tạm thời: Rollback prompt version về v1 nếu vừa nâng cấp prompt, hoặc giảm số lượng câu truy vấn song song.
- Owner: `student-2A202602992`

## Alert 2

- Tên: `HighErrorRate`
- Severity: `critical`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỷ lệ request thành công (Success Rate >= 98%)
- Điều kiện và thời gian duy trì: `error_rate_pct > 2%` duy trì trong 3 phút
- Ảnh hưởng tới người dùng: Người dùng nhận thông báo lỗi hệ thống 500 khi gửi câu hỏi
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra Panel Errors trên Dashboard để thấy tỷ lệ lỗi tăng vọt.
  2. Lọc `data/logs.jsonl` theo `level="error"` hoặc `event="request_failed"`, lấy `correlation_id` và `error_type`.
  3. Mở Trace trên Langfuse Cloud theo `correlation_id` để kiểm tra span lỗi.
- Mitigation tạm thời: Tắt bớt các tính năng thực nghiệm, kiểm tra lại kết nối dịch vụ RAG/LLM hoặc restart API server.
- Owner: `student-2A202602992`

## Alert 3

- Tên: `RetrievalFailureRateHigh`
- Severity: `critical`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Retrieval Success Rate >= 90%
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90%` duy trì trong 3 phút
- Ảnh hưởng tới người dùng: AI không tìm thấy ngữ cảnh tài liệu và trả lời câu hỏi bằng thông tin chung (fallback)
- Ba bước kiểm tra đầu tiên:
  1. Mở Panel Errors / Quality trên Dashboard kiểm tra tỷ lệ thất bại của bước RAG.
  2. Lọc `data/logs.jsonl` tìm log có `tool_name="retrieval"` và `tool_success=False`.
  3. Mở Trace trên Langfuse Cloud xem chi tiết span `retrieval` để xác định lỗi từ Vector store.
- Mitigation tạm thời: Khôi phục kết nối Vector DB hoặc tạm thời cho phép fallback an toàn trong khi chờ khắc phục DB.
- Owner: `student-2A202602992`
