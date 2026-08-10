# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời mặc định bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tuấn Vũ  Mã học viên: 2A202601845

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường production (Cloud/K8s/Docker), nếu lỡ quên không cấu hình biến môi trường `AGENT_API_KEY`, việc dùng giá trị mặc định `"changeme"` sẽ làm ứng dụng vẫn khởi động thành công và âm thầm chạy trên Internet. Kẻ tấn công hoặc bot tự động quét lỗ hổng có thể thử các API key phổ biến như `"changeme"` để gọi endpoint `/ask`, liên tục tiêu tốn tiền API OpenAI/Gemini của bạn mà bạn không hề hay biết cho tới khi nhận hóa đơn tài chính. Việc cho ứng dụng "chết sớm" (Crash ngay khi khởi động nếu thiếu `AGENT_API_KEY`) buộc developer/sysadmin phát hiện lỗi cấu hình lập tức tại bước deployment (`docker compose up` dừng ngay), ngăn chặn việc mở cổng dịch vụ công khai mà chưa được bảo mật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_request", "level": "info", "timestamp": "2026-08-10T12:00:00.123456+00:00", "user_id": "user-123", "question_length": 15, "cost_usd": 0.0003, "duration_ms": 142}`

Hai việc làm được với log JSON:
1. **Lọc và phân tích tự động bằng công cụ Log Management (Datadog/Grafana Loki/ELK):** Có thể trích xuất chính xác các trường dữ liệu cấu trúc như `user_id`, `cost_usd`, `duration_ms` để tính tổng chi phí theo người dùng hoặc đặt cảnh báo khi thời gian phản hồi `duration_ms > 1000ms`, điều mà chuỗi văn bản thuần của `print()` không thể parse chính xác ở quy mô lớn.
2. **Cảnh báo và tự động hiển thị Dashboard theo thời gian thực:** Dựa vào trường `timestamp` chuẩn ISO-8601 và `level` (info/error), các hệ thống tự động có thể đếm tần suất lỗi theo khoảng thời gian để gửi thông báo Slack/PagerDuty ngay lập tức khi số lượng lỗi tăng đột biến mà không cần đọc log bằng mắt.

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
| 1 stage (bản đầu) | 980 MB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~800 MB) bao gồm: các công cụ biên dịch mã nguồn C/C++ (gcc, g++, make), các file header thư viện phát triển (`python3-dev`), công cụ build tool (pip wheel cache), tài liệu hướng dẫn, các thư viện hệ thống thừa và các package không cần thiết ở môi trường runtime. Ở bản Multi-stage, stage `builder` dùng để biên dịch và cài đặt thư viện vào thư mục tạm, sau đó stage production (`python:3.11-slim`) chỉ copy các file thư viện đã build xong, loại bỏ toàn bộ bộ biên dịch và các file rác phát triển.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa 1 ký tự trong `app/main.py`: Các layer cài đặt môi trường từ `FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install` ở stage builder và các layer chuẩn bị môi trường runtime đều được Docker tái sử dụng lại từ cache 100%. Chỉ có các layer từ `COPY app/ app/` trở về sau mới phải chạy lại, giúp quá trình build hoàn thành chỉ trong 2-3 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Bất kỳ thay đổi nhỏ nào trong file mã nguồn cũng làm Docker invalidate (hỏng) cache của layer `COPY . .`. Do đó, Docker sẽ phải chạy lại lệnh `RUN pip install` từ đầu, tải và cài đặt lại toàn bộ thư viện Python từ Internet mỗi lần bạn chỉnh sửa code, làm quá trình build cực kỳ chậm.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công khai thác một lỗ hổng trong code Python (ví dụ RCE qua unsafe deserialization hoặc gọi lệnh hệ thống).
2. Do container chạy dưới quyền `root` (UID 0), tiến trình Python có full quyền ghi/sửa mọi file và gọi mọi lệnh trong container.
3. Kẻ tấn công thực thi mã độc để thực hiện kịch bản "Container Escape" (thoát khỏi container) thông qua lỗ hổng Kernel, mount nhầm Docker socket (`/var/run/docker.sock`), hoặc cgroup.
4. Vì tiến trình bên trong container mang UID 0 (`root`), khi thoát ra máy host thành công, kẻ tấn công lập tức chiếm được quyền `root` trên hệ thống máy host.

Lệnh `USER appuser` cắt đứt chuỗi ngay tại Bước 2: Bằng cách chuyển quyền thực thi sang một user thường (non-root, no-shell), kẻ tấn công dù chiếm được quyền điều khiển ứng dụng Python cũng bị giới hạn quyền nghiêm ngặt, không thể sửa các file hệ thống container, không có đặc quyền Linux capabilities để thực hiện Container Escape ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa trong 2 giây liên tiếp: **20 request**.
- Giải thích: Với cơ chế đếm theo phút đồng hồ cố định (Fixed Window), hạn mức được reset về 0 tại giây 00 mỗi phút. Người dùng có thể gửi 10 request ở giây 59 của phút thứ nhất. Ngay sau đó ở giây 00 của phút thứ hai (chỉ cách 1 giây), bộ đếm reset về 0, họ gửi tiếp 10 request nữa. Kết quả là trong khoảng thời gian chỉ 2 giây (giây 59 và giây 00), người dùng gửi thành công 20 request (gấp 2 lần hạn mức 10 req/phút), tạo ra xung đột tải (traffic spike). Cơ chế Sliding Window tính trong cửa sổ 60 giây trượt thực tế nên ngăn chặn được lỗ hổng này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt:
  - **Rate Limit**: Giới hạn **số lượng request/tần suất truy cập** trong thời gian ngắn (vd: 10 req/phút) nhằm bảo vệ hạ tầng máy chủ khỏi bị nghẽn mạng và DDoS.
  - **Cost Guard**: Giới hạn **tổng chi phí tài chính / mức tiêu thụ token** trong thời gian dài (vd: 10 USD/tháng) nhằm bảo vệ ví tiền của chủ ứng dụng khỏi bị kiệt quệ tài chính.
- Tình huống Rate limit cho qua nhưng Cost guard chặn: Người dùng gửi duy nhất 1 request trong phút (hoàn toàn hợp lệ với Rate limit 10 req/phút), nhưng request này yêu cầu tóm tắt một tài liệu PDF cực lớn tốn hết $15 tiền token. Nếu budget còn lại của user chỉ là $0.5, Cost guard sẽ chặn request này (402/429) dù Rate limit hoàn toàn cho qua.
- Tình huống Cost guard cho qua nhưng Rate limit chặn: Người dùng mới chi tiêu $0.10 trong tháng (còn rất nhiều budget), nhưng họ chạy script gửi 15 request liên tiếp chỉ trong 5 giây. Cost guard đồng ý vì chi phí thấp, nhưng Rate limit sẽ chặn từ request thứ 11 (429 Rate Limit Exceeded) để tránh làm sập server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Kết nối tới Redis bị gián đoạn trong 30 giây.
2. Endpoint gộp (đóng vai trò Liveness Probe) thực hiện check Redis và trả về HTTP 500/503.
3. Orchestrator (Docker Compose/Cloud Runner/Kubernetes) nhận thấy Liveness probe rớt nên kết luận cả 3 container agent đã "chết".
4. Orchestrator lập tức kill và khởi động lại (restart) cả 3 container agent cùng lúc.
5. Trong 30 giây Redis chưa khôi phục, các container mới khởi động lại tiếp tục check Redis thất bại và lại bị kill liên tục (tạo thành vòng lặp CrashLoopBackOff/Restart loop).
6. Toàn bộ hệ thống bị sập hoàn toàn, các request dở dang bị hủy đứt đoạn thay vì chỉ tạm thời ngưng nhận traffic mới (Readiness probe) cho đến khi Redis sống lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong `dict` Python in-memory của từng container:
Khi scale 3 container, Load Balancer phân phối các request của cùng một `X-User-Id` luân phiên tới 3 container khác nhau. Khi đó `history_length` trả về sẽ **nhảy số thất thường, không tăng liên tục và mất nhất quán**.
Ví dụ: Request 1 vào Container A (`history_length` = 1). Request 2 vào Container B (`history_length` = 1). Request 3 lại vào Container A (`history_length` = 2). Request 4 vào Container C (`history_length` = 1). Người dùng sẽ thấy lịch sử trò chuyện bị mất đoạn tùy thuộc vào request rơi vào container nào, vì bộ nhớ `dict` không được chia sẻ giữa 3 container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** `Healthcheck failed: Connection refused` / `Port 8000 not responding`.
- **Cách tìm nguyên nhân:** Kiểm tra log trên Railway Dashboard sau khi deploy. Phát hiện dịch vụ Railway tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=6421`), trong khi lệnh khởi chạy Uvicorn trong Dockerfile lại cố định cổng 8000 (`--port 8000`), khiến Nginx/Railway Proxy không thể chuyển tiếp request vào container.
- **Cách sửa:** Chỉnh sửa lệnh CMD trong Dockerfile để đọc động cổng từ biến `$PORT` với fallback mặc định là 8000: `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`. Sau khi gán biến môi trường `$PORT`, Railway health check đã chuyển sang HTTP 200 OK.
