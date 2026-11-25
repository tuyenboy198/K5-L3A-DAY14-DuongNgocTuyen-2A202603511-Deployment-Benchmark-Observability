# Deployment Information

> Agent được deploy là **agent Part 6** trong `06-lab-complete/` (đạt `PASS 12/12`), không phải app mẫu của `03-cloud-deployment/`.

## Public URL

⏳ `https://<điền-domain>.up.railway.app`

## Platform

Railway (config: [`06-lab-complete/railway.toml`](06-lab-complete/railway.toml): builder `DOCKERFILE`, `healthcheckPath = "/health"`). Có sẵn [`06-lab-complete/render.yaml`](06-lab-complete/render.yaml) (web + Key Value Redis) nếu chọn Render.

## Cách deploy (Railway)

```bash
npm i -g @railway/cli
railway login
cd 06-lab-complete
railway init                      # tạo project mới
railway add --database redis      # thêm Redis vào project
railway up                        # build từ Dockerfile, tạo service agent
# Trên dashboard → service agent → Variables (hoặc `railway variables --set`):
#   AGENT_API_KEY = <key bí mật tự sinh, vd: python -c "import secrets;print(secrets.token_urlsafe(24))">
#   REDIS_URL     = ${{Redis.REDIS_URL}}
#   ENVIRONMENT   = production
#   RATE_LIMIT_PER_MINUTE = 10
#   DAILY_BUDGET_USD      = 1.0
railway domain                    # nhận public URL
```

`PORT` do Railway tự cấp; `CMD` của Dockerfile đã nghe `${PORT:-8000}`.

## Test Commands (4 lệnh tự kiểm ở CP6)

```bash
export BASE=https://<điền-domain>.up.railway.app
export AGENT_API_KEY=<key đã đặt trên Railway>   # không commit, không dán vào bài

curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

Kết quả: ⏳ dán output (mong đợi `200 / 200 / 401 / 200`).

Kiểm tra thêm bằng script chấm: `python check_production_ready.py --public-url "$BASE"` (trong `06-lab-complete/`).

## Environment Variables Set

- `PORT` (Railway tự cấp)
- `REDIS_URL` (tham chiếu Redis service: `${{Redis.REDIS_URL}}`)
- `AGENT_API_KEY` (secret, chỉ đặt trên platform)
- `ENVIRONMENT=production`
- `RATE_LIMIT_PER_MINUTE=10`
- `DAILY_BUDGET_USD=1.0`

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png) ⏳
- [Service running](screenshots/running.png) ⏳
- [Test results](screenshots/test.png) ⏳ (ảnh terminal của 4 lệnh trên)
