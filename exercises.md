# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Vũ Huy  Mã học viên: 2A202602662

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tôi đã thử thật: chạy `docker run --rm agent:multi` mà không truyền `AGENT_API_KEY`.
> Lần đầu app vẫn khởi động, `/health` trả 200, chỉ đến khi gọi `/ask` mới lỗi 500
> `ValidationError ... agent_api_key Field required` — vì `get_settings()` chỉ được gọi
> lúc có request. Nếu deploy như vậy lên Render, health check sẽ báo xanh, bản mới được
> đưa vào phục vụ và mọi user đều nhận 500. Tôi sửa bằng cách gọi `get_settings()` trong
> `lifespan`: giờ container chết ngay với `Application startup failed` (exit code 3), deploy
> bị đánh fail và Render giữ nguyên bản cũ đang chạy.
> Nếu để mặc định `"changeme"` thì còn tệ hơn: app chạy "bình thường" với một khóa mà ai
> đọc repo cũng biết, người lạ gọi `/ask` bằng khóa đó và tôi chỉ phát hiện khi thấy hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật lấy từ runtime log trên Render:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:11:24.571703+00:00", "user_id": "sv-demo2", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. Lọc và cộng theo trường: lọc `event = ask_completed` rồi group theo `user_id`, cộng
>    `cost_usd` để biết user nào tiêu nhiều tiền nhất hôm nay.
> 2. Đếm/cảnh báo theo thời gian: đếm số dòng `level = error` trong 5 phút qua dựa vào
>    `timestamp`, hoặc cảnh báo khi `tokens_in` vượt ngưỡng. Tôi cũng đã dùng
>    `render logs --text ask_completed` để tìm đúng dòng này.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật trên máy tôi (`docker images`): bản 1 stage 1.73 GB, bản multi-stage 271 MB
> (giảm khoảng 6 lần). Phần chênh lệch gồm:
> - Base image `python:3.11` bản đầy đủ (Debian đầy đủ + gcc, build-essential, header dev,
>   git, curl...) so với `python:3.11-slim` chỉ đủ để chạy Python.
> - Cache của pip: bản đầu `pip install` không có `--no-cache-dir` nên file tải về nằm luôn
>   trong layer.
> - `COPY . .` mang theo mọi thứ trong thư mục (tests, markdown, nginx...), còn bản mới chỉ
>   copy `app/` và `utils/`. Ở bản multi-stage, stage `builder` cài thư viện vào `/install`
>   rồi bị vứt đi; stage cuối chỉ nhận thư viện đã cài.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một dòng comment vào `app/main.py` rồi build lại với `--progress=plain`:
> - Dockerfile của tôi: các layer `WORKDIR`, `COPY requirements.txt`, `RUN pip install`,
>   `useradd`, `COPY --from=builder` đều `CACHED`; chỉ `COPY app ./app` và `COPY utils ./utils`
>   chạy lại, mỗi bước 0.1s, cả lần build 0.4s.
> - Dockerfile 1 stage ban đầu (có `COPY . .` đứng trước `RUN pip install`): layer `COPY . .`
>   đổi nên mọi layer phía sau mất cache, `RUN pip install -r requirements.txt` chạy lại
>   mất 60.7s. Sửa một dấu phẩy cũng phải cài lại toàn bộ thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy root: (1) code Python có lỗ hổng, ví dụ thư viện bị RCE
> hoặc tôi lỡ đưa input người dùng vào `eval`/`subprocess` → (2) kẻ tấn công chạy lệnh tùy ý
> trong container với UID 0 → (3) root trong container đọc/ghi được mọi file, cài công cụ, và
> nếu container có mount nhạy cảm (`/var/run/docker.sock`, thư mục host), chạy `--privileged`
> hoặc kernel có lỗ hổng thì thoát ra được và thành root trên host → (4) chiếm máy host và
> các container khác.
> `USER appuser` (UID 10001) cắt chuỗi ở bước 2–3: lệnh của kẻ tấn công chỉ có quyền của một
> user thường, không ghi được file hệ thống, không cài gói, và UID 10001 trên host cũng không
> có quyền gì, nên việc leo thang và thoát khỏi container khó hơn rất nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong khoảng 2 giây. Với đếm theo phút đồng hồ và hạn mức 10/phút:
> gửi 10 request lúc 10:00:59 (thuộc phút 10:00, còn đủ quota), counter reset lúc 10:01:00,
> rồi gửi tiếp 10 request lúc 10:01:01 (quota của phút mới) — 20 request trong 2 giây mà
> không bị chặn. Với sliding window 60 giây của tôi, request thứ 11 lúc 10:01:01 vẫn thấy 10
> request trong 60 giây gần nhất nên bị 429. Trên bản deploy thật tôi gọi 15 lần liên tiếp
> và nhận `200` × 10 rồi `429` × 5.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số request trong một khoảng thời gian** (chống spam, trả 429, tự hết
> sau 60 giây). Cost guard giới hạn **tổng tiền trong tháng** của mỗi user (trả 402, chỉ reset
> khi sang tháng mới vì key là `cost:<user>:<YYYY-MM>`).
> - Rate limit cho qua nhưng cost guard chặn: user gửi đều 5 request/phút (dưới hạn 10) nhưng
>   câu hỏi rất dài và history 20 message làm `tokens_in` lớn; sau vài ngày tổng chi phí vượt
>   10 USD → 402 dù chưa bao giờ gọi nhanh.
> - Cost guard cho qua nhưng rate limit chặn: user mới trong tháng (chi tiêu gần 0) chạy vòng
>   lặp gửi 15 câu "test" trong vài giây → từ request thứ 11 bị 429, dù mỗi câu chỉ tốn khoảng
>   0.00002 USD.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp làm một và cho nó kiểm tra Redis:
> 1. Redis mất kết nối → cả 3 container gọi Redis thất bại cùng lúc.
> 2. Health check của cả 3 cùng trả 503; sau vài lần liên tiếp (retries) orchestrator đánh
>    dấu cả 3 là unhealthy.
> 3. Orchestrator restart cả 3 gần như cùng lúc → request đang xử lý bị cắt, không còn
>    instance nào nhận traffic.
> 4. Container mới lên nhưng health vẫn fail vì Redis chưa về → tiếp tục bị restart (loop).
> 5. Redis quay lại sau 30 giây nhưng các container đang giữa vòng restart/backoff nên service
>    còn chết thêm một lúc. Sự cố 30 giây của Redis thành sự cố toàn hệ thống.
>
> Tách ra thì `/health` (không đụng Redis) vẫn 200 → không ai restart; chỉ `/ready` trả 503
> → load balancer tạm ngừng gửi request; Redis về thì `/ready` 200 và phục vụ tiếp ngay.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy 3 container `agent` phía sau nginx (round-robin), dùng chung một Redis, gọi `/ask`
> 5 lần với `X-User-Id: sv-scale`. Log cho thấy request lần lượt vào agent-3 → agent-2 →
> agent-1 → agent-3 → agent-2, nhưng `history_length` vẫn tăng đều 0 → 2 → 4 → 6 → 8 vì lịch
> sử nằm trong Redis mà cả 3 cùng đọc.
> Nếu lưu trong dict Python, mỗi container có dict riêng: lượt 1 (agent-3) = 0, lượt 2
> (agent-2) = 0, lượt 3 (agent-1) = 0, lượt 4 quay lại agent-3 = 2, lượt 5 (agent-2) = 2.
> Con số nhảy lung tung và nhỏ hơn thực tế — agent "mất trí nhớ" tùy request rơi vào
> container nào, và restart container là mất sạch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi: lần deploy đầu tiên lên Render bị `==> Timed Out` sau khoảng 15 phút, dù log cho thấy
> uvicorn đã chạy (`Application startup complete`).
> Cách tìm nguyên nhân: tôi đọc runtime log bằng `render logs --resources <service-id>` và thấy
> health check của Render liên tục gọi `"GET /Program%20Files/Git/health HTTP/1.1" 404 Not Found`.
> Tôi đã tạo service bằng Render CLI từ Git Bash trên Windows với `--health-check-path /health`,
> và Git Bash (MSYS) tự đổi tham số bắt đầu bằng `/` thành đường dẫn Windows
> `C:/Program Files/Git/health`.
> Cách sửa: chạy `MSYS_NO_PATHCONV=1 render services update <id> --health-check-path /health`
> để tắt việc tự chuyển đường dẫn, kiểm tra lại thấy `healthCheckPath: /health`, rồi tạo deploy
> mới; lần này deploy `live` sau khoảng 1 phút và log hiện `"GET /health HTTP/1.1" 200 OK`.
