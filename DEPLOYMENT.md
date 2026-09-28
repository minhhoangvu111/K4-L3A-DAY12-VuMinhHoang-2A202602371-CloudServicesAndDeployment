# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Vũ Minh Hoàng |
| Mã học viên | 2A202602371 |
| Repo | [K4-L3A-DAY12-VuMinhHoang-2A202602371-CloudServicesAndDeployment](https://github.com/minhhoangvu111/K4-L3A-DAY12-VuMinhHoang-2A202602371-CloudServicesAndDeployment) |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | http://localhost:8000 |
| Platform | Local Docker Compose (Phương án dự phòng — LOCAL_FALLBACK=true) |
| Ngày deploy | 2026-09-28 |

> **Ghi chú:** Sử dụng phương án dự phòng LOCAL_FALLBACK do không thể đăng ký tài khoản Railway/Render trong thời gian làm bài.
> Stack chạy đầy đủ trên máy local với `docker compose up -d`. Xem ảnh minh chứng trong `screenshots/`.

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán (8000 khi chạy local) |
| `AGENT_API_KEY` | ✅ | đặt trong file .env local, không nằm trong repo |
| `REDIS_URL` | ✅ | `redis://redis:6379/0` — Redis service trong Docker Compose |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i http://localhost:8000/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i http://localhost:8000/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# docker compose ps (2026-09-28)
NAME                                                                         IMAGE                                                                   COMMAND               SERVICE   STATUS                        PORTS
k4-l3a-day12-vuminhhoang-2a202602371-cloudservicesanddeployment-agent-1    k4-l3a-...agent   "sh -c 'uvicorn app…"   agent     Up (healthy)   0.0.0.0:8000->8000/tcp
k4-l3a-day12-vuminhhoang-2a202602371-cloudservicesanddeployment-redis-1    redis:7-alpine    "docker-entrypoint.s…"   redis     Up 2 hours (healthy)   0.0.0.0:6379->6379/tcp

# GET /health → 200
{"status": "ok", "service": "day12-agent", "version": "1.0.0"}

# GET /ready → 200
{"status": "ready", "redis": true}

# POST /ask (no API key) → 401
{"detail": "Missing or invalid API key"}

# pytest tests/test_cp5.py -v
8 passed, 5 skipped in 3.79s  ✅
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — terminal chạy `docker compose ps`
- `screenshots/health.png` — kết quả gọi `/health` từ curl

---

## Phương Án Dự Phòng

Sử dụng phương án dự phòng LOCAL_FALLBACK do hạn chế về đăng ký tài khoản Cloud trong thời gian thực hiện bài lab.

Lý do: Không thể hoàn thành đăng ký và cấu hình tài khoản Railway/Render trong khung thời gian 30 phút của phase CP5. Stack đã chạy đầy đủ và ổn định trên máy local với Docker Compose, bao gồm cả Redis và ứng dụng agent, với tất cả tính năng hoạt động bình thường (auth, rate limiting, cost guard, graceful shutdown).
