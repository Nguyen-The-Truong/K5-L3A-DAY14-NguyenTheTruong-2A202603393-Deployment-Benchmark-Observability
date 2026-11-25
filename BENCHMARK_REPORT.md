# Benchmark & Observability Report — Day 14

> Mọi con số dưới đây được đo trực tiếp từ `load_test.py` và `eval/run_eval.py`. Phần 4 và Phần 5 sẽ được điền sau khi chạy Observability stack bằng Docker Compose (CP8).

**Họ tên:** Nguyễn Thế Trường (MSSV: 2A202603393)
**Ngày chạy:** 2026-11-25
**Môi trường:** Windows 11, Python 3.12.10 (CP7 local benchmark)

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Latency cơ bản của mock LLM (~120–180 ms) + retrieval (~20 ms); ngưỡng 500 ms đảm bảo 95% người dùng nhận phản hồi nhanh ngay cả khi tải cao |
| Error rate `/ask` | < 2.0 % | `MOCK_ERROR_RATE` mặc định là 1% (0.01); cho phép error budget 2% để tránh báo động giả do lỗi thoáng qua |
| Accuracy trên golden dataset | ≥ 85 % (mục tiêu sau fix: 100%) | Đủ cao để chặn regression chất lượng trong CI trước khi merge |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.17 | 202.1 | 233.8 | 1679.0 | 0.67% | PASS |
| 10 | 41.73 | 199.9 | 234.8 | 1685.4 | 1.33% | PASS |
| 50 | 118.29 | 201.8 | 257.9 | 1690.1 | 1.33% | PASS |
| 100 | 133.40 | 231.3 | 385.6 | 1818.0 | 1.33% | PASS |

**Nhận xét:** RPS không tăng tuyến tính theo concurrency: từ c=1 lên c=10 tăng gần 10× (4.17 → 41.73 req/s), nhưng từ c=50 lên c=100 chỉ tăng nhẹ (118.29 → 133.40 req/s) trong khi p50 và p95 bắt đầu tăng (p95 từ 257.9 ms lên 385.6 ms) vì chạm giới hạn CPU / event loop / xử lý socket của 1 process uvicorn trên local.

### 2.2 Tail latency (`--requests 300 --concurrency 20`)

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 198.1 | 236.6 | 244.4 |
| 0.03 (mặc định) | 224.6 | 236.7 | 1692.7 |
| 0.1 | 305.4 | 1685.5 | 1729.2 |

**Nhận xét:** `p99` (và khi tỉ lệ slow ≥ 5% thì cả `p95`) nhạy nhất với các request chậm bất thường: khi `MOCK_SLOW_RATE` tăng từ `0` lên `0.03`, `avg` chỉ tăng nhẹ (198.1 → 224.6 ms) nhưng `p99` nhảy vọt gần 7× (244.4 → 1692.7 ms). Vì giá trị trung bình (average) bị làm mượt bởi đa số request nhanh và che giấu trải nghiệm tệ của nhóm người dùng đuôi dài (tail latency), nên SLO phải đặt theo percentile (`p95` / `p99`).

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 57.2 | 108.7 | 113.7 |
| Tổng cost (USD) | $0.000406 | $0.000756 | $0.000790 |
| Cost / câu đúng (USD) | $0.00040575 | $0.00003599 | $0.00003436 |
| Quality gate | FAIL | PASS (≥ 80%) | PASS (≥ 100%, 0 regression) |

**Những câu v2 trả lời sai ban đầu và cách sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry về `jwt` (so sánh API key cố định với JWT có chữ ký số và thời điểm hết hạn) vào `KNOWLEDGE_BASE` trong `utils/mock_llm.py` |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry về `hpa` / `kubernetes` / `autoscal` (HPA scale theo CPU, memory, custom metrics) vào `KNOWLEDGE_BASE` trong `utils/mock_llm.py` |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**
Agent v2 thêm bước retrieval (RAG) khiến prompt dài hơn nên `avg_tokens` tăng gần gấp đôi (57.2 → 108.7–113.7) và tổng cost tăng gần gấp đôi (`$0.000406` → `$0.000790`), nhưng accuracy tăng vọt từ `4.3%` lên `100.0%` và **cost / câu đúng** giảm hơn **11×** (`$0.00040575` → `$0.00003436`). Ngoài ra, trong CI pipeline (`.github/workflows/eval.yml`), `run_eval.py --min-accuracy 0.85` trả exit code `1` nếu accuracy dưới ngưỡng hoặc có regression so với baseline, tự động chặn merge PR.

---

## 4. Observability stack (production)

- **Kiểm tra `promtool` unit test (`alert_rules_test.yml`):** `SUCCESS`
- **Trả lời Exercise 7.4 (quan sát từ `screenshots/grafana-dashboard.png` và stack):**
  1. **p95 và avg:** Lúc có tải (`MOCK_SLOW_RATE=0.3`), `Latency p95` đạt **1.87 s** trong khi `p50` ~200 ms và `avg` ~600–700 ms. Khoảng cách lớn giữa `p95` và `avg` phản ánh tail latency — đa số request vẫn nhanh nhưng một tỉ lệ request chậm kéo trải nghiệm đuôi dài lên gần 2 giây.
  2. **Panel "Requests/s theo instance":** Có — hai đường của `172.18.0.4:8000` và `172.18.0.6:8000` trùng khít nhau (~8 req/s mỗi instance), chứng tỏ Nginx chia đều traffic theo round-robin.
  3. **Vì sao `input tokens/s` lớn hơn `output tokens/s`:** Vì input gửi vào LLM gồm `SYSTEM_PROMPT` + context retrieval (RAG của agent v2) + câu hỏi (~90–120 tokens), dài hơn câu trả lời ngắn gọn ở output (~40–55 tokens).
  4. **`curl http://localhost:8080/metrics` trả về gì:** Trả về **HTTP 404** vì `nginx.conf` cấu hình `location = /metrics { return 404; }` để không lộ metrics nội bộ ra ngoài; chỉ Prometheus trong mạng `obs_net` mới scrape trực tiếp `agent:8000/metrics`.

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải (`screenshots/grafana-dashboard.png` — mở http://localhost:3000)
- [x] Một trace chậm trong Jaeger (`screenshots/jaeger-slow-trace.png` — trace `fdf12da3e66ddcbb11fa0f841aab6d5a`)
- [x] Một trace lỗi trong Jaeger (`screenshots/jaeger-error-trace.png` — trace `472583c0a1c80b01237fc525220431c1`)
- [x] Trang Alerts của Prometheus với ≥ 2 alert đang **firing** (`screenshots/prometheus-alert-firing.png` — mở http://localhost:9090/alerts)

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` | **80s** (pending từ 20s, `for: 1m`) | Error Rate (5xx %) |
| B | `MOCK_SLOW_RATE=0.3` | `AgentHighLatencyP95` | **140s** (pending từ 20s, `for: 2m`) | Latency p95 / p99 |
| E | `AGENT_VERSION=v1` | Không alert nào | N/A | Không đổi trên hạ tầng (cần theo dõi bằng eval) |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

Khi đổi sang `AGENT_VERSION=v1`, HTTP status vẫn là `200`, latency còn nhanh hơn (do bỏ bước retrieval) và không có lỗi 5xx nên các alert hạ tầng/SLO không kích hoạt. Để phát hiện suy giảm chất lượng, cần chạy **eval trên golden dataset** trong CI/CD trước khi deploy (`run_eval.py` với `--baseline` và `--min-accuracy`), kết hợp **eval định kỳ / LLM-as-judge** trên mẫu traffic thật và theo dõi tỉ lệ phản hồi tiêu cực từ người dùng.

### 4.3 Debug bằng trace

**Trace chậm nhất đã chọn:**

- `trace_id`: `fdf12da3e66ddcbb11fa0f841aab6d5a`
- Tổng thời gian: **1696.7 ms** (~1.7s)
- Span chiếm nhiều thời gian nhất: **`chat mock-llm-small`** (**1670.4 ms**, **98.4%**) — trong khi `agent.retrieve_context` chỉ mất 25.3 ms (1.5%)
- Dòng log tương ứng (grep theo `trace_id`):

```json
{"ts": "2026-11-25T09:50:15.473967+00:00", "level": "INFO", "service": "observable-agent", "instance": "af1d5720f567", "event": "llm_call", "trace_id": "fdf12da3e66ddcbb11fa0f841aab6d5a", "span_id": "b06c851707c7bcc0", "request_id": "beb7e55480f3", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 95, "output_tokens": 55, "cost_usd": 4.7e-05, "llm_ms": 1670.4}
```

**Trace lỗi đã chọn (bonus đối chiếu `llm_error`):**
- `trace_id`: `472583c0a1c80b01237fc525220431c1` (113.1 ms, HTTP 503)
```json
{"ts": "2026-11-25T09:50:14.647574+00:00", "level": "INFO", "service": "observable-agent", "instance": "af1d5720f567", "event": "llm_error", "trace_id": "472583c0a1c80b01237fc525220431c1", "span_id": "b7326f1f551f04fa", "request_id": "7381a519621d", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

Số liệu `Aggregated` trích từ `benchmark-results/scale_1_stats.csv` và `scale_3_stats.csv`:

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 65.82 | 190 | 230 | 1700 | 64 / 7820 (0.82%) |
| 3 | 66.31 | 190 | 220 | 1700 | 61 / 7882 (0.77%) |

**Giải thích kết quả:** Scale từ 1 lên 3 instance chỉ làm RPS nhích nhẹ (`65.82` → `66.31 req/s`) và `p95` giữ quanh `220–230 ms` vì:
1. Trong `locustfile.py`, mỗi user có `wait_time = between(0.5, 2)` (~1.25s think time) cộng ~0.23s latency nên 100 users tối đa chỉ tạo được ~67 req/s — **nút thắt nằm ở phía tải của Locust (think time)** chứ chưa làm bão hòa async event loop của agent.
2. Ngoài ra `p99` (~1700 ms) đến từ `MOCK_SLOW_RATE=0.03` nội tại của từng lần gọi `mock_llm`, nên thêm instance không làm giảm tail latency của 1 LLM call chậm.

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. `p95`/`p99` phản ánh tail latency chính xác hơn `average` khi có một tỉ lệ nhỏ request chậm bất thường.
  2. So sánh hai phiên bản agent cần đo song song cả hiệu năng (RPS, p95), chất lượng (eval trên golden dataset, chống regression) và chi phí thực tế (`cost / câu đúng`).
  3. Ba trụ cột Metrics (Prometheus/Grafana) – Traces (OpenTelemetry/Jaeger) – Logs (JSON có `trace_id`) liên kết chặt với nhau: thấy p95/error tăng trên Grafana → mở Jaeger tìm trace chậm/lỗi → lọc log theo `trace_id` để biết nguyên nhân chính xác.
- Nếu đưa agent này lên production thật, sẽ thêm metric/alert nào nữa:
  - Tỉ lệ cache hit / latency của Redis và rate-limit/cost-guard rejections (`429`, `402`).
  - Độ dài context/retrieval hit rate (số lần RAG không tìm thấy tài liệu liên quan) và đánh giá chất lượng định kỳ trên sample traffic thật.
