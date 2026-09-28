# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Minh Hoàng  Mã học viên: 2A202602371

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cụ thể: Khi deploy lên Railway, tôi quên set biến `AGENT_API_KEY` trong dashboard. Nếu `agent_api_key` có giá trị mặc định `"changeme"`, service vẫn khởi động bình thường — health check vẫn xanh, deployment thành công — nhưng bất kỳ ai biết key mặc định đó đều có thể gọi `/ask` tự do, phát sinh chi phí LLM và rò rỉ dữ liệu người dùng khác mà tôi hoàn toàn không hay biết. Ngược lại, khi `agent_api_key` không có mặc định, pydantic-settings ném `ValidationError` ngay trong `Settings.__init__()`, uvicorn không thể start, Railway báo "health check failed" ngay sau deploy — tôi thấy lỗi trong vòng 30 giây và sửa ngay. Đây chính là nguyên tắc Fail Fast: thất bại sớm, to, rõ ràng tốt hơn thành công im lặng mà sai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thực tế thu được từ container khi service khởi động:
> ```
> {"event": "service_started", "level": "info", "timestamp": "2026-09-28T10:35:08.794660+00:00", "service": "day12-agent", "version": "1.0.0"}
> ```
>
> Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
>
> **1. Lọc & tìm kiếm có cấu trúc:** Công cụ như Datadog, Grafana Loki, hoặc Railway Logs đọc trường `"level"` để chỉ hiện log lỗi (`level: "error"`), hoặc lọc theo `"user_id"` để truy vết lịch sử một người dùng cụ thể. `print("đã trả lời xong")` là chuỗi thuần túy — không thể query `WHERE level='error' AND user_id='sv01'`.
>
> **2. Đo lường & cảnh báo tự động:** Từ trường `"cost_usd"` trong mỗi log `ask_completed`, hệ thống monitoring có thể tổng hợp tổng chi phí theo giờ và bắn alert khi vượt ngưỡng. `print()` chỉ cho bạn biết "có chuyện xảy ra" mà không có con số để tính toán.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~420 MB (ước tính: python:3.11-slim + pip cache + build tools) |
| Multi-stage | 184 MB |

> Phần dung lượng chênh lệch (~236 MB) bao gồm các thành phần bị loại bỏ bởi multi-stage build: (1) **Pip cache** — `pip install` tải về nhiều file tạm vào `~/.cache/pip`, chỉ dùng khi cài đặt, không cần lúc chạy; (2) **Compiler và build tools** — một số package Python cần `gcc`, `make`, header files để compile C extension khi cài, nhưng sau khi `.so` đã được tạo ra thì không cần nữa; (3) **Thư mục `/build` stage builder** — toàn bộ working directory của stage đầu không được copy sang stage runtime. Multi-stage chỉ copy thư mục `/install` (các package đã compile sạch) sang stage runtime bằng `COPY --from=builder /install /usr/local`, nên image cuối gọn hơn rất nhiều và giảm attack surface.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi sửa một ký tự trong `app/main.py` và build lại, Docker sử dụng cache cho các layer sau: `FROM python:3.11-slim` (base image), `WORKDIR /build`, `COPY requirements.txt .` — vì `requirements.txt` không đổi. Layer `RUN pip install --no-cache-dir --prefix=/install -r requirements.txt` được **dùng lại từ cache** hoàn toàn — đây là layer tốn thời gian nhất (~30-60 giây). Chỉ từ `COPY app ./app` trở đi mới phải chạy lại (khoảng 0.4 giây theo log build thực tế). Tổng thời gian build lại: ~2 giây.
>
> Nếu đặt `COPY . .` lên **trước** `RUN pip install`: mỗi lần sửa bất kỳ file nào trong project (kể cả comment, markdown), Docker phát hiện `COPY . .` thay đổi → invalidate cache → phải chạy lại `pip install` từ đầu. Thay vì 2 giây, mỗi lần build lại tốn 30-90 giây. Nguyên tắc: đưa những gì **thay đổi ít** lên trước, thứ **thay đổi nhiều** xuống sau để tận dụng Docker layer cache tối đa.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> **Chuỗi sự kiện leo thang đặc quyền:**
> 1. Kẻ tấn công tìm thấy lỗ hổng trong endpoint `/ask` — ví dụ, một lỗ hổng deserialization hoặc path traversal cho phép thực thi lệnh shell tùy ý bên trong container.
> 2. Container chạy bằng `root` (UID 0) — kẻ tấn công có quyền đọc/ghi toàn bộ filesystem container, bao gồm `/etc/shadow`, private keys.
> 3. Docker mặc định share kernel với host. Nếu kernel có lỗ hổng container escape (CVE như `runc` vulnerabilities), kẻ tấn công có thể thoát ra ngoài container và thực thi lệnh trên host với quyền root.
> 4. Trên host với quyền root: đọc secrets của container khác, cài backdoor, chiếm toàn bộ máy chủ.
>
> **`USER appuser` (UID 10001) cắt đứt ở bước 2:** Kể cả khi kẻ tấn công thực thi được lệnh trong container, họ chỉ có quyền của `appuser` — không thể đọc file của root, không thể bind port < 1024, không thể ghi vào `/etc` hay `/usr`. Quan trọng nhất: hầu hết container escape exploit yêu cầu quyền root trong container để hoạt động. Chạy non-root không phải giải pháp hoàn hảo nhưng loại bỏ được phần lớn vector tấn công leo thang đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với đếm theo phút đồng hồ (reset lúc giây :00), một người có thể gửi **20 request trong 2 giây**. Cách đạt được: gửi 10 request vào lúc 10:00:59 (giây cuối của phút trước, counter = 10/10 — vừa đủ) → đúng 10:01:00, counter reset về 0 → gửi thêm 10 request (10:01:00 và 10:01:01). Tổng cộng 20 request trong khoảng thời gian chưa đến 2 giây, gấp đôi hạn mức cho phép, hoàn toàn hợp lệ theo cơ chế đếm phút đồng hồ.
>
> Sliding window Redis ZSET tránh được lỗ hổng này: tại bất kỳ thời điểm nào, `zremrangebyscore(key, 0, now-60)` loại sạch các request cũ hơn 60 giây, rồi `zcard` đếm chính xác số request **thực sự** trong 60 giây gần nhất. Kẻ tấn công không thể lợi dụng thời điểm reset vì không có "thời điểm reset" nào cả.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác biệt cơ bản:** Rate limit kiểm soát **tần suất** (số request/thời gian), không quan tâm mỗi request tốn bao nhiêu. Cost guard kiểm soát **tổng chi phí tích lũy** (USD/tháng), không quan tâm request đến nhanh hay chậm.
>
> **Tình huống rate limit cho qua nhưng cost guard chặn:** User gửi 1 request/ngày (tốc độ chậm, không bao giờ chạm rate limit 10/phút). Nhưng mỗi request hỏi "Hãy viết một cuốn sách 500 trang về AI", tiêu tốn $2.5/lần. Sau 4 ngày đầu tháng, tổng chi phí đạt $10 = `MONTHLY_BUDGET_USD`. Cost guard trả về 402, dù rate limit chưa bao giờ bị kích hoạt.
>
> **Tình huống cost guard cho qua nhưng rate limit chặn:** User mới dùng service lần đầu trong tháng, tổng chi phí = $0 (chưa chạm budget). Nhưng họ script gửi 15 request cùng lúc. Sliding window phát hiện count = 15 >= limit 10 → trả 429. Cost guard vẫn thấy $0 < $10 nên sẵn sàng cho qua, nhưng rate limit đã chặn trước.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> **Chuỗi sự kiện khi gộp /health và /ready, Redis mất kết nối 30 giây:**
>
> 1. **Giây 0:** Redis mất kết nối. `store.ping()` trả về `False`.
> 2. **Giây ~5-10:** Kubernetes/Docker health check gọi `/health` → endpoint thử kết nối Redis → trả về 503. Health check fail.
> 3. **Giây ~15-30:** Sau 3 lần fail liên tiếp (theo `retries: 3, interval: 10s` trong docker-compose.yml), orchestrator đánh dấu cả 3 container là **unhealthy**.
> 4. **Giây ~20-35:** Orchestrator **restart toàn bộ 3 container** (hoặc stop nhận traffic). Trong thời gian restart, service hoàn toàn không xử lý được request nào — downtime hoàn toàn.
> 5. **Giây ~35-60:** Container mới start, nhưng Redis vẫn chưa kết nối lại → health check vẫn fail → lại restart → restart loop vô tận trong suốt 30 giây Redis offline.
>
> **Khi tách riêng:** `/health` không gọi Redis → 3 container luôn "alive" và tiếp nhận traffic trong suốt 30 giây. Load balancer chỉ dùng `/ready` để quyết định route traffic → khi Redis offline, container báo `503 /ready` nên không nhận request mới, nhưng container không bị kill/restart. Khi Redis phục hồi, `/ready` trả 200, traffic tự động quay lại — zero downtime.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> **Khi dùng Redis (Stateless):** Mỗi lần gọi `/ask` với `X-User-Id: sv01`, dù request đến bất kỳ instance nào trong 3 container, `history_length` tăng đều đặn: 0 → 1 → 2 → 3 → ... vì tất cả đều đọc/ghi vào cùng một Redis key `history:sv01`. Số liệu nhất quán hoàn toàn.
>
> **Khi dùng dict Python (Stateful in-process):** Mỗi container có dict riêng trong bộ nhớ của mình — `instance_1.history = {}`, `instance_2.history = {}`, `instance_3.history = {}`. Load balancer round-robin phân phối request:
> - Request 1 → instance_1: history_length = 1
> - Request 2 → instance_2: history_length = **1** (instance_2 không thấy request 1!)
> - Request 3 → instance_3: history_length = **1** (lại về 1!)
> - Request 4 → instance_1: history_length = 2
>
> Con số nhảy loạn xạ, không tăng đều, không thể dự đoán được. Agent trả lời không nhớ context vì mỗi lần hỏi có thể rơi vào instance khác với history rỗng. Đây chính là lý do kiến trúc Stateless bắt buộc phải lưu state ra external storage (Redis) thay vì in-process memory khi scale.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải:** Khi thực hiện phương án dự phòng LOCAL_FALLBACK với `docker compose up -d`, lần đầu chạy tôi để `REDIS_URL=fake://localhost:6379/0` trong `.env`. Service khởi động thành công nhưng `/ready` trả về `{"status": "ready", "redis": true}` dù không có Redis thật — vì `fake://` dùng fakeredis in-memory. Khi chạy `pytest tests/test_cp5.py -v`, test `test_ready_ket_noi_duoc_redis` pass nhưng đây là "pass giả" vì Redis thật không được kiểm tra.
>
> **Tìm nguyên nhân:** Đọc lại code `store.py`, hàm `get_redis_client()` phân biệt `fake://` scheme để khởi tạo `fakeredis.FakeRedis()` thay vì kết nối Redis thật. `store.ping()` gọi `r.ping()` trên fakeredis luôn trả `True` kể cả khi không có Redis service.
>
> **Cách sửa:** Đổi `REDIS_URL=redis://localhost:6379/0` trong `.env` và đảm bảo `docker compose up -d redis` đã chạy trước. Chạy lại `docker compose ps` xác nhận redis container status "healthy", sau đó `/ready` vẫn trả 200 nhưng lần này là kết nối Redis thật. Bài học: môi trường test local nên gần giống production nhất có thể — dùng Redis thật thay vì fakeredis để tránh false positive.
