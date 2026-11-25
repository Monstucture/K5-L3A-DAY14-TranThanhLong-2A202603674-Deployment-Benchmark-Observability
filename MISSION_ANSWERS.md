# Bài làm — Day 14

## Part 1 — Localhost vs Production

### Exercise 1.1 — Anti-patterns trong `01-localhost-vs-production/develop/app.py`

Tôi tìm được 10 vấn đề. Số dòng tham chiếu đến file đó.

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

### Exercise 1.3 — So sánh basic và advanced (`production/app.py` và `production/config.py`)

| Tính năng | Basic (`develop/`) | Advanced (`production/`) | Vì sao quan trọng |
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
