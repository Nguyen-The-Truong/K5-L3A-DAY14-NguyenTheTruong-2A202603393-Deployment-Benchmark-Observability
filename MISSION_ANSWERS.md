# Day 14 Lab — Mission Answers

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns trong `01-localhost-vs-production/develop/app.py`
1. **Hardcode secrets trong source code** (dòng 17–18): `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"` — đẩy lên GitHub là lộ ngay thông tin nhạy cảm.
2. **Không quản lý cấu hình qua environment variables** (dòng 21–22): `DEBUG = True`, `MAX_TOKENS = 500` được gán cứng trong code, không đổi được theo môi trường dev/staging/production.
3. **Dùng `print()` và log ra secret** (dòng 33–34): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` vừa không có cấu trúc JSON để hệ thống log aggregator parse, vừa làm lộ API key ra stdout/log.
4. **Không có health check endpoint (`/health`, `/ready`)** (dòng 42–43): platform/load balancer không kiểm tra được app còn sống hay đã sẵn sàng nhận traffic.
5. **Cấu hình server gán cứng kiểu localhost + debug reload** (dòng 49–53): `host="localhost"` (không nhận được traffic từ ngoài container), `port=8000` (không đọc `PORT` do cloud platform cấp), `reload=True` (bật auto-reload trong production), và không xử lý graceful shutdown (`SIGTERM`/lifespan).

### Exercise 1.2: Kết quả chạy bản `develop`
```text
POST http://127.0.0.1:8000/ask?question=Hello
HTTP 200 {"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

### Exercise 1.3: Bảng so sánh `develop/app.py` vs `production/app.py`
| Feature | Basic (`develop`) | Advanced (`production`) | Tại sao quan trọng? |
|---|---|---|---|
| **Config** | Hardcode `OPENAI_API_KEY`, `PORT=8000`, `DEBUG=True` trong `app.py` | Tách ra `config.py` đọc từ `os.getenv(...)` và `validate()` fail-fast | Đáp ứng 12-Factor: cùng 1 artifact chạy được ở dev/staging/prod, không lộ secret trên Git |
| **Health check** | Không có endpoint `/health` hay `/ready` | Có `GET /health` (liveness) và `GET /ready` (readiness, trả 503 khi chưa sẵn sàng) | Platform biết khi nào cần restart container và khi nào được route traffic vào |
| **Logging** | Dùng `print("[DEBUG] ...")`, còn in cả `OPENAI_API_KEY` | Dùng `logging` định dạng JSON có timestamp, level, `event`, không log secret | Dễ tìm kiếm, lọc và thu thập tự động trên Loki/Datadog/Cloud Logging |
| **Shutdown** | Thoát đột ngột | Có `lifespan` chuyển `is_ready = False` khi shutdown và đăng ký `signal.SIGTERM` để uvicorn hoàn thành in-flight requests | Tránh cắt ngang request người dùng khi deploy/scale-in |
| **Network binding & reload** | `host="localhost"`, `port=8000`, `reload=True` | `host=settings.host` (`0.0.0.0`), `port=settings.port` (từ `PORT`), `reload=settings.debug` | Container bắt buộc listen `0.0.0.0` và nhận `PORT` động từ Railway/Render |

---

## Part 2: Docker

### Exercise 2.1: Phân tích `02-docker/develop/Dockerfile`
1. **Base image:** `python:3.11` (full Python image, dung lượng lớn ~1 GB vì chứa nhiều build tools và package hệ thống).
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Tại sao `COPY requirements.txt` đứng trước `COPY . .` (hay `COPY app.py`):** Để tận dụng **Docker layer cache**. Khi chỉ sửa code ứng dụng mà không đổi `requirements.txt`, Docker dùng lại cache của bước `RUN pip install`, tránh tải và cài lại toàn bộ thư viện mỗi lần build.
4. **Khác nhau giữa `CMD` và `ENTRYPOINT`:**
   - `CMD` định nghĩa lệnh/tham số mặc định khi chạy container và có thể bị ghi đè dễ dàng khi truyền command sau `docker run <image> <cmd>`.
   - `ENTRYPOINT` cố định executable chính của container; các tham số truyền thêm ở `docker run` (hoặc từ `CMD`) sẽ được nối vào làm đối số cho `ENTRYPOINT`.

### Exercise 2.2 & 2.3: Build, test container & so sánh dung lượng image
- **Kết quả gọi container `develop` (`docker run -p 8000:8000 my-agent:develop`):**
```text
POST http://127.0.0.1:8000/ask?question=What+is+Docker%3F
HTTP 200 {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```
- **So sánh dung lượng (`docker images`):**
  - `my-agent:develop`: **1.67 GB** (~1670 MB)
  - `my-agent:advanced` (multi-stage): **240 MB**
  - Chênh lệch: **giảm 85.6%** (nhỏ hơn ~1.43 GB, ~7× lần)
- **Multi-stage build giảm dung lượng ở đâu:** Stage 1 (`builder`) cài `gcc`, `libpq-dev` và compile/install dependencies vào `/root/.local`. Stage 2 (`runtime`) tạo image mới từ `python:3.11-slim`, chỉ copy `/root/.local` và source code sang (bỏ toàn bộ `gcc`, apt cache, build tools), đồng thời chạy dưới user non-root `appuser`.

### Exercise 2.4: Docker Compose stack (`02-docker/production/docker-compose.yml`)
```text
Client ──▶ nginx (:80/:443) ──▶ agent (:8000, network internal)
                                  ├──▶ redis (:6379, session/rate limit)
                                  └──▶ qdrant (:6333, vector DB)
```
- **Services được start:** `agent`, `redis`, `qdrant`, `nginx` cùng chung bridge network `internal` (dữ liệu `redis_data`, `qdrant_data` lưu qua named volumes). Chỉ `nginx` expose port `80`/`443` ra ngoài.
- **Kết quả test qua Nginx:**
```text
GET http://127.0.0.1/health -> 200 {"status":"ok","uptime_seconds":0.4,"version":"2.0.0","timestamp":"2026-11-25T08:54:14.572378"}
POST http://127.0.0.1/ask   -> 200 {"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

---

## Part 3: Cloud Deployment

### Exercise 3.1 / 3.2: Deploy agent Part 6
- **Nền tảng chính:** `[Chờ deploy Railway hoặc Render sau khi PASS 12/12]`
- **Public URL:** `[Chờ URL public]`
- **Screenshots:** xem `DEPLOYMENT.md` và thư mục `screenshots/` (`dashboard.png`, `running.png`, `test.png`).

### So sánh `railway.toml` vs `render.yaml`
- **Cấu trúc & phạm vi:** `railway.toml` (TOML) gọn nhẹ, chủ yếu khai báo cấu hình build/deploy cho 1 service (`[build] builder`, `[deploy] startCommand`, `healthcheckPath`, `restartPolicyType`). `render.yaml` (YAML Blueprint) là Infrastructure-as-Code cho phép khai báo nhiều resource trong cùng file (cả `type: web` lẫn `type: redis`, region, plan free/starter, `autoDeploy`, và danh sách `envVars`).
- **Quản lý secret & biến môi trường:** Trên Railway, biến môi trường/secret đặt qua Dashboard hoặc `railway variables set`, không khai báo trực tiếp trong `railway.toml`. Trên Render, `render.yaml` hỗ trợ khai báo trước tên biến trong `envVars` với `generateValue: true` (nền tảng tự sinh secret ngẫu nhiên) hoặc `sync: false` (nhập tay trên dashboard để không commit secret vào Git).

### Exercise 3.3: Phân tích CI/CD pipeline Cloud Run (`03-cloud-deployment/production-cloud-run/`)
Pipeline trong `cloudbuild.yaml` chạy tự động gồm 4 bước tuần tự (`waitFor`):
1. **`test`:** Dùng `python:3.11-slim` cài `requirements.txt` + `pytest` và chạy `pytest tests/ -v`.
2. **`build`:** Build Docker image gắn tag `$COMMIT_SHA` và `latest` (tận dụng `--cache-from` từ image `latest`).
3. **`push`:** Đẩy image với các tag lên Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
4. **`deploy`:** Gọi `gcloud run deploy ai-agent` lên region `asia-southeast1` với `--min-instances=1` (tránh cold start), `--max-instances=10`, giới hạn CPU/RAM (`1` CPU / `512Mi`), inject env `ENVIRONMENT=production` và secret từ Google Secret Manager (`--set-secrets`). File `service.yaml` mô tả cấu hình Knative Service tương ứng (concurrency 80, livenessProbe `/health`, startupProbe `/ready`).

---

## Part 4: API Security

### Exercise 4.1: API Key Authentication (`04-api-gateway/develop/app.py`)
- API key được kiểm tra trong dependency `verify_api_key` (đọc header `X-API-Key` bằng `APIKeyHeader(name="X-API-Key", auto_error=False)`) và gắn vào `/ask` qua `Depends(verify_api_key)`.
- Nếu thiếu key → trả **401**; nếu sai key → trả **403** (hoặc **401** như yêu cầu Part 6).
- Để rotate key: đọc `AGENT_API_KEY` từ environment variable; khi xoay key có thể hỗ trợ tạm thời danh sách key cũ/mới (comma-separated) qua env trong giai đoạn chuyển tiếp rồi bỏ key cũ.
- **Kết quả test bản `develop`:**
```text
POST /ask (không có key) -> HTTP 401
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

POST /ask (X-API-Key: secret-key-123) -> HTTP 200
{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 4.2 & 4.3: JWT Authentication & Rate Limiting (`04-api-gateway/production/`)
- **Rate limiter (`rate_limiter.py`):** Dùng thuật toán **Sliding Window Counter** (lưu timestamp các request trong `deque` và loại bỏ timestamp cũ hơn `now - 60s`). Giới hạn mặc định: `student` (role `user`) = **10 req/phút**, `teacher` (role `admin`) = **100 req/phút** (`app.py` chọn `rate_limiter_admin` khi `role == "admin"`).
- **Output test thực tế (đã che token):**
```text
POST /ask (không token) -> HTTP 401
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

POST /auth/token (student / demo123) -> HTTP 200
{"access_token": "<REDACTED_JWT>", "token_type": "bearer", "expires_in_minutes": 60, "hint": "<REDACTED>"}

Loop 12 requests với Bearer token:
Req #1:  HTTP 200 | {"question":"Test 1",...,"usage":{"requests_remaining":9,"budget_remaining_usd":1.9e-05}}
...
Req #10: HTTP 200 | {"question":"Test 10",...,"usage":{"requests_remaining":0,"budget_remaining_usd":0.000181}}
Req #11: HTTP 429 | {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
Req #12: HTTP 429 | {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

### Exercise 4.4: Cost Guard (`cost_guard.py`)
- **Tại sao cần cả Rate Limit và Cost Guard:** Rate limit giới hạn **tốc độ** (số request/phút để chống spam/DDoS), nhưng một user gọi chậm rãi với prompt/output rất dài vẫn có thể đốt hết tiền API. Cost guard bảo vệ **ngân sách** bằng cách tính chi phí theo `input_tokens` và `output_tokens` rồi cộng dồn vào Redis theo key ngày/tháng (ví dụ `cost:daily:YYYY-MM-DD`); trước mỗi lần gọi LLM đều kiểm tra tổng chi tiêu hiện tại, nếu `>= budget` thì chặn bằng **HTTP 402 Payment Required**.

---

## Part 5: Scaling & Reliability

### Exercise 5.1 & 5.2: Health checks & Graceful shutdown (`05-scaling-reliability/develop/app.py`)
- **Phân biệt `/health` (liveness) vs `/ready` (readiness):**
  - `/health`: Kiểm tra process còn sống không. Nếu fail liên tục, platform sẽ **restart container**. Không nên phụ thuộc vào Redis/DB ngoài để tránh restart dây chuyền khi Redis tạm chập chờn.
  - `/ready`: Kiểm tra instance đã sẵn sàng nhận traffic chưa (đã khởi động xong và kết nối được Redis/DB). Nếu trả **503**, load balancer ngừng route traffic tới instance đó nhưng **không** restart container.
- **Graceful shutdown:** Khi nhận `SIGTERM`, app chuyển cờ `_is_ready = False` để load balancer ngừng gửi request mới, đợi các request đang xử lý (`_in_flight_requests`) hoàn thành trước khi đóng connections và thoát với exit code `0`.

### Exercise 5.3, 5.4 & 5.5: Stateless + Nginx Load Balancing (`05-scaling-reliability/production/`)
- **Tại sao state trong RAM hỏng khi scale:** Khi chạy 3 instance (`--scale agent=3`) sau Nginx, mỗi instance có bộ nhớ RAM riêng. Request 1 vào `agent-1` tạo session trong RAM của `agent-1`, nhưng request 2 được Nginx round-robin sang `agent-2` sẽ không tìm thấy session.
- **Giải pháp stateless:** Lưu toàn bộ `session:<session_id>` vào **Redis** (`save_session` dùng `setex`, `load_session` dùng `get`). Bất kỳ instance nào xử lý request cũng đọc/ghi chung một nguồn state trên Redis.
- **Kết quả chạy `docker compose -p k5-day14-scaling up -d --build --scale agent=3` và `python test_stateless.py`:**
```text
Session ID: fab397ce-bd89-49af-9faf-910a6337168a

Request 1: [instance-a000a4] -> Q: What is Docker?
Request 2: [instance-7ef157] -> Q: Why do we need containers?
Request 3: [instance-5f6ca2] -> Q: What is Kubernetes?
Request 4: [instance-a000a4] -> Q: How does load balancing work?
Request 5: [instance-7ef157] -> Q: What is Redis used for?

Total requests: 5
Instances used: {'instance-a000a4', 'instance-7ef157', 'instance-5f6ca2'}
✅ All requests served despite different instances!
Total messages: 10
✅ Session history preserved across all instances via Redis!
```
  Giải thích: Nginx round-robin phân tán 5 request qua cả 3 container (`instance-a000a4`, `instance-7ef157`, `instance-5f6ca2`), nhưng toàn bộ 10 message (5 turn hỏi–đáp) của session vẫn giữ nguyên vẹn vì state được lưu tập trung trong Redis.

---

## Part 6: Checklist Deliverable (`check_production_ready.py`)
Kết quả chạy `python check_production_ready.py` trong `06-lab-complete/` trên Windows + Docker 27.2.0 (image build: **69 MB**, `docker stop`: **exit code 0, 0.9s**; mục `SIGTERM` trực tiếp trên tiến trình uvicorn cần chạy trên WSL2/Linux để không bị `⚠️`):
```text
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
     ⚠️  Graceful shutdown   (Docker `docker stop` exit code 0, 0.9s ✅; Windows Python không gửi được SIGTERM)
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 11/12 tiêu chí đạt · 42/42 checks passed
```

---

## AI Log
- Sử dụng AI assistant để rà soát yêu cầu lab, kiểm tra chạy thử các ví dụ Part 1 & Part 4, hỗ trợ triển khai các file khung TODO trong `06-lab-complete/` và thiết lập benchmark/eval ở Part 7. Mọi kết quả test đều được tự chạy kiểm chứng trực tiếp trên môi trường máy cá nhân.
