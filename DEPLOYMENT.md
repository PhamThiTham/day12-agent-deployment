# Deployment Information

> **Học viên:** Phạm Thị Thắm
> **Mã số học viên:** 2A202600789

---

## Public URL

> ⏳ Điền URL sau khi deploy (xem hướng dẫn cuối file).

```
https://<your-agent>.up.railway.app
```

## Platform

- [ ] Railway
- [ ] Render
- [ ] Cloud Run

## Environment Variables đã set

| Biến | Mô tả | Ví dụ |
|------|-------|-------|
| `PORT` | Port cloud inject | `8000` (tự động) |
| `ENVIRONMENT` | Môi trường | `production` |
| `AGENT_API_KEY` | API key bảo vệ `/ask` | *(secret mạnh, KHÔNG dùng `dev-key-change-me`)* |
| `JWT_SECRET` | Secret ký JWT | *(secret mạnh)* |
| `LLM_PROVIDER` | Chọn LLM: `auto`/`openai`/`google`/`mock` | `auto` |
| `OPENAI_API_KEY` | Key OpenAI (nếu dùng) | `sk-...` |
| `GOOGLE_API_KEY` | Key Google Gemini (nếu dùng) | `AIza...` |
| `LLM_MODEL` | Model | `gpt-4o-mini` / `gemini-2.5-flash` |
| `RATE_LIMIT_PER_MINUTE` | Giới hạn request | `20` |
| `DAILY_BUDGET_USD` | Ngân sách/ngày | `5.0` |
| `REDIS_URL` | (tuỳ chọn) Redis cho state | `redis://...` |

---

## Test Commands

### Health Check
```bash
curl https://<your-agent>.up.railway.app/health
# Expected: {"status": "ok", ...}
```

### Readiness Check
```bash
curl https://<your-agent>.up.railway.app/ready
# Expected: {"ready": true}
```

### Auth bắt buộc (không key → 401)
```bash
curl -X POST https://<your-agent>.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: 401 Unauthorized
```

### API Test (có key → 200)
```bash
curl -X POST https://<your-agent>.up.railway.app/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is deployment?"}'
# Expected: 200 + {"question": ..., "answer": ..., "model": ..., "timestamp": ...}
```

### Rate limiting (vượt limit → 429)
```bash
for i in $(seq 1 25); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST https://<your-agent>.up.railway.app/ask \
    -H "X-API-Key: YOUR_KEY" -H "Content-Type: application/json" \
    -d '{"question": "test"}'
done
# Expected: một số request trả 429 sau khi vượt 20 req/phút
```

---

## Screenshots

- [ ] `screenshots/dashboard.png` — Deployment dashboard
- [ ] `screenshots/running.png` — Service running
- [ ] `screenshots/test.png` — Test results

---

## Hướng dẫn deploy

### Cách A — Railway (CLI)
```bash
cd 06-lab-complete
npm i -g @railway/cli
railway login
railway init
railway variables set ENVIRONMENT=production
railway variables set AGENT_API_KEY=<secret-mạnh>
railway variables set JWT_SECRET=<secret-mạnh>
railway up
railway domain        # lấy public URL
```

### Cách B — Render (Blueprint)
1. Push repo lên GitHub (public hoặc cấp quyền cho giảng viên).
2. Render Dashboard → **New → Blueprint** → connect repo (Render đọc `06-lab-complete/render.yaml`).
3. Set secret `AGENT_API_KEY` (và `JWT_SECRET` nếu cần) trong dashboard.
4. Deploy → nhận public URL.

> ⚠️ KHÔNG commit `.env` / `.env.local`. Chỉ commit `.env.example`.
