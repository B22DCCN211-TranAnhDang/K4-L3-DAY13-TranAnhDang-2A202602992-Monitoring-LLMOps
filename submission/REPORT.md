# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Trần Anh Đăng
- **MSSV:** 2A202602992 / B22DCCN211
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/B22DCCN211-TranAnhDang/K4-L3-DAY13-TranAnhDang-2A202602992-Monitoring-LLMOps
- **Commit SHA cuối:**
- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602992`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 | **100/100** | Đã đạt 100/100, bổ sung đủ correlation_id, context enrichment & scrub PII |
| `validate_dashboard.py` | 6/6 | **6/6** | Hợp lệ 6/6 panel |
| `pytest` | 22/22 passed | **24/24 passed** | Đã viết thêm test cho CCCD & Thẻ ngân hàng |
| Số traces hợp lệ | 0 | | |
| Số PII leak | 0 | **0** | Đã scrub toàn bộ PII (Email, Phone VN, CCCD, Credit Card) |
| Latency P95 / TTFT P95 | ~476ms | ~517ms | Ghi nhận từ load_test |
| Retrieval success rate | 100% | 100% | 10/10 request thành công |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware` (`app/middleware.py`), kiểm tra header `x-request-id` từ client request. Nếu chưa có, tự động sinh mã mới dạng `req-<8-char-hex>` bằng `uuid.uuid4().hex[:8]`. Mã này sau đó được gán vào `structlog.contextvars.bind_contextvars(correlation_id=...)` và trả về client trong Header `x-request-id` cũng như `x-response-time-ms`.
- **Các metadata được ghi vào structured log:** Mỗi request API đều được tự động gán các thông tin ngữ cảnh gồm `user_id_hash` (mã hóa SHA256 12 ký tự), `session_id`, `feature`, `model` (`fake-llm-v1`), `env` (`dev`) cùng các chỉ số vận hành (`latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`).
- **Cách bảo đảm PII được scrub trước khi ghi:** Sử dụng bộ lọc PII (`scrub_event` trong `app/logging_config.py`) đính kèm trực tiếp vào chuỗi xử lý structlog processors. Tất cả thông tin Email, Số điện thoại Việt Nam, CCCD (12 chữ số), Thẻ tín dụng/ngân hàng (16 chữ số) trong payload/message đều bị thay thế bởi nhãn `[REDACTED_*]` trước khi ghi ra file `data/logs.jsonl`.
- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py` đạt điểm tuyệt đối **100/100**, kiểm tra file `data/logs.jsonl` đảm bảo đủ các trường bắt buộc và chạy `pytest` pass 24/24 unit tests.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Traces được ghi trực tiếp về project Langfuse Cloud cá nhân `day13-k4-l3b-2A202602992` sử dụng Public/Secret API Key cá nhân cấu hình trong `.env`.
- **Cấu trúc root/retrieval/generation observations:** Mỗi request tạo ra một cây Trace gồm Root observation `day13-agent-request` (chứa `lab-agent-run`), bên trong có 2 child observations: `retrieval` (loại `retriever` / `span` đo thời gian RAG) và `generation` (loại `generation` đo mô hình LLM, ghi nhận `model`, `usage` token in/out và `cost_usd`).
- **Cách nối trace với log:** Trường `correlation_id` (mã dạng `req-<8-hex>`) được đưa vào `metadata` của Langfuse trace và đính kèm đồng thời trong tất cả dòng JSON log của file `data/logs.jsonl`.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** `v1` (gắn label `baseline` và `production`)
- **Version/label candidate:** `v2` (gắn label `candidate`)
- **Trace ID của mỗi version:** Trace v1 ghi nhận `prompt_version=1`, Trace v2 ghi nhận `prompt_version=2` trong metadata.
- **Cách promote và rollback `production`:** Trên giao diện Langfuse Prompt Management, chuyển label `production` sang `v2` để promote; khi cần khôi phục chỉ cần chuyển lại nhãn `production` trỏ về `v1` mà không phải thay đổi mã nguồn ứng dụng.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Cấu hình chuẩn 6 panel trong [`config/dashboard.yaml`](file:///d:/Lab%20VinUni/K4-L3-DAY13-TranAnhDang-2A202602992-Monitoring-LLMOps/config/dashboard.yaml): Latency (P50/P95/P99, TTFT), Traffic (Request rate), Errors (Error rate, Retrieval success rate), Cost (Daily/Cumulative USD), Tokens (Input/Output token length), Quality (Average quality score proxy). `validate_dashboard.py` đạt 6/6 panel hợp lệ.
- **SLO và lý do chọn:** SLO chính chọn `99.5% request thành công và latency <= 3000ms trong 28 ngày`. Lý do: đảm bảo trải nghiệm người dùng không phải chờ quá 3 giây và duy trì độ tin cậy cao cho ứng dụng RAG/LLM.
- **Cách tính error budget:** SLO 99.5% trong cửa sổ 28 ngày tương ứng với Error Budget là `0.5%`. Giả sử hệ thống phục vụ 10,000 request trong 28 ngày thì tối đa 50 request được phép thất bại hoặc phản hồi chậm hơn 3000ms.
- **Ba alert và runbook tương ứng:**
  1. `HighLatencyP95` (Warning, `p95(latency_ms) > 3000ms` trong 5m) -> Runbook [`docs/alerts.md#alert-1`](file:///d:/Lab%20VinUni/K4-L3-DAY13-TranAnhDang-2A202602992-Monitoring-LLMOps/docs/alerts.md#alert-1).
  2. `HighErrorRate` (Critical, `error_rate_pct > 2%` trong 3m) -> Runbook [`docs/alerts.md#alert-2`](file:///d:/Lab%20VinUni/K4-L3-DAY13-TranAnhDang-2A202602992-Monitoring-LLMOps/docs/alerts.md#alert-2).
  3. `RetrievalFailureRateHigh` (Critical, `retrieval_success_rate_pct < 90%` trong 3m) -> Runbook [`docs/alerts.md#alert-3`](file:///d:/Lab%20VinUni/K4-L3-DAY13-TranAnhDang-2A202602992-Monitoring-LLMOps/docs/alerts.md#alert-3).

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** 2026-09-30 04:06:00Z - 04:07:00Z
- **Triệu chứng từ metrics:** Latency P95 phía Server vượt quá ngưỡng 2000ms (đạt ~2650ms) và Client Queue Latency vọt lên ~13.2s ở feature `monitoring` dưới tải `concurrency 5`.
- **Log line và correlation ID liên quan:** Event `response_sent` với `correlation_id=req-ff995f29`, `feature=monitoring`, `latency_ms=2650`, `service=api`.
- **Trace ID và span gây ảnh hưởng:** Trace có `correlation_id=req-ff995f29` trên Langfuse Cloud cho thấy span `retrieval` bị nghẽn tới 2500ms (vượt ngưỡng 2000ms), trong khi span `generation` của LLM chỉ mất 150ms.
- **Root cause:** Đề bài `day13-k4-l3b-monitoring-llmops-v1` kích hoạt sự cố `rag_slow` (Seed: 1312) làm chậm 2.5 giây ở bước Tìm kiếm tài liệu RAG (`app/mock_rag.py`).
- **Fix action:** Tắt sự cố bằng lệnh `python scripts/inject_incident.py --scenario rag_slow --disable` để khôi phục hiệu năng tìm kiếm RAG.
- **Preventive measure:** Cấu hình Alert `HighLatencyP95` (khi P95 > 2000ms duy trì 5m) kết hợp cơ chế Timeout 1.5s cho RAG retrieval kèm Fallback answer.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Đính kèm `correlation_id` đồng thời vào Structlog ContextVars và Langfuse Trace Metadata để tạo liên kết 1-1 trực tiếp giữa Log và Trace.
- **Một lỗi/blocker đã gặp:** Gặp xung đột giải quyết phụ thuộc (dependency resolution) của `opentelemetry` và thiếu `wrapt` khi dùng `pip` tiêu chuẩn.
- **Cách tìm nguyên nhân và xử lý:** Chuyển sang sử dụng trình quản lý gói `uv` (`uv pip install -r requirements.txt`) giúp tự động giải quyết xung đột phụ thuộc và cài đặt thành công 39 gói thư viện.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics cho biết triệu chứng bất thường và khoảng thời gian xảy ra -> Logs lọc ra đúng `correlation_id` của request đại diện -> Traces đào sâu vào từng span để chỉ ra chính xác bước gây lỗi/chậm.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Giúp kiểm soát chi phí token, đảm bảo cam kết chất lượng SLO 99.5% và hỗ trợ Rollback prompt tức thì về v1 nếu v2 gặp sự cố regression.
- **Điều quan trọng nhất đã học:** Quy trình vận hành và quan sát ứng dụng AI (LLMOps) chuẩn mực dựa trên chuỗi bằng chứng thực tế.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Không có.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
