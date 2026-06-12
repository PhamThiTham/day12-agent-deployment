#  Delivery Checklist — Day 12 Lab Submission

> **Student Name:** Phạm Thị Thắm  
> **Student ID:** 2A202600789
> **Date:** 12/06/2026

---

##  Submission Requirements

Submit a **GitHub repository** containing:

### 1. Mission Answers (40 points)

Create a file `MISSION_ANSWERS.md` with your answers to all exercises:

# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns tìm được trong `01-localhost-vs-production/develop/app.py`

1. **Hardcode secret** — `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL` chứa password ngay trong code → lộ key khi push lên Git.
2. **Log ra secret** — `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` ghi cả API key ra log.
3. **Dùng `print()` thay vì logging** — không có log level, không structured, khó parse/aggregate.
4. **Không có config management** — `DEBUG = True`, `MAX_TOKENS = 500` cứng trong code, không đọc từ environment.
5. **Không có health check endpoint** — platform không biết khi nào agent chết để restart.
6. **Bind `host="localhost"`** — không chạy được trong container (phải là `0.0.0.0`).
7. **Port cứng `port=8000`** — không đọc `PORT` env mà Railway/Render inject.
8. **`reload=True`** — chế độ dev reload không nên bật ở production (tốn RAM, không an toàn).
9. **Không xử lý graceful shutdown** — khi nhận SIGTERM sẽ ngắt đột ngột, mất request đang xử lý.
10. **Không input validation** — `ask_agent(question: str)` nhận query param thô, không validate độ dài.

### Exercise 1.3: Bảng so sánh Develop vs Production

| Feature | Develop | Production | Tại sao quan trọng? |
|---------|---------|------------|----------------------|
| Config | Hardcode trong code | Đọc từ environment variables (`config.py`) | Đổi cấu hình không cần sửa code; không lộ secret; theo 12-Factor |
| Health check | ❌ Không có | ✅ `GET /health` + `GET /ready` | Platform tự restart container chết; LB chỉ route khi sẵn sàng |
| Logging | `print()` | JSON structured (`logging` + `json.dumps`) | Dễ parse/tìm kiếm trong log aggregator (Datadog, Loki) |
| Shutdown | Đột ngột | Graceful (lifespan + SIGTERM, finish in-flight) | Không mất request đang xử lý khi deploy/scale |
| Host/Port | `localhost:8000` cứng | `0.0.0.0` + `PORT` từ env | Chạy được trong container & cloud (port được inject) |
| Secrets | Trong source code | Trong env / secret manager | Tránh rò rỉ khi push code |
| CORS | Không cấu hình | Chỉ cho phép origins định nghĩa | Hạn chế truy cập trái phép từ domain lạ |

### Checkpoint 1
- [x] Hiểu tại sao hardcode secrets là nguy hiểm
- [x] Biết cách dùng environment variables
- [x] Hiểu vai trò của health check endpoint
- [x] Biết graceful shutdown là gì

---

## Part 2: Docker

### Exercise 2.1: Câu hỏi về Dockerfile (`02-docker/develop/Dockerfile`)

1. **Base image:** `python:3.11` (full distribution, ~1 GB).
2. **Working directory:** `/app` (đặt bằng `WORKDIR /app`).
3. **Tại sao COPY `requirements.txt` trước?** Tận dụng **Docker layer cache** — khi chỉ đổi code mà
   không đổi dependencies, Docker dùng lại layer `pip install` đã cache → build nhanh hơn nhiều.
4. **CMD vs ENTRYPOINT:**
   - `CMD` định nghĩa lệnh mặc định, **có thể bị ghi đè** khi `docker run <image> <other-cmd>`.
   - `ENTRYPOINT` định nghĩa lệnh **cố định** luôn chạy; tham số sau `docker run` được nối vào làm
     đối số. Thường dùng `ENTRYPOINT` cho executable chính + `CMD` cho đối số mặc định.

### Exercise 2.3: So sánh kích thước image (ước lượng)

| Image | Base | Kỹ thuật | Size ước lượng |
|-------|------|----------|----------------|
| Develop | `python:3.11` | Single-stage | ~1.0 GB |
| Production | `python:3.11-slim` | Multi-stage + non-root | ~200–250 MB |
| **Chênh lệch** | | | **~75–80% nhỏ hơn** |

> Lệnh đo thực tế: `docker images | grep my-agent` (cần Docker daemon đang chạy).

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://your-app.railway.app
- Screenshot: [Link to screenshot in repo]

---

## Part 4: API Security

### Exercise 4.1: API Key authentication (`04-api-gateway` & `06-lab-complete/app/main.py`)
- API key được check trong dependency `verify_api_key()` qua header **`X-API-Key`**, so khớp với
  `settings.agent_api_key` (đọc từ env).
- Nếu sai/thiếu key → raise `HTTPException(401)`.
- **Rotate key:** chỉ cần đổi biến môi trường `AGENT_API_KEY` trên dashboard rồi restart — không sửa code.

**Kết quả test (local, TestClient):**
```
GET  /health                         → 200
GET  /ready                          → 503 (chưa ready / lifespan) / 200 khi chạy uvicorn
POST /ask (không key)                → 401 Unauthorized
POST /ask (X-API-Key: test-key-123)  → 200 + answer
```

**Tích hợp LLM thật + test (cập nhật 12/06/2026 — sau khi thanh toán billing):**
Agent dùng client LLM (`utils/llm.py`) với **Google Gemini** — `LLM_PROVIDER=auto` chọn Google → mock,
fallback mock khi lỗi. (OpenAI đã được gỡ bỏ khỏi project theo yêu cầu.)

| Provider | Model | Trạng thái | Kết quả |
|----------|-------|-----------|---------|
| Google Gemini | `gemini-2.5-flash` | ✅ **Hoạt động** | Trả lời thật, `/ask` → `200` |

Ví dụ phản hồi thật từ Gemini:
> *"Docker is a containerization platform that packages applications and their dependencies into
> portable, isolated containers to ensure consistent execution across any computing environment."*

→ Sau khi thanh toán billing, lỗi `429` đã hết; agent trả lời bằng **Gemini thật** (không còn mock).

### Exercise 4.2: JWT authentication (`04-api-gateway/production/auth.py`)
- Flow: `POST /token` (username/password) → server trả JWT (HS256, hết hạn 60 phút) →
  client gửi `Authorization: Bearer <token>` → server verify chữ ký, trích `sub`/`role`.
- Token chứa `sub`, `role`, `iat`, `exp` → **stateless**, không cần truy vấn DB mỗi request.
- Lỗi: `ExpiredSignatureError` → 401; `InvalidTokenError` → 403.

### Exercise 4.3: Rate limiting (`04-api-gateway/production/rate_limiter.py`)
- **Algorithm:** Sliding Window Counter (deque các timestamp, loại bỏ entry cũ hơn 60s).
- **Limit:** User = 10 req/phút; Admin = 100 req/phút.
- **Bypass cho admin:** dùng instance riêng `rate_limiter_admin` với limit cao hơn (phân theo role trong JWT).
- Vượt limit → `429 Too Many Requests` kèm header `Retry-After`, `X-RateLimit-*`.

### Exercise 4.4: Cost guard (`04-api-gateway/production/cost_guard.py`)
- Theo dõi token input/output → quy ra USD theo đơn giá GPT-4o-mini.
- **Per-user budget** vượt → `402 Payment Required`; **global budget** vượt → `503`.
- Cảnh báo khi đạt 80% budget.
- Cách tiếp cận với Redis (gợi ý trong `CODE_LAB.md`):

```python
import redis
from datetime import datetime

r = redis.Redis()

def check_budget(user_id: str, estimated_cost: float) -> bool:
    month_key = datetime.now().strftime("%Y-%m")
    key = f"budget:{user_id}:{month_key}"
    current = float(r.get(key) or 0)
    if current + estimated_cost > 10:   # $10/tháng
        return False
    r.incrbyfloat(key, estimated_cost)
    r.expire(key, 32 * 24 * 3600)       # 32 ngày
    return True
```

### Checkpoint 4
- [x] Implement API key authentication
- [x] Hiểu JWT flow
- [x] Implement rate limiting
- [x] Hiểu cost guard với Redis

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health & readiness checks
```python
@app.get("/health")   # Liveness — luôn 200 nếu process còn sống
def health():
    return {"status": "ok"}

@app.get("/ready")    # Readiness — 503 khi chưa sẵn sàng / dependency lỗi
def ready():
    if not _is_ready:
        raise HTTPException(503, "Not ready")
    return {"ready": True}
```
- **Liveness** trả lời "process còn sống không?" → fail thì platform **restart**.
- **Readiness** trả lời "sẵn sàng nhận traffic chưa?" → fail thì LB **ngừng route** (không restart).

### Exercise 5.2: Graceful shutdown
```python
import signal
def _handle_signal(signum, _frame):
    logger.info(json.dumps({"event": "signal", "signum": signum}))
signal.signal(signal.SIGTERM, _handle_signal)
# + uvicorn timeout_graceful_shutdown=30 và lifespan shutdown đóng connection
```
- Khi orchestrator gửi `SIGTERM`: ngừng nhận request mới, hoàn thành request đang chạy, đóng
  connection rồi thoát → không drop request giữa chừng.

### Exercise 5.3: Stateless design
- **Anti-pattern:** lưu `conversation_history = {}` trong RAM → mỗi instance có bộ nhớ riêng, scale ra
  nhiều instance sẽ không nhất quán.
- **Đúng:** lưu state trong **Redis** (`r.lrange(f"history:{user_id}", 0, -1)`) → mọi instance dùng
  chung, request route tới instance nào cũng thấy đủ history.

### Exercise 5.4: Load balancing (Nginx)
- `nginx.conf` dùng `upstream agent_cluster { server agent:8000; }` + Docker DNS → round-robin các
  instance. `proxy_next_upstream error timeout http_503` để failover khi 1 instance lỗi.
- Scale: `docker compose up --scale agent=3` → Nginx phân tán request, header `X-Served-By` cho thấy
  instance phục vụ.

### Exercise 5.5: Test stateless (`test_stateless.py`)
- Script gửi nhiều request, in `served_by` (instance khác nhau) và xác minh `history` vẫn nguyên vẹn
  dù request đến các instance khác nhau → chứng minh state nằm ở Redis chứ không ở RAM.

### Checkpoint 5
- [x] Implement health và readiness checks
- [x] Implement graceful shutdown
- [x] Hiểu cách refactor thành stateless (Redis)
- [x] Hiểu load balancing với Nginx
- [x] Hiểu cách test stateless design

---

## Tổng kết Mission

| Part | Trạng thái |
|------|-----------|
| Part 1 — Localhost vs Production | ✅ Hoàn thành |
| Part 2 — Docker | ✅ Hoàn thành |
| Part 3 — Cloud Deployment | ✅ Config sẵn sàng (deploy cần tài khoản cá nhân) |
| Part 4 — API Security | ✅ Hoàn thành |
| Part 5 — Scaling & Reliability | ✅ Hoàn thành |
| Part 6 — Final Project | ✅ `check_production_ready.py` → 20/20 (100%) |
