# Day 14 Lab — Mission Answers

> Mọi output dưới đây là output thật, chạy ngày 2026-11-25 trên Windows 11 + Docker Desktop 29.9.3. Token và key đã che.

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns trong `01-localhost-vs-production/develop/app.py`

1. **Hardcode secret** (dòng 17–18): `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@..."`. Push lên GitHub là lộ key và mật khẩu DB; muốn đổi key thì phải sửa code và deploy lại.
2. **Log ra secret** (dòng 34–35): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`. Log thật khi chạy in ra `[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this`, nên ai đọc được log (log aggregator, người support) cũng lấy được key.
3. **`print()` thay vì logging có cấu trúc** (dòng 34–38): không có level, timestamp, request id; không parse hay lọc được trong Loki/Datadog. Câu trả lời đầy đủ của user cũng bị in ra (rò rỉ dữ liệu).
4. **Config cứng trong code, không đọc env** (dòng 21–22): `DEBUG = True`, `MAX_TOKENS = 500`. Không đổi được giữa dev, staging và prod nếu không sửa code.
5. **Không có health check**: `GET /health` trả **404** (đã thử). Platform không biết agent còn sống hay không để restart.
6. **Bind `host="localhost"`** (dòng 51): trong container chỉ nghe loopback, nên traffic từ bên ngoài container không vào được.
7. **Port cứng `port=8000`** (dòng 52): Railway/Render cấp port qua biến `PORT`; port cứng làm deploy fail healthcheck.
8. **`reload=True` trong production** (dòng 53): chạy thêm process theo dõi file, tốn tài nguyên và có thể restart giữa chừng.
9. **Không xử lý shutdown**: không có lifespan hay SIGTERM handler, nên không có chỗ đóng kết nối hay dọn dẹp khi platform tắt container.
10. **Không validate input**: `question: str` là query param, không giới hạn độ dài và không có body model.

### Exercise 1.2: Chạy bản develop

```text
$ curl -X POST "http://localhost:8000/ask?question=Hello"
HTTP/1.1 200 OK
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}

$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 422 Unprocessable Entity
{"detail":[{"type":"missing","loc":["query","question"],"msg":"Field required","input":null}]}

$ curl http://localhost:8000/health
HTTP/1.1 404 Not Found
{"detail":"Not Found"}
```

Log server: `[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this` (lộ secret trong log). **Chạy được, nhưng chưa production-ready:** không health check, lộ secret, chỉ nghe `localhost:8000`.

### Exercise 1.3: Bảng so sánh

| Feature | Develop | Production | Tại sao quan trọng? |
|---|---|---|---|
| Config | Hardcode trong code (`DEBUG = True`, key, DB URL) | `config.py`: mọi giá trị đọc từ env (`os.getenv("PORT")`, `AGENT_API_KEY`...), `validate()` fail fast nếu thiếu `AGENT_API_KEY` ở production | Cùng một image chạy được dev/staging/prod chỉ bằng cách đổi env; secret không nằm trong git; đổi config không cần build lại |
| Secrets | Ghi thẳng trong code và in ra log | Lấy từ env/`.env` (đã gitignore), không log secret, chỉ log `question_length` | Lộ key thì mất tiền hoặc bị tấn công; xoá khỏi code chưa đủ vì key vẫn nằm trong lịch sử git |
| Health check | Không có (`/health` → 404) | `GET /health` (liveness: uptime, version) + `GET /ready` (readiness, 503 khi chưa sẵn sàng); đã test cả hai đều 200 | Platform/LB dùng `/health` để restart container chết và `/ready` để chỉ route traffic tới instance sẵn sàng |
| Logging | `print()` text tự do | JSON (`logging` với format JSON, `json.dumps({"event": ...})`) | Log aggregator parse, lọc, đếm được theo field; có level, timestamp. *Quan sát thêm:* khi chạy, các dòng JSON không xuất hiện vì `config.validate()` gọi `logging.warning()` trước, làm root logger bị cấu hình mặc định (level WARNING) trước khi `app.py` gọi `basicConfig`. Bài học: cấu hình logging phải chạy đầu tiên |
| Shutdown | Đột ngột, không có hook | `lifespan` (startup/shutdown) + handler SIGTERM; uvicorn ngừng nhận request mới và chờ request đang chạy | Không cắt ngang request của user khi deploy hay scale down; đóng kết nối DB/Redis sạch sẽ |
| Host / Port | `localhost:8000` cứng | `0.0.0.0` và `PORT` từ env | Container nhận được traffic từ bên ngoài; platform tự cấp port |
| Reload / Debug | `reload=True` luôn bật | `reload=settings.debug` (chỉ khi `DEBUG=true`) | Production không chạy file watcher, không lộ thông tin debug |
| Input | Query param, không validate | Body JSON, trả 422 nếu thiếu `question` | Lỗi rõ ràng cho client, chặn input xấu |

Kết quả gọi bản production:

```text
$ curl http://localhost:8000/health
HTTP/1.1 200 OK
{"status":"ok","uptime_seconds":5.5,"version":"1.0.0","environment":"development","timestamp":"2026-11-25T07:16:02.854540+00:00"}
$ curl http://localhost:8000/ready
HTTP/1.1 200 OK
{"ready":true}
$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 200 OK
{"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","model":"gpt-4o-mini"}
```

---

## Part 2: Docker

### Exercise 2.1: Câu hỏi về `02-docker/develop/Dockerfile`

1. **Base image:** `python:3.11`, bản Debian đầy đủ (có compiler, header, nhiều tool), nên rất nặng (~1 GB).
2. **Working directory:** `/app`. Mọi lệnh `COPY`, `RUN`, `CMD` sau đó chạy tương đối với thư mục này.
3. **Vì sao COPY `requirements.txt` trước:** Docker cache theo layer. Requirements ít đổi còn code đổi liên tục. Copy requirements rồi `pip install` trước thì sửa code chỉ làm mất cache từ bước `COPY app.py` trở đi, bước cài dependency (chậm nhất) được dùng lại. Nếu `COPY . .` trước thì mỗi lần sửa một dòng code là cài lại toàn bộ thư viện.
4. **CMD vs ENTRYPOINT:** `CMD` là lệnh/tham số mặc định, bị ghi đè khi chạy `docker run image <lệnh khác>`. `ENTRYPOINT` là chương trình cố định của container; tham số của `docker run` được nối thêm vào (muốn đổi phải dùng `--entrypoint`). Hay dùng kết hợp: `ENTRYPOINT ["uvicorn"]` + `CMD ["app:app", "--port", "8000"]`. Nên dùng exec form (`["..."]`) để process là PID 1 và nhận được SIGTERM.

### Exercise 2.2: Build và chạy (từ gốc repo)

```text
$ docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
$ docker run -p 8000:8000 my-agent:develop
$ curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker?"
HTTP/1.1 200 OK
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Multi-stage build và kích thước image

- **Stage 1 (`builder`, `python:3.11-slim`):** cài `gcc`, `libpq-dev` rồi `pip install --user -r requirements.txt`. Stage này có compiler và cache apt, chỉ dùng để build, **không** được deploy.
- **Stage 2 (`runtime`, `python:3.11-slim`):** tạo user non-root `appuser`, chỉ `COPY --from=builder /root/.local` (các package đã cài) cùng code, có `HEALTHCHECK`, chạy uvicorn 2 worker.
- **Vì sao nhỏ hơn:** image cuối không chứa gcc, header, apt lists hay pip cache của stage builder, và dùng base `slim` thay vì bản full.

```text
$ docker images my-agent
REPOSITORY   TAG        SIZE
my-agent     advanced   240MB
my-agent     develop    1.67GB
$ docker image inspect my-agent:develop my-agent:advanced --format '{{index .RepoTags 0}} {{.Size}}'
my-agent:develop 1667784216
my-agent:advanced 239811436
```

- Develop: **1667.8 MB** (1,667,784,216 bytes)
- Production: **239.8 MB** (239,811,436 bytes)
- Chênh lệch: **giảm 85.6%** (production chỉ bằng 14.4% develop)

### Exercise 2.4: Docker Compose stack (`02-docker/production`)

```text
                      ┌──────────────────────────── network "internal" (bridge) ─────────────────────────┐
Client ──:80/:443──▶  │  nginx (nginx:alpine)  ──proxy_pass──▶  agent:8000 (build Dockerfile, healthcheck)│
                      │   rate limit 10r/s,                       │ REDIS_URL=redis://redis:6379/0          │
                      │   security headers                         │ QDRANT_URL=http://qdrant:6333           │
                      │                                            ├──▶ redis:6379  (redis:7-alpine, volume)  │
                      │                                            └──▶ qdrant:6333 (qdrant v1.9.0, volume)   │
                      └──────────────────────────────────────────────────────────────────────────────────┘
```

- **Services được start:** `agent`, `redis`, `qdrant`, `nginx`. Chỉ Nginx publish port ra host (80, 443); agent, redis, qdrant chỉ có trong mạng `internal`.
- **Giao tiếp:** các container gọi nhau bằng tên service qua DNS của Docker (`agent:8000`, `redis:6379`, `qdrant:6333`). `depends_on: condition: service_healthy` đảm bảo redis và qdrant healthy trước khi agent start.
- *Quan sát:* `main.py` của agent hiện chưa thực sự dùng Redis hay Qdrant (chỉ nhận biến môi trường); trong stack đó chúng đóng vai trò minh hoạ kiến trúc.

```text
$ docker compose -p k5-day14-docker ps
SERVICE   IMAGE                   STATUS                    PORTS
agent     k5-day14-docker-agent   Up 15 seconds (healthy)   8000/tcp
nginx     nginx:alpine            Up 15 seconds             0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
qdrant    qdrant/qdrant:v1.9.0    Up 26 seconds (healthy)   6333-6334/tcp
redis     redis:7-alpine          Up 26 seconds (healthy)   6379/tcp

$ curl -i http://localhost/health
HTTP/1.1 200 OK
Server: nginx
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
{"status":"ok","uptime_seconds":13.6,"version":"2.0.0","timestamp":"2026-11-25T07:19:45.886798"}

$ curl -i -X POST http://localhost/ask -H "Content-Type: application/json" -d '{"question":"Explain microservices"}'
HTTP/1.1 200 OK
Server: nginx
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

Header `Server: nginx` và các security header cho thấy request đi qua Nginx trước khi tới agent.

---

## Part 3: Cloud Deployment

> Lớp K5 deploy **agent Part 6** (`06-lab-complete/`) thay vì app mẫu. Chi tiết: [DEPLOYMENT.md](DEPLOYMENT.md).

### Exercise 3.1 / 3.2: Deploy

- **Nền tảng chính:** **Render** (Exercise 3.2, Blueprint `06-lab-complete/render.yaml`): web service `day14-agent` (Docker) + Key Value `day14-redis`.
- **URL:** **https://day14-agent-pm39.onrender.com**
- **Kết quả 4 lệnh tự kiểm ở CP6** (2026-11-25):

```text
$ curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"
200
$ curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"
200
$ curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'
401
$ curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'
200
```

- Rate limit trên cloud: 12 request liên tiếp → `200 ×8, 429 ×4` (cộng 2 request trước đó trong cùng phút là đúng 10/phút; state trong Redis của Render).
- `check_production_ready.py --public-url https://day14-agent-pm39.onrender.com` → `✅ Public URL: GET .../health → 200`.
- Railway: đã thử deploy từ GitHub nhưng Railway báo `Invalid service name` vì lấy tên repo (quá dài) làm tên service; Railway CLI thì bị Windows Smart App Control chặn. Vì vậy chọn Render làm nền tảng chính. `railway.toml` vẫn được giữ đúng chuẩn (script chấm ✅).
- Screenshot: [`screenshots/dashboard.png`](screenshots/dashboard.png), [`screenshots/running.png`](screenshots/running.png), [`screenshots/test.png`](screenshots/test.png). Chi tiết: [DEPLOYMENT.md](DEPLOYMENT.md).

### So sánh `railway.toml` với `render.yaml` (của `06-lab-complete/`)

| | `railway.toml` | `render.yaml` |
|---|---|---|
| Định dạng / phạm vi | TOML, cấu hình **một service** (build + deploy). Project, plugin Redis và biến môi trường tạo trên dashboard hoặc CLI | YAML Blueprint, mô tả **cả hạ tầng** (IaC): web service **và** Key Value (Redis) trong một file |
| Build | `[build] builder = "DOCKERFILE"`, `dockerfilePath` | `runtime: docker`, `rootDir: 06-lab-complete`, `dockerfilePath` |
| Start / port | Không có `startCommand`: dùng `CMD` của Dockerfile, đọc `$PORT` do Railway cấp | Cũng dùng `CMD` của Dockerfile; Render cấp `PORT` |
| Health check | `healthcheckPath = "/health"`, `healthcheckTimeout = 60` | `healthCheckPath: /health` |
| Restart | `restartPolicyType = "ON_FAILURE"`, tối đa 3 lần | Render tự restart, không cấu hình trong file |
| Secret | Không nằm trong file: `railway variables set AGENT_API_KEY=...` | Khai báo tên trong file nhưng không có giá trị: `sync: false` (nhập tay trên dashboard) hoặc `generateValue: true` |
| Redis | Thêm Redis service trong project, đặt `REDIS_URL=${{Redis.REDIS_URL}}` | `fromService: {type: keyvalue, name: day14-redis, property: connectionString}`: Render tự nối `REDIS_URL` |
| Region / plan | Chọn trên dashboard | `region: singapore`, `plan: free` ngay trong file |

**Vì sao khác nhau:** Railway theo hướng "CLI-first": file chỉ cấu hình cách build và chạy một service, phần còn lại là trạng thái của project trên Railway. Render Blueprint theo hướng Infrastructure-as-Code: commit một file là tái tạo được toàn bộ stack (kể cả Redis và liên kết biến), nên phải có cơ chế khai báo secret mà không lộ giá trị (`sync: false` / `generateValue`).

### Exercise 3.3: Pipeline Cloud Run (`03-cloud-deployment/production-cloud-run/`)

1. `cloudbuild.yaml` chạy trên Cloud Build khi push lên `main`. Bước **test**: trong image `python:3.11-slim`, cài requirements và chạy `pytest tests/`; test fail thì pipeline dừng.
2. Bước **build**: `docker build` tag theo `$COMMIT_SHA` và `latest`, dùng `--cache-from` image `latest` để tận dụng layer cache. Bước **push** đẩy cả hai tag lên Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
3. Bước **deploy**: `gcloud run deploy ai-agent --image=...:$COMMIT_SHA` ở `asia-southeast1` với min 1 / max 10 instance, 512Mi, 1 CPU, timeout 60s, `ENVIRONMENT=production`; `OPENAI_API_KEY` lấy từ Secret Manager (`--set-secrets`), không ghi trong code.
4. `service.yaml` là phiên bản IaC (Knative) của service: autoscaling min 1 / max 10, `containerConcurrency: 80`, giới hạn tài nguyên, secret qua `secretKeyRef`, `livenessProbe` gọi `/health` và `startupProbe` gọi `/ready`. Áp dụng bằng `gcloud run services replace service.yaml`.
5. Mỗi bước chờ bước trước (`waitFor`). Deploy theo `$COMMIT_SHA` nên mỗi revision truy ngược được về commit và rollback được.

---

## Part 4: API Security

### Exercise 4.1: API key (`04-api-gateway/develop`, chạy `AGENT_API_KEY=secret-key-123 python app.py`)

- **Check ở đâu:** dependency `verify_api_key()` đọc header `X-API-Key` qua `APIKeyHeader(name="X-API-Key", auto_error=False)`, được gắn vào `/ask` bằng `Depends(verify_api_key)`.
- **Sai key thì sao:** thiếu key → **401** `Missing API key`; sai key → **403** `Invalid API key.` (Part 6 yêu cầu cả hai trường hợp đều 401.)
- **Rotate key:** key đọc từ env `AGENT_API_KEY`, nên đổi giá trị biến trên platform rồi restart hoặc redeploy, không phải sửa code. Muốn không downtime thì cho phép tạm thời 2 key (cũ + mới), chuyển client sang key mới rồi gỡ key cũ.

```text
$ curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

$ curl -i -X POST http://localhost:8000/ask -H "X-API-Key: wrong-key" -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 403 Forbidden
{"detail":"Invalid API key."}

$ curl -i -X POST http://localhost:8000/ask -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 200 OK
{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

### Exercise 4.2: JWT (`04-api-gateway/production`)

**JWT flow** (`auth.py`): `POST /auth/token` kiểm tra username/password → `create_token()` ký payload `{sub, role, iat, exp = +60 phút}` bằng HS256 với `JWT_SECRET` → client gửi `Authorization: Bearer <token>` → `verify_token()` xác minh chữ ký và hạn, lấy `username` và `role` mà **không cần tra DB** (stateless). Hết hạn → 401, chữ ký sai → 403.

```text
$ curl -X POST http://localhost:8000/auth/token -H "Content-Type: application/json" -d '{"username":"student","password":"demo123"}'
{"access_token":"eyJ***MASKED***","token_type":"bearer","expires_in_minutes":60,"hint":"Include in header: Authorization: Bearer eyJ***..."}

$ curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Explain JWT"}'
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}
```

### Exercise 4.3: Rate limiting

- **Algorithm:** sliding window log. Mỗi user có một `deque` timestamp; mỗi request xoá các timestamp cũ hơn 60 s rồi đếm số còn lại.
- **Limit:** user thường 10 request/phút (`rate_limiter_user`), admin 100 request/phút (`rate_limiter_admin`).
- **Bypass cho admin:** `app.py` chọn `limiter = rate_limiter_admin if role == "admin" else rate_limiter_user`, nên admin (`teacher`) có hạn mức riêng cao hơn. Muốn bỏ hẳn giới hạn thì bỏ qua `limiter.check()` khi `role == "admin"` (vẫn nên giữ một trần để chống token admin bị lộ).
- Hạn chế: state nằm trong RAM, nên nhiều instance hoặc restart sẽ đếm sai. Part 6 đã chuyển sang Redis.

Vòng lặp 20 request với token `student` (đã lọc chỉ giữ status, header và body):

```text
--- Request 1
HTTP/1.1 200 OK
{"question":"Test 1","answer":"...","usage":{"requests_remaining":9,"budget_remaining_usd":1.6e-05}}
--- Request 2 … Request 9: HTTP/1.1 200 OK (requests_remaining 8 → 1)
--- Request 10
HTTP/1.1 200 OK
{"question":"Test 10","answer":"...","usage":{"requests_remaining":0,"budget_remaining_usd":0.000186}}
--- Request 11
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: 1795591279
retry-after: 59
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
--- Request 12 … Request 20: HTTP/1.1 429 Too Many Requests (retry-after 59 → 58)
```

10 request đầu 200, từ request 11 trở đi 429 kèm `Retry-After`. *Quan sát:* field `budget_remaining_usd` thực ra trả về **số tiền đã dùng** (`usage.total_cost_usd`), tăng dần chứ không giảm. Đây là bug đặt tên trong code mẫu.

### Exercise 4.4: Cost guard

**Rate limit bảo vệ tốc độ, cost guard bảo vệ tiền:** một client gọi chậm nhưng đều đặn (hoặc prompt rất dài) vẫn đốt hết budget mà không bao giờ chạm rate limit. Ngược lại, rate limit chặn spam và DoS mà cost guard không phản ứng kịp. Cần cả hai.

Cách tôi làm trong Part 6 (`06-lab-complete/app/cost_guard.py`), state trong Redis để mọi worker và instance dùng chung:

```python
def _today_key() -> str:
    return "cost:" + time.strftime("%Y-%m-%d", time.gmtime())     # tự "reset" mỗi ngày UTC

def check_budget() -> None:                                       # gọi TRƯỚC khi gọi LLM
    spent = float(r.get(_today_key()) or 0)
    if spent >= settings.daily_budget_usd:                        # DAILY_BUDGET_USD từ env
        raise HTTPException(status_code=402, detail={...})

def record_cost(input_tokens, output_tokens) -> float:            # gọi SAU khi gọi LLM
    cost = input_tokens/1000*PRICE_INPUT_PER_1K + output_tokens/1000*PRICE_OUTPUT_PER_1K
    pipe = r.pipeline()
    pipe.incrbyfloat(_today_key(), cost)                           # cộng nguyên tử, an toàn khi nhiều worker
    pipe.expire(_today_key(), 2*24*3600)                           # tự xoá key cũ
    pipe.execute()
    return cost
```

- Key theo ngày, nên budget tự reset khi sang ngày mới; muốn budget **$10/tháng/user** như đề CODE_LAB thì đổi key thành `f"budget:{user_id}:{YYYY-MM}"`, ngưỡng `MONTHLY_BUDGET_USD=10`, `EXPIRE` 32 ngày.
- `INCRBYFLOAT` là thao tác nguyên tử của Redis nên 2 worker cộng cùng lúc không mất số.
- Đánh đổi: kiểm tra trước, ghi sau, nên nhiều request song song ngay sát ngưỡng có thể vượt nhẹ (soft limit). Muốn chặt hơn thì ước tính cost trước và "giữ chỗ" bằng `INCRBYFLOAT`, sau đó điều chỉnh theo số thật.
- Script chấm xác nhận: budget 0.000001 USD → **402** ngay ở request đầu (mục Part 6).

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks (`05-scaling-reliability/develop`, chạy trong container Linux)

```text
$ curl -i http://localhost:8000/health
HTTP/1.1 200 OK
{"status":"ok","uptime_seconds":2.6,"version":"1.0.0","environment":"development","timestamp":"2026-11-25T07:21:03.648307+00:00","checks":{"memory":{"status":"ok","used_percent":13.4}}}
$ curl -i http://localhost:8000/ready
HTTP/1.1 200 OK
{"ready":true,"in_flight_requests":1}
```

- **Liveness (`/health`):** "process còn sống không?". Fail thì platform **restart** container. Không được phụ thuộc vào Redis hay DB, nếu không Redis chết sẽ kéo theo restart toàn bộ agent vô ích.
- **Readiness (`/ready`):** "có nhận traffic được không?". Trả 503 khi đang khởi động, đang shutdown hoặc mất dependency; LB **ngừng route** tới instance đó nhưng không restart. Part 6 của tôi: `/ready` ping Redis, mất Redis thì 503 còn `/health` vẫn 200.

### Exercise 5.2: Graceful shutdown

Test trên Linux (Windows không gửi được SIGTERM): gửi request `/ask` rồi `kill -TERM` ngay lập tức.

```text
=== 5.2: gửi request rồi kill -TERM ngay lập tức
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."} [HTTP 200]
process exit code: 0
--- app log ---
INFO:     Shutting down
INFO:     127.0.0.1:34922 - "POST /ask?question=Long%20task HTTP/1.1" 200 OK
INFO:     Waiting for application shutdown.
2026-11-25 07:21:06,801 INFO 🔄 Graceful shutdown initiated...
2026-11-25 07:21:06,801 INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
INFO:     Finished server process [7]
2026-11-25 07:21:06,801 INFO Received signal 15 — uvicorn will handle graceful shutdown
```

Request đang chạy **vẫn hoàn thành với 200** dù SIGTERM đến trước (log "Shutting down" đứng trước dòng `POST /ask ... 200`), rồi process thoát với exit code 0. *Quan sát:* uvicorn tự cài handler SIGTERM khi start server, đè lên `signal.signal(...)` ở cấp module; dòng "Received signal 15" chỉ in ra **sau cùng** vì uvicorn giữ signal lại và gọi handler cũ khi đã tắt xong. Logic dọn dẹp nên đặt trong `lifespan` (phần sau `yield`) chứ không phải trong signal handler riêng.

### Exercise 5.3: Stateless design

`05-scaling-reliability/production/app.py` lưu conversation trong Redis: `save_session()` dùng `SETEX session:<id>` (TTL 1 giờ), `load_session()` dùng `GET`, `append_to_history()` giữ tối đa 20 message. **Vì sao:** khi chạy nhiều instance sau LB, request sau của cùng user có thể rơi vào instance khác; nếu history nằm trong dict RAM của instance A thì instance B không thấy, user "mất trí nhớ". RAM cũng mất sạch khi restart hay deploy. Redis là state dùng chung nên instance nào cũng phục vụ được. *Lưu ý:* code mẫu có fallback sang `_memory_store` khi không kết nối được Redis lúc khởi động; ở production nên fail (readiness 503) thay vì âm thầm chạy chế độ stateful. Part 6 của tôi lưu cả rate limit lẫn cost trong Redis.

### Exercise 5.4: Load balancing (3 instance + Nginx)

```text
$ docker compose -p k5-day14-scaling up -d --build --scale agent=3
NAME                       STATUS                    PORTS
k5-day14-scaling-agent-1   Up 13 seconds (healthy)   8000/tcp
k5-day14-scaling-agent-2   Up 13 seconds (healthy)   8000/tcp
k5-day14-scaling-agent-3   Up 12 seconds (healthy)   8000/tcp
k5-day14-scaling-nginx-1   Up 12 seconds             0.0.0.0:8080->80/tcp
k5-day14-scaling-redis-1   Up 24 seconds (healthy)   6379/tcp

10 request POST /chat qua Nginx, header X-Served-By:
req 1: 172.19.0.5:8000   req 2: 172.19.0.4:8000   req 3: 172.19.0.3:8000
req 4: 172.19.0.5:8000   req 5: 172.19.0.4:8000   req 6: 172.19.0.3:8000
req 7: 172.19.0.5:8000   req 8: 172.19.0.4:8000   req 9: 172.19.0.3:8000
req 10: 172.19.0.5:8000
```

Nginx chia **round-robin** đều cho 3 instance (Docker DNS `agent` trả về 3 IP).

### Exercise 5.5: Test stateless

```text
$ python test_stateless.py
Session ID: 706bf9c0-d3e5-4b34-bc9b-3fb97a51afab
Request 1: [instance-c9c159]  Q: What is Docker?
Request 2: [instance-c4a04d]  Q: Why do we need containers?
Request 3: [instance-714244]  Q: What is Kubernetes?
Request 4: [instance-c9c159]  Q: How does load balancing work?
Request 5: [instance-c4a04d]  Q: What is Redis used for?
Total requests: 5
Instances used: {'instance-c4a04d', 'instance-714244', 'instance-c9c159'}
✅ All requests served despite different instances!
--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```

**Giải thích:** 5 câu của cùng một session do cả 3 instance khác nhau phục vụ, nhưng history đọc lại đủ 10 message (5 user + 5 assistant) theo đúng thứ tự, nên state nằm trong Redis chứ không ở RAM của instance nào.

Thử thêm (đúng kịch bản CODE_LAB): tắt hẳn một instance giữa session.

```text
$ docker stop k5-day14-scaling-agent-1
turn 2 -> HTTP 200 via 172.19.0.3:8000, 172.19.0.5:8000
turn 3 -> HTTP 200 via 172.19.0.4:8000
turn 4 -> HTTP 200 via 172.19.0.5:8000
turn 5 -> HTTP 200 via 172.19.0.4:8000
history count = 10
```

Turn 2: Nginx thử `172.19.0.3` (instance vừa chết), lỗi thì tự chuyển sang `172.19.0.5` (`proxy_next_upstream error timeout http_503`). User vẫn nhận 200 và history vẫn đủ 10 message.

---

## Part 6: Final Project — output `check_production_ready.py`

Chạy bản đầy đủ (có Docker) trong `06-lab-complete/`. Vì máy dùng Windows (Python Windows không gửi được SIGTERM), script được chạy trong một container Linux `python:3.11-slim` có Docker CLI, dùng chung Docker engine của máy (`-v /var/run/docker.sock`, `--network host`), tương đương chạy trong WSL2:

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
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

Một số chi tiết từ output: image **238 MB**; `429 sau 5 request thành công (RATE_LIMIT_PER_MINUTE=5)`; restart agent vẫn bị 429 (state trong Redis); Redis mất kết nối thì `/ready` 503 còn `/health` 200; `402 ở request thứ 1` với budget 0.000001; `17 dòng JSON`; SIGTERM lúc đang có request thì request vẫn 200, exit code 0; `docker stop` → exit code 0 trong 0.8 s.

Kiểm tra thêm bằng `docker compose up` (agent + redis thật): `/health` 200, `/ready` 200, `/ask` không key → 401, có key → 200, body sai `{"invalid":"data"}` → 422 với thông báo rõ ràng, không lộ stack trace.

Tóm tắt thiết kế:
- `config.py`: mọi giá trị đọc từ env (`PORT`, `ENVIRONMENT`, `AGENT_API_KEY`, `REDIS_URL`, `RATE_LIMIT_PER_MINUTE`, `DAILY_BUDGET_USD`, `LOG_LEVEL`).
- `auth.py`: `APIKeyHeader(auto_error=False)` + `secrets.compare_digest`; thiếu, sai hoặc server chưa đặt key đều trả 401.
- `rate_limiter.py`: sliding window bằng Redis sorted set (`ZREMRANGEBYSCORE` → `ZCARD` → `ZADD` + `EXPIRE`) theo từng API key (key được băm SHA-256, không lưu key thô); vượt giới hạn trả 429 kèm `Retry-After`.
- `cost_guard.py`: như Exercise 4.4.
- `main.py`: `JsonFormatter` (mỗi dòng log là một JSON object), middleware log mỗi request, `/health` không phụ thuộc Redis, `/ready` ping Redis, `RedisError` giữa chừng thành 503 thay vì 500 + stack trace, `lifespan` ghi log startup/shutdown (uvicorn lo phần chờ request đang chạy).
- `Dockerfile`: builder `pip install --prefix=/install` → runtime `python:3.11-slim`, user `agent` non-root, `HEALTHCHECK`, `CMD ["sh","-c","exec uvicorn ... --port ${PORT:-8000} --workers 2"]` để uvicorn là PID 1.

---

## AI log

Theo [RULES.md](RULES.md) mục 1. Công cụ: **Claude Code (Claude Opus 5.5)**, chạy trong VS Code với quyền đọc/ghi repo và chạy lệnh trên máy.

| # | Hỏi / giao việc | AI đã làm / trả về | Kiểm tra thế nào |
|---|---|---|---|
| 1 | "Đọc toàn bộ project, liệt kê các task cần làm rồi làm hết" | Đọc README, RULES, CHECKPOINTS, SUBMISSION, RUBRIC, CODE_LAB, INSTRUCTOR_GUIDE và `check_production_ready.py`; lập danh sách task CP1–CP8 | Đối chiếu với bảng điểm trong RUBRIC |
| 2 | Part 6: viết code cho các TODO | Viết `config.py`, `auth.py`, `redis_client.py`, `rate_limiter.py`, `cost_guard.py`, `main.py`, `Dockerfile`, `.dockerignore`, `docker-compose.yml`, `.env.example`, `railway.toml`, `render.yaml` | `check_production_ready.py --static-only` (28/28) rồi bản đầy đủ: **PASS 12/12, 43/43**; thêm `docker compose up` + curl (401/200/422) |
| 3 | Chạy được script chấm đầy đủ trên Windows | WSL Ubuntu không có tích hợp Docker và có Python 3.14 (không có wheel cho `pydantic-core` cũ). AI build một image `python:3.11-slim` + Docker CLI để chạy script trong Linux, dùng chung Docker engine | Output trên; không sửa `check_production_ready.py` |
| 4 | Part 1, 2, 4, 5: chạy ví dụ và trả lời câu hỏi | Chạy từng app, build image, chạy compose, gọi curl và ghi lại output thật; viết câu trả lời | Mọi output trong file là output của lệnh đã chạy; số MB lấy từ `docker image inspect` |
| 5 | Part 7 Bước 1 | Thêm metric `agent_question_length_chars`; chạy load test 4 mức concurrency + 3 mức `MOCK_SLOW_RATE`; eval v1/v2; thêm 2 entry `KNOWLEDGE_BASE` (JWT, HPA); viết `.github/workflows/eval.yml` | Eval cuối: 23/23, `Quality gate ✅ PASS`, 0 regression; số trong `BENCHMARK_REPORT.md` khớp `reports/*.json` |
| 6 | Part 7 Bước 2 | Phát hiện và sửa lỗi `ModuleNotFoundError: uvicorn` trong `production/Dockerfile` (thiếu `PYTHONUSERBASE`); chạy Locust scale 1/3; làm thí nghiệm A–E, đo thời gian firing qua Prometheus API; chụp Grafana/Jaeger/Prometheus bằng Edge headless (Playwright) | `promtool test rules` SUCCESS; số scaling khớp `benchmark-results/*.csv`; ảnh là màn hình thật của stack đang chạy |
| 7 | Viết `MISSION_ANSWERS.md`, `BENCHMARK_REPORT.md`, `DEPLOYMENT.md` | AI soạn nội dung dựa trên output đã chạy | Cần tự đọc lại và hiểu từng đoạn: Lab Coach có thể hỏi bất kỳ dòng code hay con số nào |

| 8 | Deploy cloud (Part 3) | Hướng dẫn từng bước deploy Render Blueprint (tôi tự đăng nhập và bấm deploy trên dashboard; Railway lỗi `Invalid service name` do tên repo dài). Sau khi có URL, AI chạy 4 lệnh tự kiểm, test rate limit, `--public-url`, cập nhật `DEPLOYMENT.md` | Output 200 / 200 / 401 / 200 trong `DEPLOYMENT.md` và `screenshots/test.png`; `dashboard.png` và `running.png` là ảnh tôi tự chụp từ Render |
