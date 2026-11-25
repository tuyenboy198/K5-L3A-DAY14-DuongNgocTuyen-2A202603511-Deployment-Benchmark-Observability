# Deployment Information

> Agent được deploy là **agent Part 6** trong `06-lab-complete/` (đạt `PASS 12/12`), không phải app mẫu của `03-cloud-deployment/`.

## Public URL

**https://day14-agent-pm39.onrender.com**

> Gói Free của Render tự ngủ sau khoảng 15 phút không có traffic. Request đầu tiên sau đó có thể mất 30–60 giây để service khởi động lại, các request sau trả lời bình thường.

## Platform

**Render** (Blueprint), dùng file [`06-lab-complete/render.yaml`](06-lab-complete/render.yaml):

| Resource | Loại | Ghi chú |
|---|---|---|
| `day14-agent` | Web Service, runtime Docker, plan Free, region Singapore | Build từ `06-lab-complete/Dockerfile` (multi-stage), `healthCheckPath: /health` |
| `day14-redis` | Key Value (Redis), plan Free | `REDIS_URL` của agent được Render tự nối qua `fromService ... property: connectionString` |

Trạng thái: **Live**, commit `56a793e` (deploy mất 58.9 s).

## Cách deploy

1. Push repo lên GitHub.
2. Render Dashboard → **New → Blueprint** → chọn repo → **Blueprint Path** = `06-lab-complete/render.yaml` (file không nằm ở gốc repo).
3. Render đọc file và tạo `day14-redis` + `day14-agent`; hỏi giá trị `AGENT_API_KEY` (`sync: false`) → nhập key bí mật (không commit).
4. **Deploy Blueprint**, chờ service chuyển sang **Live**, lấy URL ở đầu trang service.

(Có sẵn [`06-lab-complete/railway.toml`](06-lab-complete/railway.toml) nếu muốn deploy Railway. Railway từ chối tạo service từ GitHub vì tên repo quá dài làm tên service, nên chọn Render.)

## Test Commands (4 lệnh tự kiểm ở CP6)

```bash
export BASE=https://day14-agent-pm39.onrender.com
export AGENT_API_KEY=<key đã đặt trên Render>   # không commit, không dán vào bài

curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

Kết quả thật (2026-11-25, 09:45 UTC):

```text
200
200
401
200
```

### Health Check

```bash
curl https://day14-agent-pm39.onrender.com/health
# {"status":"ok","environment":"production"}
curl https://day14-agent-pm39.onrender.com/ready
# {"status":"ready"}        ← Redis (Render Key Value) kết nối được
```

### API Test (with authentication)

```bash
curl -X POST https://day14-agent-pm39.onrender.com/ask \
  -H "X-API-Key: $AGENT_API_KEY" -H "Content-Type: application/json" \
  -d '{"question": "What is deployment?"}'
# {"question":"What is deployment?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.","usage":{"input_tokens":6,"output_tokens":32,"cost_usd":2.01e-05}}
```

### Rate limiting trên cloud (state trong Redis)

```text
$ for i in $(seq 1 12); do curl -s -o /dev/null -w "%{http_code} " -X POST "$BASE/ask" -H "X-API-Key: $AGENT_API_KEY" -H "Content-Type: application/json" -d "{\"question\":\"rate test $i\"}"; done
200 200 200 200 200 200 200 200 429 429 429 429
```

(Trong cùng phút đó đã có 2 request có key từ các lệnh ở trên, nên tổng cộng là đúng 10 request/phút trước khi bị 429. Response 429 có header `retry-after`.)

### Script chấm

```text
$ python check_production_ready.py --public-url https://day14-agent-pm39.onrender.com
     ✅ Public URL: GET https://day14-agent-pm39.onrender.com/health → 200
```

## Environment Variables Set

Đặt trên Render (khai báo trong `render.yaml`, secret không ghi giá trị vào code):

- `PORT`: Render tự cấp; `CMD` của Dockerfile nghe theo `${PORT:-8000}`
- `REDIS_URL`: tự nối từ Key Value `day14-redis` (`fromService`)
- `AGENT_API_KEY`: secret, `sync: false`, nhập tay trên dashboard
- `ENVIRONMENT=production`
- `RATE_LIMIT_PER_MINUTE=10`
- `DAILY_BUDGET_USD=1.0`

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png): Blueprint `day14-agent` đã sync, tạo Key Value `day14-redis` + web service `day14-agent`
- [Service running](screenshots/running.png): web service `day14-agent` trạng thái **Live**, URL public
- [Test results](screenshots/test.png): 4 lệnh tự kiểm (200 / 200 / 401 / 200) và vòng lặp rate limit (429)
