# Benchmark & Observability Report — Day 14

**Họ tên:** Dương Ngọc Tuyên — MSSV 2A202603511
**Ngày chạy:** 2026-11-25 (giờ ghi trong bài là UTC; giờ Việt Nam = UTC+7)
**Môi trường:** Laptop AMD Ryzen 7 5800H, 14 GB RAM, Windows 11. Docker Desktop 29.9.3 (VM: 16 vCPU / 6.7 GB). Bước 1 (develop) chạy Python 3.12 trực tiếp trên Windows; load generator và agent chạy chung một máy.

Bằng chứng: `reports/*.json` (load test + eval), `benchmark-results/scale_*_stats.csv` (Locust), `screenshots/` (Grafana, Jaeger, Prometheus).

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Ở tải bình thường (c ≤ 20) p95 đo được 234–468 ms, nên 500 ms là ngưỡng có biên nhỏ nhưng đạt được. Dùng p95 vì average che mất ~3% request chậm 1,7 s (mục 2.2). Alert đặt lỏng hơn (p95 > 1 s trong 2 phút) để tránh báo động vì spike ngắn. |
| Error rate `/ask` | < 2 % | Mock LLM tự lỗi ~1% (`MOCK_ERROR_RATE=0.01`); các lần chạy đo được 0–1,67%. SLO 2% chừa error budget cho lỗi upstream ngẫu nhiên. Alert ở 5% trong 1 phút. |
| Accuracy trên golden dataset | ≥ 85 % (CI), 100 % cho bản release | 85% là ngưỡng chặn merge trong CI (`.github/workflows/eval.yml`). Sau khi sửa KB, v2 đạt 23/23 nên bản release yêu cầu 100% và **không regression** so với baseline. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2, 300 request mỗi mức, `MOCK_SLOW_RATE=0.03`)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 3.94 | 202.4 | 234.3 | 1706.9 | 1.67% (5×503) | ✅ PASS |
| 10 | 43.87 | 187.9 | 234.2 | 1694.7 | 0.00% | ✅ PASS |
| 50 | 104.61 | 204.1 | 547.8 | 1800.1 | 0.33% (1×503) | ❌ FAIL (p95) |
| 100 | 54.32 | 667.8 | 3575.8 | 4033.5 | 0.67% (2×503) | ❌ FAIL (p95) |

File: `reports/load_v2_c1.json`, `load_v2_c10.json`, `load_v2_c50.json`, `load_v2_c100.json`.

**Nhận xét:** RPS chỉ tăng gần tuyến tính ở giai đoạn đầu: c=1 → c=10 tăng 11× (3.94 → 43.87), vì latency gần như không đổi (p50 ~190–200 ms) và throughput ≈ concurrency / latency. Từ c=10 → c=50 chỉ tăng 2,4× (104.61) và p95 bắt đầu vượt 500 ms. Ở c=100 RPS còn **giảm** xuống 54.32 và p50 nhảy lên 667.8 ms: hệ thống đã bão hoà. Agent là **một** process uvicorn trên Windows; mỗi request còn tốn CPU cho JSON log, Pydantic, đo metrics, và load generator chạy chung máy nên tranh CPU với agent. Khi request đến nhanh hơn tốc độ xử lý, chúng xếp hàng: latency tăng theo độ dài hàng đợi chứ throughput không tăng thêm (định luật Little). Điểm hoạt động tốt nhất là khoảng c ≈ 20–50; muốn hơn phải scale ngang (nhiều process/instance).

### 2.2 Tail latency (agent v2, `--requests 300 --concurrency 20`)

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 210.2 | 468.3 | 513.8 |
| 0.03 (mặc định) | 243.7 | 468.2 | 1704.4 |
| 0.1 | 355.9 | 1693.2 | 2020.5 |

File: `reports/load_v2_slow_0.json`, `load_v2_slow_0.03.json`, `load_v2_slow_0.1.json`.

**Nhận xét:** Từ 0 → 0.03, avg chỉ tăng 33.5 ms (+16%) nhưng **p99 tăng từ 513.8 lên 1704.4 ms (×3,3)**: 3% request chậm thêm 1,5 s nằm trọn trong top 1%. Từ 0.03 → 0.1, số request chậm (10%) vượt quá 5% nên **p95 cũng nhảy lên 1693.2 ms**, trong khi avg mới tăng thêm 112 ms. Percentile cao (p99, rồi p95) là chỉ số nhạy nhất với request chậm. SLO không nên đặt theo average vì average "pha loãng" nhóm user bị chậm vào số đông: với 0.1, avg 356 ms trông vẫn ổn, nhưng cứ 10 người thì 1 người phải chờ gần 2 giây. SLO theo p95/p99 cam kết trực tiếp trải nghiệm của nhóm tệ nhất.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.35% (1/23) | 91.3% (21/23) | 100% (23/23) |
| Avg tokens / câu | 57.2 | 109.1 | 113.9 |
| Tổng cost (USD) | 0.000405 | 0.000762 | 0.000793 |
| Cost / câu đúng (USD) | 0.00040515 | 0.00003627 | 0.00003448 |
| Quality gate | FAIL (gate 80%) | PASS (gate 80%, 0 regression so với v1) | PASS (gate 100%, 0 regression so với v2) |

File: `reports/eval_v1.json`, `reports/eval_v2.json`, `reports/eval_v2_fixed.json`. Lệnh cuối: `python eval/run_eval.py --url http://localhost:8002 --min-accuracy 1.0 --baseline reports/eval_v2.json --out reports/eval_v2_fixed.json` → `Quality gate (accuracy ≥ 100%, không regression): ✅ PASS`, "Câu được cải thiện: 2, Câu bị regression: 0".

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| security-02 ("JWT khác gì API key trong xác thực?") | `token`, `hết hạn`, `chữ ký` | Thêm entry KB kích hoạt bởi `["jwt"]`: JWT là *token* có *chữ ký* số, chứa thông tin user và thời điểm *hết hạn*; API key là chuỗi bí mật cố định không tự hết hạn. |
| scaling-03 ("Kubernetes HPA scale dựa trên những chỉ số nào?") | `cpu`, `memory`, `custom metrics` | Thêm entry KB kích hoạt bởi `["hpa", "kubernetes", "autoscal"]`: HPA tăng giảm số pod dựa trên CPU, memory hoặc custom metrics (RPS, độ dài hàng đợi). |

Trước khi thêm, tôi grep `golden_dataset.jsonl` để chắc rằng các từ kích hoạt mới (`jwt`, `hpa`, `kubernetes`, `autoscal`) không xuất hiện ở câu hỏi khác, vì retrieval chọn tài liệu có nhiều keyword trùng nhất và một từ khoá quá chung có thể "cướp" câu đang đúng, gây regression. Không sửa `golden_dataset.jsonl`. Đoạn sửa được chép sang `production/utils/mock_llm.py` để stack Docker cũng dùng KB mới.

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:** v2 thêm bước retrieval nên prompt dài gần gấp đôi (109.1 so với 57.2 token/câu), tổng cost tăng 88% (0.000762 so với 0.000405 USD) và latency p50 eval tăng ~46 ms (202.2 so với 156.5 ms: 20 ms retrieval + thời gian xử lý prompt dài hơn). Đổi lại accuracy tăng từ 4.35% lên 91.3%, nên **cost / câu đúng rẻ hơn ~11 lần** (0.0000363 so với 0.000405 USD). So "cost / câu" thì v1 rẻ hơn, nhưng một câu trả lời sai là vô giá trị; "cost / câu đúng" mới phản ánh chi phí để tạo ra giá trị. Sửa KB tốn thêm 4.8 token/câu (+4%) để có thêm 2 câu đúng nên cost / câu đúng còn giảm tiếp.

**CI (Exercise 7.3):** `.github/workflows/eval.yml` chạy agent v2 rồi `run_eval.py --min-accuracy 0.85 --baseline reports/eval_v2_fixed.json`. `run_eval.py` trả **exit code 1** khi accuracy < ngưỡng hoặc có regression, nên step fail, job fail; bật branch protection "Require status checks to pass" trên `main` thì PR không merge được. Kiểm chứng: chạy eval cho v1 với `--baseline reports/eval_v2.json` thì exit code = 1 ("Câu bị regression: 20", gate FAIL); bản v2 đã sửa thì exit code = 0.

---

## 4. Observability stack (production)

Stack bật bằng `docker compose -p k5-day14-observability up -d --build`. Lỗi gặp phải: container agent crash-loop với `ModuleNotFoundError: No module named 'uvicorn'`, vì Dockerfile copy package `--user` sang `/home/agent/.local` nhưng user `agent` có home là `/app` (`useradd -d /app`), nên Python không tìm thấy. Đã sửa bằng một dòng `ENV PYTHONUSERBASE=/home/agent/.local` trong `production/Dockerfile`. `promtool test rules` trên `alert_rules_test.yml`: **SUCCESS**.

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải: [`screenshots/grafana-dashboard.png`](screenshots/grafana-dashboard.png). Locust 30 user, 2 instance: Traffic 14.7 req/s, Error rate 0.53%, p95 294.67 ms, 2 instance up. Thêm [`grafana-chaos-A-B.png`](screenshots/grafana-chaos-A-B.png) chụp lúc đang chạy thí nghiệm A + B.
- [x] Một trace chậm trong Jaeger: [`screenshots/jaeger-slow-trace.png`](screenshots/jaeger-slow-trace.png)
- [x] Một trace lỗi trong Jaeger (có event `exception`): [`screenshots/jaeger-error-trace.png`](screenshots/jaeger-error-trace.png)
- [x] Trang Alerts của Prometheus với 2 alert cùng **firing** (`AgentHighErrorRate` + `AgentHighLatencyP95`): [`screenshots/prometheus-alert-firing.png`](screenshots/prometheus-alert-firing.png)

### 4.2 Thí nghiệm "phá hệ thống"

Tải nền: Locust headless 30 user / ramp 5 chạy suốt bài; 2 instance agent. Thời gian đo bằng script poll `GET /api/v1/alerts` mỗi 2 giây, tính từ lúc chạy `docker compose up -d` với `.env` mới (hoặc `docker pause`), đối chiếu với `activeAt` của Prometheus.

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` (07:47:12) | `AgentHighErrorRate` (giá trị 21.4%) | pending sau 25 s, **firing sau ~85 s** (activeAt 07:47:37 + `for: 1m`) | Error rate (đỏ), LLM calls theo kết quả (error tăng), Requests/s theo status (503) |
| B | thêm `MOCK_SLOW_RATE=0.3` (07:49:10, giữ A) | `AgentHighLatencyP95` (p95 = 1.89 s) | pending sau 29 s, **firing sau 149 s** (`for: 2m`) | Latency p50/p95/p99: p95/p99 vọt lên ~1,9 s còn p50 gần như giữ nguyên |
| C | `LLM_MODEL=mock-llm-large` (07:53:55) | `LLMCostBurnRateHigh` | pending sau 23 s, **firing sau 144 s** | Cost burn rate: ~2.0 → **34.8 USD/giờ (~17×)**; LLM latency p95 0.197 → 0.497 s (×2,5) |
| D | `docker pause k5-day14-observability-agent-1` (07:52:53) | `AgentInstanceDown` (`up{instance="172.19.0.7:8000"} = 0`) | pending sau 11 s, **firing sau 41 s** (`for: 30s`) | Instances up 2 → 1. Panel Error rate của agent vẫn xanh (0.89%) dù Nginx ghi 29 response 504 và Locust thấy lỗi |
| E | `AGENT_VERSION=v1` (07:56:39) | **Không có** alert mới (theo dõi 200 s) | — | Retrieval hit ratio: 16 hit/s → không còn series; Tokens/s giảm một nửa (input 1234 → 666/s); bảng version đổi sang v1 |

Ghi chú trung thực về cost alert: `LLMCostBurnRateHigh` (ngưỡng $2/giờ trên cửa sổ 5 phút) **đã firing từ trước thí nghiệm C**, ngay sau lượt benchmark 100 user ở mục 5 (activeAt 07:35:17); với 30 user, model nhỏ cũng đốt ~$2/giờ, sát ngưỡng. Nó hết firing khi A/B làm throughput giảm, nên trước C trạng thái là inactive (1m rate ≈ $2.0/giờ, 5m rate $1.54/giờ) và thời gian 144 s ở trên là thật. Trong E, cost alert vẫn hiện firing là do cửa sổ 5 phút còn chứa chi phí của model lớn ở C, không phải do E. Còn `AgentInstanceDown` *pending* thoáng qua ở giây thứ 5 là lúc container bị tạo lại; nó không lên firing. Bài học: ngưỡng $2/giờ quá thấp so với tải thật, sẽ gây alert fatigue; nên đặt theo budget thực tế, ví dụ dựa trên `DAILY_BUDGET_USD`.

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

Metric hạ tầng (latency, error, saturation) chỉ cho biết agent trả lời *nhanh và không lỗi*, không cho biết trả lời *đúng*. v1 thậm chí nhanh và rẻ hơn nên mọi panel đều "đẹp hơn". Cần giám sát chất lượng:
1. **Chặn trước khi deploy:** eval golden dataset trong CI (đã làm ở mục 3). v1 so với baseline v2 cho exit code 1 với 20 regression, nên bản này không bao giờ được merge.
2. **Online eval / canary:** định kỳ (vd mỗi 5 phút) chạy một tập câu hỏi chuẩn vào production qua synthetic check, xuất metric `agent_eval_accuracy` rồi alert khi accuracy < 85%. Cũng có thể chấm mẫu ngẫu nhiên traffic thật bằng LLM-as-judge.
3. **Proxy metric về chất lượng:** alert khi `agent_retrieval_total{result="hit"}` / tổng giảm mạnh, hoặc khi `agent_info` báo version khác version đã release. Trong E, hit ratio rơi về 0 là tín hiệu rõ nhất.
4. **Tín hiệu từ người dùng:** tỉ lệ thumbs-down, tỉ lệ hỏi lại hoặc rephrase.

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất tìm được (Jaeger, lọc `minDuration=1s`, chọn trace dài nhất trong 500 trace):

- `trace_id`: `c96b8335ac1b832c100b304e864f38db`
- Tổng thời gian: 1864.4 ms (span gốc `POST /ask`)
- Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (1737.4 ms, **93.2%** tổng thời gian). `agent.retrieve_context` chỉ mất 19.5 ms. Attributes: `gen_ai.request.model=mock-llm-small`, `gen_ai.usage.input_tokens=70`, `gen_ai.usage.output_tokens=33`. Kết luận: chậm do upstream LLM (request rơi vào nhóm "slow" của mock), không phải code của agent hay retrieval.
- Dòng log tương ứng (`docker compose -p k5-day14-observability logs agent | grep c96b8335ac1b832c100b304e864f38db`):

```json
{"ts": "2026-11-25T07:34:04.452971+00:00", "level": "INFO", "service": "observable-agent", "instance": "2fc0887baebb", "event": "llm_call", "trace_id": "c96b8335ac1b832c100b304e864f38db", "span_id": "45646687df98fc54", "request_id": "3a5b5ba0842c", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 70, "output_tokens": 33, "cost_usd": 3e-05, "llm_ms": 1691.3}
{"ts": "2026-11-25T07:34:04.555176+00:00", "level": "INFO", "service": "observable-agent", "instance": "2fc0887baebb", "event": "http_request", "trace_id": "c96b8335ac1b832c100b304e864f38db", "span_id": "45646687df98fc54", "request_id": "3a5b5ba0842c", "method": "POST", "route": "/ask", "status": 200, "ms": 1861.4}
```

Trace lỗi: `e8b76890bac63305c9398d1463714d81` (120 ms, HTTP 503). Span `chat mock-llm-small` có `error.type=upstream_unavailable`, status ERROR và event `exception` với `exception.message = Upstream LLM error: 503 Service Unavailable (mock)`. Dòng log khớp:

```json
{"ts": "2026-11-25T07:40:55.150497+00:00", "level": "INFO", "service": "observable-agent", "instance": "93f02c210d14", "event": "llm_error", "trace_id": "e8b76890bac63305c9398d1463714d81", "span_id": "710e6026427c8ddf", "request_id": "0e13dcf68c08", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
```

### 4.4 Trả lời Exercise 7.4

Số liệu lấy từ Prometheus lúc chụp dashboard (Locust 30 user, 2 instance):

1. **p95 và avg?** avg = 232.6 ms, p95 = 293.1 ms, còn p99 = 1835.6 ms. avg và p95 gần nhau nghĩa là phần lớn request đồng đều (~190–230 ms). Nhưng p99 cao gấp ~8 lần avg cho thấy có một nhóm nhỏ (~3%) rất chậm (tail latency do `MOCK_SLOW_RATE`). Khoảng cách giữa avg và các percentile cao cho biết phân bố bị lệch đuôi; avg không cho thấy điều đó.
2. **Nginx có chia đều traffic?** Có: `rate(http_requests_total{route="/ask"}[1m])` theo instance là 9.24 và 9.29 req/s (lệch < 1%), đúng kiểu round-robin.
3. **Vì sao input tokens/s > output tokens/s?** 1419 so với 747 token/s. Input = system prompt + tài liệu retrieval (context RAG) + câu hỏi, còn output chỉ là câu trả lời ngắn. RAG làm prompt dài ra, đó cũng là lý do v2 tốn token gấp đôi v1 (khi đổi sang v1 ở thí nghiệm E, input giảm từ 1234 xuống 666/s).
4. **`curl http://localhost:8080/metrics`?** Trả **404**. `nginx.conf` chặn `/metrics` ở cổng public: metrics lộ thông tin nội bộ (số request, cost, version, model) và ai cũng có thể spam endpoint này. Prometheus scrape thẳng từng container qua mạng nội bộ Docker (`agent:8000/metrics`, DNS service discovery) nên không cần đi qua Nginx.

### 4.5 Trả lời Exercise 7.1, 7.5, 7.6

- **7.1 Metric types** (từ `GET /metrics` của v2): Counter `http_requests_total{method="POST",route="/ask",status="200"} 1.0` và `llm_tokens_total{type="input"} 91.0`; Gauge `agent_in_flight_requests 0.0`, `agent_info{model="mock-llm-small",version="v2"} 1.0`; Histogram `http_request_duration_seconds_bucket{le="0.2",...}`, `llm_request_duration_seconds_*`.
- **7.1 Vì sao dùng `route.path` thay vì `request.url.path`?** Mỗi giá trị label khác nhau tạo một time series. Bot quét hàng nghìn URL lạ (`/wp-admin`, `/.env`, `/a1b2c3`...) sẽ sinh hàng nghìn series mới cho mỗi metric (nhân tiếp với status và các bucket histogram), khiến Prometheus tốn RAM và query chậm, có thể bị OOM (tấn công cardinality). Dùng template route (và gom URL lạ vào `unmatched`) giữ số series cố định.
- **7.1 Đã thêm metric** `agent_question_length_chars` (Histogram, bucket 20–2000) vào `develop/app.py`. Kiểm tra: sau 1 câu hỏi 40 ký tự thì `agent_question_length_chars_count 1.0`, `_sum 40.0`. Dùng Histogram chứ không dùng label vì độ dài có vô số giá trị.
- **7.5** Trace chậm nhất có span `chat mock-llm-small` chiếm 93%; trace lỗi có event `exception`; tra từ log `llm_error` sang trace qua `trace_id` (mục 4.3).
- **7.6** Thời gian tới lúc firing = thời gian để biểu thức vượt ngưỡng (phụ thuộc cửa sổ `rate(...[1m])` hoặc `[5m]` và scrape 5 s) + thời gian `for`. Vì vậy A (for 1m) mất ~85 s, D (for 30s) mất 41 s, còn B và C (for 2m) mất ~145 s. Vì sao lỗi 504 ở D không hiện trên metric agent: lỗi xảy ra ở Nginx (hết `proxy_read_timeout` khi chờ instance bị pause), request chưa từng được agent xử lý xong nên metric của agent không đếm. Metric chỉ phản ánh những gì chính service đó nhìn thấy, nên production cần giám sát cả tầng load balancer hoặc synthetic check từ bên ngoài. Dùng `pause` thay `stop` vì container stop biến mất khỏi DNS, Prometheus coi như target không còn tồn tại (không phải down) nên không có alert.

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS (`/ask`) | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 59.66 | 190 | 230 | 1700 | 69 / 7133 (0.97%) |
| 3 | 59.30 | 190 | 230 | 1700 | 63 / 7083 (0.89%) |

File: `benchmark-results/scale_1_stats.csv`, `benchmark-results/scale_3_stats.csv` (dòng `/ask`; dòng Aggregated tính cả `/health`: 65.81 và 65.89 req/s). Failures đều là 503 từ lỗi upstream giả lập (~1%).

**Giải thích kết quả:** Scale từ 1 lên 3 instance **không cải thiện** p95 hay RPS (chênh lệch nằm trong nhiễu đo). Lý do:
1. **RPS bị giới hạn bởi tải, không phải bởi agent.** Locust có 100 user, mỗi user nghỉ 0,5–2 s (trung bình 1,25 s) giữa các request, nên tải tối đa ≈ 100 / (1,25 + 0,21) ≈ 68 req/s. Cả hai lần đều đạt ~66 req/s tổng, tức là agent đã phục vụ hết số request được gửi tới; thêm instance không có thêm request để xử lý.
2. **Latency do LLM quyết định.** Mỗi request mất ~190 ms gần như hoàn toàn trong lời gọi LLM (`await asyncio.sleep` trong mock, không tốn CPU). Panel LLM latency p95 ≈ 0.2 s, đúng bằng p95 của HTTP. Agent async nên một instance (giới hạn 0,5 CPU) vẫn giữ hàng trăm request đang chờ LLM cùng lúc mà chưa bão hoà. Thêm instance không làm LLM trả lời nhanh hơn.
3. p99 = 1700 ms ở cả hai lần vì 3% request chậm nằm ở upstream LLM; scale ngang không sửa được tail latency của dependency.

Nút thắt nằm ở **upstream LLM** (latency của model) và ở **mức tải sinh ra**. Scale ngang chỉ có ích khi instance bị bão hoà CPU/hàng đợi (như mục 2.1 ở c=100 trên một process). Muốn thấy khác biệt phải tăng user hoặc giảm wait time tới mức 1 instance quá tải. Muốn giảm p95/p99 thật thì phải đụng vào LLM: model nhanh hơn, cache câu trả lời, timeout + retry/hedging, hoặc streaming để giảm thời gian tới token đầu tiên.

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. **Average nói dối:** với 10% request chậm, avg 356 ms trông ổn trong khi p95 là 1,7 s. SLO và alert phải theo percentile, và chỉ có ý nghĩa khi đo bằng load test đủ mẫu (eval 23 câu thì p95 nhảy lung tung vì 1 câu chậm).
  2. **Đo trước khi scale:** benchmark scaling cho thấy thêm instance không giúp gì khi nút thắt là LLM upstream hoặc tải chưa đủ; còn load test c=100 lại cho thấy một process sẽ bão hoà. Phải biết nút thắt ở đâu (trace cho thấy 93% thời gian nằm ở span LLM) rồi mới chọn giải pháp.
  3. **Hạ tầng xanh ≠ agent đúng:** đổi sang v1 hay pause một instance đều có chỗ metric của agent không phát hiện được (chất lượng giảm; 504 ở Nginx). Cần cả 3 trụ cột (metric → trace → log nối bằng `trace_id`), eval trong CI và giám sát từ phía ngoài.
- Nếu đưa agent này lên production thật, tôi sẽ thêm:
  - Metric chất lượng: `agent_eval_accuracy` từ synthetic eval định kỳ, alert < 85%; alert khi retrieval hit ratio < 80%.
  - Synthetic check từ bên ngoài (blackbox exporter) vào URL public, alert khi status khác 200 hoặc latency cao. Thêm metric ở tầng Nginx/LB (tỉ lệ 5xx/504) để bắt lỗi mà agent không thấy.
  - Alert theo **error-budget burn rate** nhiều cửa sổ (vd 5m và 1h) thay vì ngưỡng cố định, giảm alert fatigue.
  - Cost alert theo budget ngày thật (liên kết `DAILY_BUDGET_USD` của cost guard Part 6), cảnh báo ở 80% budget; metric `agent_rate_limited_total` (429) và `agent_budget_exceeded_total` (402).
  - Saturation: event loop lag, CPU/memory container, độ dài hàng đợi; TTFT (time to first token) khi dùng streaming.
