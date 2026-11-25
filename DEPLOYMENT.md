# Deployment Information

## Public URL
https://k5-l3a-day14-nguyenthetruong-2a202603393.onrender.com

## Platform
Render (Web Service — Docker runtime, Root Directory: `06-lab-complete`, kèm Render Key Value / Redis miễn phí)

## Test Commands

### Health & Readiness Check
```bash
BASE="https://k5-l3a-day14-nguyenthetruong-2a202603393.onrender.com"

curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"   # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"    # 200
```

### API Test (Authentication & Inference)
```bash
# Không có API key → 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" \
  -H 'Content-Type: application/json' \
  -d '{"question":"Hello"}'

# Có API key hợp lệ → 200
curl -X POST "$BASE/ask" \
  -H "X-API-Key: <YOUR_AGENT_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

### Kết quả kiểm tra thực tế
```text
GET /health -> 200
GET /ready  -> 200
POST /ask (no key)   -> 401
POST /ask (with key) -> 200
```

## Environment Variables Set (trên Render Dashboard, không commit secret)
- `PORT` (Render tự cấp)
- `REDIS_URL` (Internal URL của Render Key Value / Redis)
- `AGENT_API_KEY` (bí mật)
- `ENVIRONMENT=production`
- `RATE_LIMIT_PER_MINUTE=10`
- `DAILY_BUDGET_USD=1.0`

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
