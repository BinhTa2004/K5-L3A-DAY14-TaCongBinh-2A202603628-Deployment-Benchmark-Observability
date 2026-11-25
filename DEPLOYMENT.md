# Deployment Information

## Public URL
https://day14-ai-agent-mqab.onrender.com

## Platform
Render (Web Service Docker + Render Key Value / Redis, region Singapore)

## Test Commands

### 1. Health Check (expect 200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "https://day14-ai-agent-mqab.onrender.com/health"
```

### 2. Readiness Check (expect 200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "https://day14-ai-agent-mqab.onrender.com/ready"
```

### 3. API Auth Check — No Key (expect 401)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "https://day14-ai-agent-mqab.onrender.com/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

### 4. API Test — With API Key (expect 200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "https://day14-ai-agent-mqab.onrender.com/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <YOUR_AGENT_API_KEY>" \
  -d '{"question":"Hello"}'
```

## Environment Variables Set
- `PORT` (provided by platform)
- `ENVIRONMENT=production`
- `REDIS_URL` (pointing to managed Redis service)
- `AGENT_API_KEY` (set on platform dashboard, not in code)
- `RATE_LIMIT_PER_MINUTE=10`
- `DAILY_BUDGET_USD=10.0`

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
