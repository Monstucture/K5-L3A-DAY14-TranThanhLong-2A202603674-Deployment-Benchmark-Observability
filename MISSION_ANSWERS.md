#  Delivery Checklist — Day 14 Lab Submission

> **Student Name:** __________Trần Thanh Long_______________  
> **Student ID:** ___________2A202603674______________  
> **Date:** ____________25/11/2026_____________

---

##  Submission Requirements

Submit a **GitHub repository** containing:

### 1. Mission Answers (40 points)

Create a file `MISSION_ANSWERS.md` with your answers to all exercises:

# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

| # | Anti-pattern | Dòng | Vì sao nguy hiểm |
|---|---|---|---|
| 1 | API key hardcode: `OPENAI_API_KEY = "sk-..."` | 17 | Push lên GitHub là key bị lộ ngay. Key nên được đọc từ biến môi trường. |
| 2 | Mật khẩu database hardcode trong `DATABASE_URL` | 18 | Cùng rủi ro lộ thông tin, và chỉ chạy được trên một máy. |
| 3 | Secret bị in ra log: `print(f"... {OPENAI_API_KEY}")` | 34 | Ai đọc được log là có key. |
| 4 | Bật debug cứng: `DEBUG = True` | 21 | Debug output trên production lộ chi tiết nội bộ, và không đổi được theo môi trường. |
| 5 | Không có quản lý config: `MAX_TOKENS = 500` hardcode | 22 | Đổi một giá trị phải sửa code và deploy lại. |
| 6 | Host và port cố định: `host="localhost"`, `port=8000` | 51–52 | `localhost` chỉ nhận kết nối từ chính máy đó, nên không chạy được trong container. Port cố định bỏ qua biến `PORT` mà Railway và Render cấp. |
| 7 | Bật auto-reload: `reload=True` | 53 | Reload chỉ dành cho lúc phát triển. Nó tốn tài nguyên và có thể khởi động lại server bất ngờ. |
| 8 | Dùng `print()` thay vì structured logging | 33, 34, 38 | Không có log level, không có timestamp, và không có cấu trúc JSON để công cụ log đọc được. |
| 9 | Không có health check endpoint | 42–43 | Platform không biết app còn sống hay không, nên không restart được instance bị treo. |
| 10 | Không có graceful shutdown | (không có handler) | Khi nhận SIGTERM, process dừng ngay, có thể làm đứt các request đang chạy. |

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Why Important? |
|---|---|---|---|
| Config | Giá trị hardcode trong `app.py` (dòng 17–22) | Đọc từ biến môi trường trong `config.py` (`HOST`, `PORT`, `DEBUG`, `OPENAI_API_KEY`, `AGENT_API_KEY`…). `Settings.validate()` báo lỗi trên production nếu thiếu `AGENT_API_KEY`. | Cùng một image có thể chạy ở development, staging và production với giá trị khác nhau, và secret không nằm trong code. |
| Health check | Không có | `GET /health` (liveness: "process còn sống không?") và `GET /ready` (readiness: "đã nhận traffic được chưa?", trả 503 cho đến khi khởi động xong) | Platform dùng hai endpoint này để restart instance đã chết và tránh gửi traffic vào instance đang khởi động. |
| Logging | `print()`, secret nằm trong output | `logging` với format kiểu JSON; app log các sự kiện (`agent_request`, `agent_response`) và không log key | Công cụ log có thể tìm kiếm và cảnh báo theo field có cấu trúc. Secret trong log là lộ thông tin. |
| Shutdown | Đột ngột: process dừng, không dọn dẹp | Handler `lifespan` log lúc tắt, đặt `is_ready = False`, và chờ một chút cho request đang chạy; có đăng ký handler SIGTERM | Khi tắt instance lúc deploy, không nên làm đứt các request đang xử lý. |

**Những giới hạn tôi nhận thấy ở bản advanced (để thảo luận):**

- Handler SIGTERM (`handle_sigterm`) chỉ ghi log. Uvicorn mới là nơi thực hiện việc tắt, nên handler không thêm logic dọn dẹp nào.
- Thời gian chờ khi tắt là `time.sleep(0.1)` cố định, không phải chờ thật các request đang chạy.
- Phần khởi động có `time.sleep(0.1)` để giả lập việc load, đây là phần giữ chỗ.

Nên nêu các điểm này khi trả lời phần thảo luận, vì tên gọi "graceful shutdown" trong code mạnh hơn những gì code thực sự làm.

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)
1. **Base image:** `python:3.11` bản Python đầy đủ (khoảng 1 GB), có sẵn nhiều công cụ build không cần khi chạy.
2. **Working directory:** `/app` (đặt bằng `WORKDIR /app`). Các lệnh `COPY`, `RUN`, `CMD` sau đó đều chạy trong thư mục này.
3. **Tại sao COPY requirements.txt trước:** Docker cache theo từng layer. Copy `requirements.txt` rồi `pip install` trước khi copy code, nên khi chỉ sửa `app.py` thì layer cài dependencies được dùng lại từ cache, build lại nhanh hơn nhiều.
4. **CMD vs ENTRYPOINT:** `CMD` là lệnh mặc định khi container chạy, có thể bị ghi đè bằng tham số của `docker run`. `ENTRYPOINT` là chương trình cố định luôn chạy; nếu có cả hai thì `CMD` đóng vai trò tham số mặc định cho `ENTRYPOINT`. File này chỉ dùng `CMD ["python", "app.py"]`.

### Exercise 2.2: Build và run (develop)
Build từ gốc repo: `docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .`

Kết quả gọi container:
```text
GET  /health                         → {"status":"ok","uptime_seconds":6.3,"container":true}
POST /ask?question=What is Docker?   → {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Multi-stage build và so sánh kích thước
- **Stage 1 (builder):** dùng `python:3.11-slim`, cài `gcc`, `libpq-dev` và toàn bộ dependencies bằng `pip install --user`. Stage này chỉ để build, không được deploy.
- **Stage 2 (runtime):** bắt đầu lại từ `python:3.11-slim` sạch, chỉ copy thư mục packages đã cài (`/root/.local`) và code (`main.py`, `utils/mock_llm.py`), chạy bằng user non-root `appuser`, có `HEALTHCHECK`.
- **Tại sao image nhỏ hơn:** base `slim` nhỏ hơn nhiều so với bản đầy đủ, và các công cụ build (`gcc`, apt cache, pip cache) nằm lại ở stage 1, không vào image cuối.

Kích thước đo bằng `docker images` (ngày 25/11/2026):

| Image | Kích thước |
|---|---|
| `my-agent:develop` | 1.67 GB |
| `my-agent:advanced` | 240 MB |
| **Chênh lệch** | **≈ 85.6% nhỏ hơn** (nhỏ hơn khoảng 7 lần) |

Cách tính: (1668 − 240) / 1668 × 100 ≈ 85.6%. Image production đạt yêu cầu < 500 MB.

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://your-app.railway.app
- Screenshot: [Link to screenshot in repo]

## Part 4: API Security

### Exercise 4.1-4.3: Test results
[Paste your test outputs]

### Exercise 4.4: Cost guard implementation
[Explain your approach]

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
[Your explanations and test results]
```

---

### 2. Full Source Code - Lab 06 Complete (60 points)

Your final production-ready agent, built from the skeleton in `06-lab-complete/` (xem mục **Hợp đồng** trong [06-lab-complete/README.md](06-lab-complete/README.md)):

```
your-repo/
├── app/
│   ├── main.py              # Main application
│   ├── config.py            # Configuration
│   ├── auth.py              # Authentication
│   ├── rate_limiter.py      # Rate limiting (Redis)
│   ├── cost_guard.py        # Cost protection (Redis)
│   └── redis_client.py      # Redis connection + health
├── utils/
│   └── mock_llm.py          # Mock LLM (provided)
├── check_production_ready.py  # Grading script (provided, do not modify)
├── Dockerfile               # Multi-stage build
├── docker-compose.yml       # Full stack
├── requirements.txt         # Dependencies
├── .env.example             # Environment template
├── .dockerignore            # Docker ignore
├── railway.toml             # Railway config (or render.yaml)
└── README.md                # Setup instructions
```

**Requirements:**
-  **`python check_production_ready.py` báo `PASS 12/12`** (chạy đầy đủ, có Docker) — dán phần "📋 Checklist Deliverable" của output vào `MISSION_ANSWERS.md`
-  All code runs without errors
-  Multi-stage Dockerfile (image < 500 MB)
-  API key authentication
-  Rate limiting (10 req/min)
-  Cost guard ($10/month)
-  Health + readiness checks
-  Graceful shutdown
-  Stateless design (Redis)
-  No hardcoded secrets

---

### 3. Service Domain Link

Create a file `DEPLOYMENT.md` with your deployed service information:

````markdown
# Deployment Information

## Public URL
https://your-agent.railway.app

## Platform
Railway / Render / Cloud Run

## Test Commands

### Health Check
```bash
curl https://your-agent.railway.app/health
# Expected: {"status": "ok"}
```

### API Test (with authentication)
```bash
curl -X POST https://your-agent.railway.app/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

## Environment Variables Set
- PORT
- REDIS_URL
- AGENT_API_KEY
- LOG_LEVEL

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
````

---

### 4. Benchmark & Observability Report — Part 7 (20 points)

Copy [06-lab-complete/benchmark-observability/BENCHMARK_REPORT_TEMPLATE.md](06-lab-complete/benchmark-observability/BENCHMARK_REPORT_TEMPLATE.md) thành `BENCHMARK_REPORT.md` và điền đầy đủ, kèm các file bằng chứng:

```
your-repo/
├── BENCHMARK_REPORT.md
├── reports/
│   ├── load_v2_c1.json ... load_v2_c100.json   # load_test.py --out
│   ├── eval_v1.json
│   ├── eval_v2.json
│   └── eval_v2_fixed.json                      # sau khi sửa knowledge base
├── benchmark-results/
│   ├── scale_1_stats.csv                       # Locust headless
│   └── scale_3_stats.csv
└── screenshots/
    ├── grafana-dashboard.png
    ├── jaeger-slow-trace.png
    ├── jaeger-error-trace.png
    └── prometheus-alert-firing.png
```

**Requirements:**
-  Load test ở 4 mức concurrency, có p50/p95/p99 và đối chiếu SLO
-  Eval v1 vs v2, sửa knowledge base để v2 đạt 100% không regression
-  Dashboard Grafana có tải, 1 trace chậm + 1 trace lỗi trong Jaeger
-  Ít nhất 2 alert firing, ghi lại thời gian phát hiện
-  Số liệu trong report khớp với file JSON/CSV

> **Cách tính điểm:** tổng thô 120 (40 + 60 + 20), quy đổi về thang 100.

---

##  Pre-Submission Checklist

- [ ] Repository is public (or instructor has access)
- [ ] `MISSION_ANSWERS.md` completed with all exercises
- [ ] `DEPLOYMENT.md` has working public URL
- [ ] All source code in `app/` directory
- [ ] `README.md` has clear setup instructions
- [ ] No `.env` file committed (only `.env.example`)
- [ ] No hardcoded secrets in code
- [ ] Public URL is accessible and working
- [ ] Screenshots included in `screenshots/` folder
- [ ] Repository has clear commit history
- [ ] `python check_production_ready.py` → `PASS 12/12`
- [ ] `BENCHMARK_REPORT.md` điền đủ, có `reports/`, `benchmark-results/` và screenshot Part 7

---

##  Self-Test

Before submitting, verify your deployment:

```bash
# 1. Health check
curl https://your-app.railway.app/health

# 2. Authentication required
curl https://your-app.railway.app/ask
# Should return 401

# 3. With API key works
curl -H "X-API-Key: YOUR_KEY" https://your-app.railway.app/ask \
  -X POST -d '{"user_id":"test","question":"Hello"}'
# Should return 200

# 4. Rate limiting
for i in {1..15}; do 
  curl -H "X-API-Key: YOUR_KEY" https://your-app.railway.app/ask \
    -X POST -d '{"user_id":"test","question":"test"}'; 
done
# Should eventually return 429

# 5. Part 7 — eval quality gate (agent v2 đang chạy ở local)
cd 06-lab-complete/benchmark-observability/develop
python eval/run_eval.py --min-accuracy 1.0
# Should print: Quality gate ... ✅ PASS
```

---

##  Submission

**Submit your GitHub repository URL:**

```
https://github.com/your-username/day14-agent-deployment
```

---

##  Quick Tips

1.  Test your public URL from a different device
2.  Make sure repository is public or instructor has access
3.  Include screenshots of working deployment
4.  Write clear commit messages
5.  Test all commands in DEPLOYMENT.md work
6.  No secrets in code or commit history

---

##  Need Help?

- Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- Review [CODE_LAB.md](CODE_LAB.md)
- Ask in office hours
- Post in discussion forum

---

**Good luck! **
