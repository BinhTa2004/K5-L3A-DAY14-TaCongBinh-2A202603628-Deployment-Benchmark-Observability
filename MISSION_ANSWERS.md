# Day 14 Lab — Mission Answers

> **Họ tên:** Tạ Công Bình  
> **MSSV:** 2A202603628  
> **Lớp:** K5-L3A

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns trong `01-localhost-vs-production/develop/app.py`

1. **Hardcoded secrets (dòng 17–18):** Khai báo trực tiếp `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"` trong source code — lộ ngay khi push lên Git.
2. **Hardcoded config (dòng 21–22):** `DEBUG = True` và `MAX_TOKENS = 500` ghi cứng trong code, không đọc từ biến môi trường nên không tách biệt được dev/staging/production.
3. **Dùng `print()` và lộ secret vào log (dòng 33–38):** Không có structured JSON logging; đặc biệt dòng 34 in thẳng `OPENAI_API_KEY` ra stdout (`print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`).
4. **Thiếu health check / readiness endpoint (dòng 42–43):** Không có `/health` hay `/ready` để platform/load balancer kiểm tra trạng thái process và tự động restart hoặc ngắt route.
5. **Bind `localhost`, cứng port và bật `reload=True` (dòng 49–53):** `host="localhost"` khiến container không nhận được traffic từ bên ngoài; `port=8000` bỏ qua biến `PORT` do cloud platform cấp; `reload=True` gây tốn tài nguyên và mất ổn định trong production.
6. **Không có graceful shutdown & CORS:** Không dùng `lifespan` hay xử lý `SIGTERM` để hoàn tất request đang chạy trước khi tắt.

### Exercise 1.2: Kết quả chạy bản `develop`

Lệnh chạy:

```bash
cd 01-localhost-vs-production/develop
python app.py
curl -i -X POST "http://127.0.0.1:8000/ask?question=Hello"
```

Output thực tế:

```http
HTTP/1.1 200 OK
date: Wed, 25 Nov 2026 08:36:06 GMT
server: uvicorn
content-length: 103
content-type: application/json

{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 1.3: So sánh `develop/app.py` và `production/app.py`

| Feature | Develop (`develop/app.py`) | Production (`production/app.py`) | Tại sao quan trọng? |
| --- | --- | --- | --- |
| Config & Secrets | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG`, `port=8000` (dòng 17–22, 52) | Đọc từ biến môi trường qua `config.settings` (`PORT`, `ENVIRONMENT`, `DEBUG`...) | Tuân thủ 12-Factor App; không lộ secret lên Git và đổi cấu hình giữa dev/staging/prod không cần sửa code. |
| Health check | Không có (dòng 42–43) | Có `GET /health` (liveness, dòng 135–149) và `GET /ready` (readiness, trả 503 khi chưa sẵn sàng, dòng 152–161) | Orchestrator/cloud biết khi nào restart container bị treo và khi nào instance sẵn sàng nhận traffic. |
| Logging | `print(...)` text thuần, log cả `OPENAI_API_KEY` (dòng 33–38) | `logging` định dạng JSON có timestamp, level, `event`, `question_length` (dòng 28–32, 111–122), không log secret | Log aggregators (Loki, Datadog) dễ parse, lọc, và đảm bảo an toàn thông tin. |
| Network binding & Reload | `host="localhost"`, `port=8000`, `reload=True` (dòng 51–53) | `host=settings.host` (`0.0.0.0`), `port=settings.port` (từ `PORT`), `reload=settings.debug` (dòng 194–199) | Cho phép nhận kết nối từ bên ngoài container và tương thích port động trên Railway/Render. |
| Shutdown | Tắt đột ngột, không dọn dẹp | `lifespan` + `signal.signal(signal.SIGTERM, handle_sigterm)` (dòng 39–67, 180–189) | Cho phép hoàn tất request đang xử lý (in-flight) và đóng connection an toàn khi deploy/scale down. |

---

## Part 2: Docker

### Exercise 2.1: Phân tích `02-docker/develop/Dockerfile`

1. **Base image:** `python:3.11` (full Python distribution, ~1 GB).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao `COPY 02-docker/develop/requirements.txt .` trước `COPY 02-docker/develop/app.py .`?** Để tận dụng Docker layer cache. Khi chỉ thay đổi code ứng dụng (`app.py`), Docker tái sử dụng lại layer `pip install` đã cache mà không phải tải và cài lại toàn bộ thư viện.
4. **`CMD` vs `ENTRYPOINT`:** `ENTRYPOINT` định nghĩa tiến trình chính cố định của container (khó bị ghi đè khi truyền thêm argument ở `docker run`), còn `CMD` cung cấp lệnh hoặc tham số mặc định cho `ENTRYPOINT` (dễ bị ghi đè bởi arguments truyền lúc `docker run`).

### Exercise 2.2 – 2.4: Build image, đo dung lượng và chạy Compose stack

Các lệnh đã chạy từ gốc repo:

```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .
docker images my-agent --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

- **Image size comparison (đo thực tế):**
  - `my-agent:develop`: **1.67 GB** (`~1670 MB`)
  - `my-agent:advanced`: **240 MB**
  - Chênh lệch: **giảm 85.6%** (`1.67 GB → 240 MB`), nhờ multi-stage build tách stage `builder` (chứa `gcc`, `libpq-dev`, pip cache) khỏi stage `runtime` (`python:3.11-slim` chỉ copy `/root/.local`, code và chạy dưới user non-root `appuser`).

- **Kết quả gọi container `my-agent:develop` (`POST http://127.0.0.1:8000/ask?question=What%20is%20Docker%3F`):**

```http
HTTP/1.1 200 OK
date: Wed, 25 Nov 2026 08:50:32 GMT
server: uvicorn
content-length: 103
content-type: application/json

{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

- **Kiến trúc Docker Compose (`02-docker/production/docker-compose.yml`):**
  - `nginx` (`:80`, `:443`) làm reverse proxy/load balancer từ bên ngoài → định tuyến qua mạng nội bộ `internal` tới `agent:8000` (áp dụng rate limit `10r/s` và security headers).
  - `agent` kết nối tới `redis:6379/0` (session/rate limit) và `qdrant:6333` (vector DB), dùng `depends_on` với `condition: service_healthy`.
- **Kết quả test qua Nginx (`docker compose -p k5-day14-docker`):**
  - `GET http://127.0.0.1/health` → `200 OK`: `{"status":"ok","uptime_seconds":6.7,"version":"2.0.0","timestamp":"2026-11-25T08:51:14.619743"}`
  - `POST http://127.0.0.1/ask` (`{"question":"Explain microservices"}`) → `200 OK`: `{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}`

---

## Part 3: Cloud Deployment

### Exercise 3.1 / 3.2: Deploy agent Part 6 (`06-lab-complete/`)

- **Platform chính:** Render (Web Service Docker + Render Key Value / Redis, region Singapore, Root Directory `06-lab-complete`)
- **Public URL:** https://day14-ai-agent-mqab.onrender.com
- **Kiểm tra 4 lệnh CP6:** `/health` → `200`, `/ready` → `200`, `/ask` (không có `X-API-Key`) → `401`, `/ask` (có `X-API-Key`) → `200` (xem ảnh `screenshots/test.png`)
- **Screenshots:** [dashboard.png](screenshots/dashboard.png), [running.png](screenshots/running.png), [test.png](screenshots/test.png)

### So sánh `railway.toml` và `render.yaml`

1. **Phạm vi cấu hình:** `railway.toml` dùng định dạng TOML để cấu hình build và deploy cho **1 service** (`[build]`, `[deploy]`), còn `render.yaml` là **Blueprint (Infrastructure as Code)** cho phép khai báo **nhiều service** cùng lúc (cả Web Service lẫn Redis add-on) trong một file YAML.
2. **Cách khai báo build/runtime:** `railway.toml` chỉ định `builder = "DOCKERFILE"` (hoặc `NIXPACKS`), còn `render.yaml` khai báo `runtime: docker` (hoặc `python` kèm `buildCommand`, `startCommand`), `region`, `plan`, `autoDeploy`.
3. **Quản lý biến môi trường & secret:** `railway.toml` không chứa danh sách env var/secret (đặt qua `railway variables set` hoặc Dashboard), trong khi `render.yaml` khai báo trực tiếp trong mục `envVars` với `generateValue: true` (tự sinh secret ngẫu nhiên) hoặc `sync: false` (nhập tay trên Dashboard để không lộ secret vào Git).
4. **Health check:** Cả hai đều cấu hình đường dẫn health check `/health` nhưng tên key khác nhau (`healthcheckPath` trong `railway.toml` vs `healthCheckPath` trong `render.yaml`).

### Exercise 3.3: Mô tả CI/CD pipeline Cloud Run (`03-cloud-deployment/production-cloud-run/`)

Pipeline trong `cloudbuild.yaml` chạy tự động qua 4 bước tuần tự (`waitFor`):
1. **`test`:** Dùng container `python:3.11-slim` cài `requirements.txt` + `pytest` và chạy `pytest tests/ -v`.
2. **`build`:** Build Docker image gắn tag `$COMMIT_SHA` và `latest` (tận dụng `--cache-from` từ `latest`).
3. **`push`:** Push các image tags lên Google Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
4. **`deploy`:** Chạy `gcloud run deploy ai-agent` triển khai image vừa build lên Cloud Run (`asia-southeast1`, `min-instances=1`, `max-instances=10`, `512Mi`), đồng thời gắn secret từ Secret Manager. File `service.yaml` bổ sung đặc tả Knative Service gồm autoscaling (`minScale: 1`, `maxScale: 10`, `containerConcurrency: 80`), `livenessProbe` (`/health`), và `startupProbe` (`/ready`).

---

## Part 4: API Security

### Exercise 4.1: API Key Authentication (`04-api-gateway/develop/`)

Chạy với `AGENT_API_KEY=secret-key-123`:

- **Không có key → `401 Unauthorized`:**

```http
HTTP/1.1 401 Unauthorized
date: Wed, 25 Nov 2026 08:36:11 GMT
server: uvicorn
content-length: 67
content-type: application/json

{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
```

- **Có key hợp lệ → `200 OK`:**

```http
HTTP/1.1 200 OK
date: Wed, 25 Nov 2026 08:36:11 GMT
server: uvicorn
content-length: 114
content-type: application/json

{"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

### Exercise 4.2 & 4.3: JWT Auth + Rate Limiting (`04-api-gateway/production/`)

Lấy token từ `POST /auth/token` (`student` / `demo123`, giới hạn 10 req/phút, sliding window dùng `deque` trong `rate_limiter.py`):

```text
Token response (masked): {"access_token":"<TOKEN>","token_type":"bearer","expires_in_minutes":60,"hint":"Include in header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."}
```

Gọi vòng lặp 12 request liên tiếp bằng `curl -i` với `Authorization: Bearer <TOKEN>`:

```text
[1] HTTP/1.1 200 OK | {"question":"Test 1","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":9,"budget_remaining_usd":2.1e-05}}
[2] HTTP/1.1 200 OK | {"question":"Test 2","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":8,"budget_remaining_usd":4e-05}}
...
[10] HTTP/1.1 200 OK | {"question":"Test 10","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":0,"budget_remaining_usd":0.000191}}
[11] HTTP/1.1 429 Too Many Requests | {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
[12] HTTP/1.1 429 Too Many Requests | {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

- **Thuật toán rate limit (`rate_limiter.py`):** Sliding window counter (loại bỏ timestamp ngoài cửa sổ 60s; `user`: 10 req/phút, `admin`: 100 req/phút; chọn limiter theo `role` trong JWT payload nên `admin` có ngưỡng cao hơn).

### Exercise 4.4: Cost Guard (`cost_guard.py`)

- **Cơ chế hoạt động:** Tính chi phí dựa trên số `input_tokens` (`$0.00015 / 1K`) và `output_tokens` (`$0.0006 / 1K`). Trước mỗi lần gọi LLM, `check_budget()` kiểm tra tổng chi phí trong ngày: nếu vượt ngưỡng user (`$1.0`/ngày) trả HTTP `402 Payment Required`, nếu vượt ngưỡng toàn hệ thống (`$10.0`/ngày) trả `503`, đạt 80% thì ghi log cảnh báo. Sau khi LLM trả lời, `record_usage()` cộng dồn token và chi phí.
- **Lý do cần cả Rate Limit và Cost Guard:** Rate limit kiểm soát tốc độ request (chống spam/DDoS), nhưng một ít request có prompt/output rất dài vẫn có thể đốt cạn tiền API. Cost guard kiểm soát trực tiếp chi phí tính theo token. Trong Part 6 (`06-lab-complete/app/cost_guard.py`), chi phí được lưu bằng `INCRBYFLOAT` trên Redis theo ngày để dùng chung giữa nhiều worker/instance.

---

## Part 5: Scaling & Reliability

### Exercise 5.1 – 5.3: Health checks, Graceful shutdown & Stateless design

- **Liveness (`/health`) vs Readiness (`/ready`):**
  - `/health` kiểm tra process còn sống hay không (luôn trả `200` kể cả khi Redis tạm mất kết nối) — nếu fail thì platform restart container.
  - `/ready` kiểm tra khả năng phục vụ traffic thật (ping Redis; trả `503` khi mất kết nối hoặc đang tắt) — nếu fail thì load balancer ngừng đưa request mới tới instance đó mà không restart oan.
- **Tại sao state trong RAM hỏng khi scale nhiều instance:** Mỗi instance có bộ nhớ RAM riêng. Khi Load Balancer chia request theo round-robin, request 1 vào `agent-1` lưu session vào RAM của `agent-1`, request 2 vào `agent-2` sẽ không thấy lịch sử đó. Lưu session/rate-limit/cost vào Redis dùng chung giúp mọi instance đều stateless.

### Exercise 5.4 & 5.5: Load balancing & `test_stateless.py` (`05-scaling-reliability/production/`)

Chạy `docker compose -p k5-day14-scaling up -d --build --scale agent=3` rồi chạy `python test_stateless.py`:

```text
============================================================
Stateless Scaling Demo
============================================================

Session ID: 99caf199-c225-41e1-9a16-ba121cb3405c

Request 1: [instance-9e7116]
  Q: What is Docker?
  A: Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!...

Request 2: [instance-8a80e3]
  Q: Why do we need containers?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....

Request 3: [instance-1e0b0f]
  Q: What is Kubernetes?
  A: Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ O...

Request 4: [instance-9e7116]
  Q: How does load balancing work?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....

Request 5: [instance-8a80e3]
  Q: What is Redis used for?
  A: Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ O...

------------------------------------------------------------
Total requests: 5
Instances used: {'instance-9e7116', 'instance-1e0b0f', 'instance-8a80e3'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
  [user]: What is Docker?...
  [assistant]: Container là cách đóng gói app để chạy ở mọi nơi. Build once...
  [user]: Why do we need containers?...
  [assistant]: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đ...
  [user]: What is Kubernetes?...
  [assistant]: Đây là câu trả lời từ AI agent (mock). Trong production, đây...
  [user]: How does load balancing work?...
  [assistant]: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đ...
  [user]: What is Redis used for?...
  [assistant]: Đây là câu trả lời từ AI agent (mock). Trong production, đây...

✅ Session history preserved across all instances via Redis!
```

**Giải thích:** 5 request trong cùng `session_id` được Nginx chia đều sang cả 3 instance khác nhau (`instance-9e7116`, `instance-8a80e3`, `instance-1e0b0f`), nhưng nhờ toàn bộ lịch sử chat được lưu trong Redis (`session:<id>`) nên truy vấn `/chat/{session_id}/history` vẫn còn đầy đủ cả 10 messages.

---

## Part 6: Final Project (`06-lab-complete/`) — Kết quả kiểm tra

Chạy đầy đủ `python check_production_ready.py` trên Linux (build image **56 MB**, `docker stop` thoát êm **exit code 0 trong 1.2s**, `SIGTERM` lúc đang có request trả **200** và tự thoát **exit code 0**):

```text
================================================================
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 43/43 checks passed
  PASS 12/12 — PRODUCTION READY!
================================================================
```

---

## AI log

1. **Hỏi/nhờ AI:** Cách triển khai các `TODO` trong khung `06-lab-complete/` theo đúng hợp đồng 12 tiêu chí của `check_production_ready.py`, và phân tích và chạy benchmark local Part 7.
2. **Nhận được:** Code hoàn chỉnh cho `config.py`, `redis_client.py`, `auth.py`, `rate_limiter.py`, `cost_guard.py`, `main.py`, `Dockerfile`, `docker-compose.yml`, `.dockerignore`, cập nhật `KNOWLEDGE_BASE` trong `mock_llm.py` cho 2 câu `security-02` và `scaling-03`.
3. **Tự kiểm tra:** Đã đọc kỹ từng file sửa, chạy `python check_production_ready.py` đầy đủ trên Linux đạt **`PASS 12/12` (43/43 checks passed, Docker image 56 MB, `docker stop` exit 0 trong 1.2s, `SIGTERM` exit 0)**; chạy thực tế Part 1, Part 2 (1.67 GB → 240 MB, giảm 85.6%), Part 4, Part 5 (`test_stateless.py` qua 3 instances) và toàn bộ `load_test.py` + `eval/run_eval.py` (`23/23` — `100.0%`, exit code 0) và đối chiếu từng con số với file JSON trong `reports/`.
