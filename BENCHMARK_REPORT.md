# Benchmark & Observability Report — Day 14

**Họ tên:** Tạ Công Bình (MSSV: 2A202603628)  
**Ngày chạy:** 2026-11-25  
**Môi trường:** Windows 11, Python 3.12.9, Docker Desktop (WSL2 backend)

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Với `MOCK_LATENCY_MS=120` + `MOCK_JITTER_MS=60` + retrieval ~20ms, 95% request bình thường hoàn thành ~150–250ms; ngưỡng 500ms giữ trải nghiệm chat phản hồi nhanh mà vẫn chịu được dao động tải vừa phải. |
| Error rate `/ask` | < 2.0 % | `MOCK_ERROR_RATE` mặc định là 1% (0.01); ngưỡng SLO 2% tạo khoảng an toàn nhỏ cho biến động ngẫu nhiên nhưng vẫn bắt được lỗi upstream tăng bất thường. |
| Accuracy trên golden dataset | ≥ 80 % (và 100 % sau khi fix KB, 0 regression) | Đảm bảo agent v2 (RAG) giữ chất lượng cao trên bộ 23 câu hỏi chuẩn trước khi cho phép deploy. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 3.46 | 204.3 | 1662.8 | 1729.7 | 1.00% | FAIL (p95 > 500ms) |
| 10 | 37.74 | 195.0 | 234.1 | 1696.8 | 1.33% | PASS |
| 50 | 99.39 | 210.0 | 415.0 | 1763.0 | 0.67% | PASS |
| 100 | 42.00 | 1177.6 | 5184.3 | 5380.8 | 2.00% | FAIL (p95 > 500ms) |

**Nhận xét:** RPS không tăng tuyến tính mãi theo concurrency. Từ `c=1` lên `c=10` và `c=50`, vì endpoint `async` dùng `await asyncio.sleep(...)` không block event loop nên RPS tăng mạnh (3.46 → 37.74 → 99.39 req/s). Tuy nhiên khi lên `c=100` trên 1 process uvicorn chạy local, event loop / socket contention và hàng đợi xử lý bị quá tải (saturation), khiến p50 nhảy từ 210.0 ms lên 1177.6 ms, p95 vọt lên 5184.3 ms và RPS tụt xuống 42.00 req/s. (Ở `c=1`, p95=1662.8 ms do mẫu tail latency ngẫu nhiên >5% trên 300 request rơi vào nhóm `MOCK_SLOW_RATE=0.03`).

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 195.8 | 233.3 | 243.8 |
| 0.03 (mặc định) | 244.7 | 236.1 | 1722.1 |
| 0.1 | 319.4 | 1687.6 | 1718.9 |

**Nhận xét:** `p95` và `p99` nhạy nhất với request chậm: khi tăng `MOCK_SLOW_RATE` từ `0` lên `0.03`, `avg` chỉ tăng nhẹ (195.8 → 244.7 ms, trông vẫn rất ổn) trong khi `p99` nhảy vọt từ 243.8 ms lên 1722.1 ms (gấp ~7 lần). Khi lên `0.1` (10% request chậm), cả `p95` cũng vọt từ 236.1 ms lên 1687.6 ms. Vì `average` bị làm mượt bởi 90–97% request nhanh nên che giấu trải nghiệm tệ của nhóm user bị chậm; SLO bắt buộc phải đặt theo `p95`/`p99`.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 57.8 | 108.7 | 114.0 |
| Tổng cost (USD) | $0.000414 | $0.000756 | $0.000794 |
| Cost / câu đúng (USD) | $0.00041415 | $0.00003599 | $0.00003452 |
| Quality gate | FAIL | PASS (`--min-accuracy 0.8`) | PASS (`--min-accuracy 1.0`, 0 regression) |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry `(["jwt", "xác thực"], ...)` vào `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` (và đồng bộ sang `production/utils/mock_llm.py`) giải thích JWT là token có chữ ký số và thời gian hết hạn, khác với API key tĩnh. |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry `(["hpa", "kubernetes"], ...)` vào `KNOWLEDGE_BASE` giải thích Kubernetes HPA tự động scale pod dựa trên CPU, memory hoặc custom metrics. |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**
- Agent v2 thêm bước retrieval làm prompt dài hơn (avg tokens tăng từ 57.8 lên 114.0, tổng chi phí tăng từ `$0.000414` lên `$0.000794`) và latency tăng thêm ~20ms cho bước tìm tài liệu (`p50` 170.1 ms → 204.0 ms).
- Đổi lại, accuracy tăng vọt từ `4.3%` lên `100.0%`, nên **`cost / câu đúng` giảm ~12 lần** (từ `$0.00041415` xuống `$0.00003452`). Đây là đánh đổi hoàn toàn xứng đáng.

**Quality gate trong CI/CD:**
- `eval/run_eval.py` trả exit code `1` nếu `accuracy < --min-accuracy` hoặc có bất kỳ câu nào bị regression so với `--baseline`. Trong GitHub Actions, đặt bước `python eval/run_eval.py --url ... --min-accuracy 1.0 --baseline reports/eval_v2.json` sau khi khởi động server test; nếu exit code khác `0`, CI job fail và chặn merge PR vào `main`.

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải (`screenshots/grafana-dashboard.png`)
- [x] Một trace chậm trong Jaeger (`screenshots/jaeger-slow-trace.png`)
- [x] Một trace lỗi trong Jaeger (`screenshots/jaeger-error-trace.png`)
- [x] Trang Alerts của Prometheus với ≥ 2 alert đang **firing** (`screenshots/prometheus-alert-firing.png`)

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` | 75s (`for: 1m`) | `Error rate`, `Requests/s theo status code`, `LLM calls theo kết quả` |
| B | `MOCK_SLOW_RATE=0.3` | `AgentHighLatencyP95` | 135s (`for: 2m`) | `Latency p95`, `Latency /ask — p50 / p95 / p99` |
| C | `LLM_MODEL=mock-llm-large` | `LLMCostBurnRateHigh` | Pending sau ~15s (đạt `$3.06–$4.17/giờ` > `$2/giờ`), firing sau `for: 2m` (đã xác nhận firing ở 15s khi tích lũy tải cao) | `Cost burn rate (USD/giờ)`, `LLM cost (1h)`, `LLM latency p95 theo model` |
| D | `docker pause` 1 instance | `AgentInstanceDown` | 45s (`for: 30s`) | `Instances up`, `Requests/s theo instance` |
| E | `AGENT_VERSION=v1` | Không có | N/A | `Retrieval hit ratio (agent v2)`, `Tokens/s`, `Phiên bản đang chạy` |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

Các metric hạ tầng (HTTP 2xx, latency, CPU/RAM) vẫn xanh vì agent v1 trả HTTP 200 nhanh và ít lỗi hơn. Để phát hiện suy giảm chất lượng trong production cần: (1) chạy automated eval trên golden dataset làm quality gate trong CI/CD trước khi deploy; (2) theo dõi các metric đặc thù của RAG/LLM như `Retrieval hit ratio (agent v2)` (sẽ tụt về 0 ở v1) hoặc số token trung bình; (3) thu thập feedback người dùng (thumbs up/down) và chạy LLM-as-a-judge sampling trên traffic thật.

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất tìm được trong Jaeger:

- `trace_id`: `ab945c7ca5740bae15cd5524ef96c427` (ngoài ra đã quan sát trace lỗi `2f0198574f029c2459341d7b4df01684` với `"event": "llm_error"`)
- Tổng thời gian: `1710.5 ms` (`POST /ask`, HTTP log ghi `1711.1 ms`; `agent.run`: `1707.93 ms`; `agent.retrieve_context`: `21.68 ms` / `1.3%`)
- Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (`1686.00 ms`, chiếm `98.6%` tổng thời gian request)
- Dòng log tương ứng (grep theo `trace_id`):

```json
{"ts": "2026-11-25T09:13:47.185417+00:00", "level": "INFO", "service": "observable-agent", "instance": "b05e7da048b9", "event": "llm_call", "trace_id": "ab945c7ca5740bae15cd5524ef96c427", "span_id": "c32dc5895886fb09", "request_id": "234065d39b3c", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 95, "output_tokens": 55, "cost_usd": 4.7e-05, "llm_ms": 1686.8}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

Số liệu từ `benchmark-results/scale_1_stats.csv` và `benchmark-results/scale_3_stats.csv` (dòng `Aggregated` — `POST /ask` + `GET /health`):

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 65.33 (`POST /ask`: 59.09) | 190 (`POST /ask`: 200) | 290 (`POST /ask`: 320) | 1700 | 73 / 7755 (0.94%) |
| 3 | 65.88 (`POST /ask`: 60.03) | 190 (`POST /ask`: 200) | 270 (`POST /ask`: 270) | 1700 | 60 / 7880 (0.76%) |

**Giải thích kết quả:** Scale từ 1 lên 3 instance chỉ giảm nhẹ `p95` của `POST /ask` từ `320 ms` xuống `270 ms` (`Aggregated`: `290 ms → 270 ms`), trong khi RPS và `p50`/`p99` gần như giữ nguyên (~65.3 → ~65.9 req/s, `p99` vẫn `1700 ms`). Nguyên nhân là nút thắt không nằm ở CPU của agent mà nằm ở: (1) thời gian chờ I/O bất đồng bộ mô phỏng gọi LLM (`MOCK_LATENCY_MS=120` + `MOCK_JITTER_MS=60` + `MOCK_SLOW_RATE=0.03` cộng thêm `1500 ms` tạo ra `p99=1700 ms`); và (2) kịch bản Locust giới hạn 100 user có `wait_time` giữa các request nên 1 async worker FastAPI đã đủ phục vụ ~65 req/s mà chưa bão hoà CPU. Muốn giảm `p95`/`p99` cần tối ưu tầng LLM (model nhanh hơn, caching, streaming) chứ không chỉ thêm replica.

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. Latency trung bình (avg) che giấu tail latency; phải dùng `p95`/`p99` làm SLO và chạy load test ở nhiều mức concurrency để thấy điểm bão hoà (saturation).
  2. Eval trên golden dataset kết hợp với chỉ số `cost / câu đúng` và baseline regression check giúp đánh giá đúng giá trị của RAG (v2) thay vì chỉ nhìn tổng chi phí.
  3. 3 trụ cột Metrics – Traces – Logs liên kết qua `trace_id` giúp đi từ triệu chứng tổng quan (Grafana/Alert) xuống đúng span gây chậm (Jaeger) và chi tiết lỗi (JSON log).
- Nếu đưa agent này lên production thật, cần thêm metric / alert:
  - Metric & alert cho retrieval hit ratio (phát hiện lỗi RAG như thí nghiệm E), tỷ lệ HTTP 429/402 từ rate limit & cost guard, và Redis connection/latency metrics.
