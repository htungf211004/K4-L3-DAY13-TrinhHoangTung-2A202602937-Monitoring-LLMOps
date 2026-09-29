# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: HighLatencyP95
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-llmops-l3a)
- SLI/SLO liên quan: `fast_successful_requests` (SLI: `latency_ms <= 3000ms`, mục tiêu 99.5%)
- Điều kiện và thời gian duy trì: `latency_p95 > 3000ms` duy trì liên tục trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng bị phản hồi chậm, trải nghiệm giao tiếp gián đoạn, nguy cơ client timeout
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel Latency trên dashboard để xác nhận khoảng thời gian bắt đầu tăng độ trễ và tỷ lệ request bị chậm.
  2. Lọc log trong `data/logs.jsonl` tìm request có `latency_ms > 3000`, lấy `correlation_id`.
  3. Mở Langfuse tìm trace có `correlation_id` tương ứng, soi waterfall xem span nào bị chậm (retrieval chậm do RAG hay generation chậm do LLM).
- Mitigation tạm thời:
  - Nếu span `retrieval` bị chậm (ví dụ do vector DB overload): kích hoạt fallback cache hoặc tăng timeout tạm thời, kiểm tra incident `rag_slow`.
  - Nếu LLM generation chậm: kiểm tra số lượng token đầu ra hoặc chuyển hướng sang model fallback / giảm max_tokens.
- Owner: oncall-engineer

## Alert 2

- Tên: HighErrorRate
- Severity: critical
- Duration: 3m
- Kênh thông báo: Slack (#alerts-llmops-l3a)
- SLI/SLO liên quan: `fast_successful_requests` (tỷ lệ request lỗi tối đa 2%)
- Điều kiện và thời gian duy trì: `error_rate_pct > 2.0%` duy trì liên tục trong 3 phút
- Ảnh hưởng tới người dùng: Người dùng nhận mã lỗi 500 (Internal Server Error), không nhận được câu trả lời từ chatbot
- Ba bước kiểm tra đầu tiên:
  1. Mở panel Errors trên dashboard để xác định `error_type` (ví dụ `RuntimeError`, `HTTPException`, etc.).
  2. Tra cứu log `event == "request_failed"` trong `data/logs.jsonl`, trích xuất `correlation_id` và `payload.detail`.
  3. Mở Langfuse trace để xác định bước gây exception (thường là retrieval lỗi `tool_fail` hoặc model generation lỗi).
- Mitigation tạm thời:
  - Nếu retrieval lỗi (`Vector store timeout`): vô hiệu hóa incident `tool_fail` qua `/incidents/tool_fail/disable` hoặc bật cơ chế fallback corpus cục bộ.
  - Nếu API service crash: restart lại container/process uvicorn.
- Owner: oncall-engineer

## Alert 3

- Tên: LowQualityOrRetrievalDegraded
- Severity: warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-llmops-l3a)
- SLI/SLO liên quan: Quality Guardrail (`quality_score_avg >= 0.75`), Retrieval Guardrail (`retrieval_success_rate >= 90%`)
- Điều kiện và thời gian duy trì: `quality_score_avg < 0.75` hoặc `retrieval_success_rate < 90%` trong 5 phút
- Ảnh hưởng tới người dùng: Câu trả lời chatbot chung chung, sai lệch hoặc không dựa trên tài liệu doanh nghiệp
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel Quality proxy và panel Errors (retrieval success rate) trên dashboard.
  2. Kiểm tra log `response_sent` có `tool_success == false` hoặc `quality_score < 0.75`.
  3. Mở Langfuse đối chiếu prompt version đang chạy (`prompt_version`, `prompt_label`) xem có phải phiên bản prompt mới (candidate) làm giảm chất lượng không.
- Mitigation tạm thời:
  - Nếu do prompt mới: thực hiện rollback `LANGFUSE_PROMPT_LABEL=production` về version ổn định trước đó (v1).
  - Nếu do retrieval không tìm thấy tài liệu: kiểm tra bộ index embedding và cập nhật tài liệu domain.
- Owner: oncall-engineer

