# Phiếu Phản Ánh --- K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì
> bạn quan sát được khi chạy code --- không sao chép đáp án của người
> khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tuấn Vũ Mã học viên: 2A202601845

------------------------------------------------------------------------

### Câu 1 --- Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết
ngay khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống
cụ thể mà việc "chết sớm" này cứu bạn, so với việc để mặc định
`"changeme"`.

> Khi deploy service lên cloud, nếu tôi quên cấu hình `AGENT_API_KEY` mà
> chương trình vẫn dùng mặc định `"changeme"`, service vẫn có thể khởi
> động và public ra Internet với một khóa rất dễ đoán. Khi đó người khác
> có thể gọi `/ask` bằng khóa mặc định và sử dụng tài nguyên của
> service. Với cách fail fast, ứng dụng báo lỗi ngay khi khởi động vì
> thiếu biến môi trường bắt buộc. Tôi phát hiện lỗi cấu hình trước khi
> service nhận traffic và buộc phải đặt secret đúng trên môi trường
> deploy.

------------------------------------------------------------------------

### Câu 2 --- Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được,
rồi nêu **hai** việc bạn làm được với dòng log đó mà
`print("đã trả lời xong")` không làm được.

> Khi `/ask` xử lý thành công, code của tôi gọi
> `log_event("ask_completed", ...)` và ghi các trường `user_id`,
> `tokens_in`, `tokens_out`, `cost_usd`. Một dòng log theo cấu trúc đó
> có dạng:
>
> `{"event":"ask_completed","level":"info","timestamp":"...","user_id":"sv01","tokens_in":41,"tokens_out":46,"cost_usd":0.00003375}`
>
> Hai việc tôi làm được với log JSON mà `print("đã trả lời xong")` không
> làm được:
>
> 1.  Lọc và thống kê theo từng trường, ví dụ tìm request của một
>     `user_id`, cộng `cost_usd`, hoặc theo dõi số token input/output.
> 2.  Đưa log vào hệ thống thu thập log để máy parse tự động theo
>     `event`, `level`, `timestamp` và tạo dashboard/cảnh báo. Với một
>     chuỗi `print()` không có cấu trúc, các thông tin này không được
>     tách thành field rõ ràng.

------------------------------------------------------------------------

### Câu 3 --- Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

``` bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

  -----------------------------------------------------------------------
  Bản                                          Dung lượng
  -------------------------------------------- --------------------------
  1 stage (bản đầu)                            **CHƯA ĐO LẠI --- cần chạy
                                               bản 1-stage trước khi
                                               nộp**

  Multi-stage                                  **270 MB disk usage; 63.7
                                               MB content size (số đã
                                               quan sát trên máy)**
  -----------------------------------------------------------------------

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Dockerfile hiện tại của tôi dùng hai stage. Stage `builder` cài
> dependency vào `/install`, còn stage runtime dùng `python:3.11-slim`
> và chỉ copy dependency đã cài cùng source `app/` và `utils/`. Vì vậy
> image runtime không cần mang theo toàn bộ môi trường trung gian của
> quá trình build. Tôi cũng dùng `pip install --no-cache-dir` để không
> giữ pip cache trong image. **Tôi chưa có số đo thật của bản 1-stage
> trong quá trình kiểm tra hiện tại, nên không ghi một con số ước lượng.
> Trước khi nộp câu này, tôi cần build lại bản 1-stage và thay ô
> `CHƯA ĐO LẠI` bằng kết quả thực tế.**

------------------------------------------------------------------------

### Câu 4 --- Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn,
những layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn
đặt `COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của tôi copy `requirements.txt` và chạy `pip install` trước
> khi copy source code. Vì vậy khi tôi chỉ sửa `app/main.py` mà
> `requirements.txt` không đổi, các layer từ base image, `WORKDIR`,
> `COPY requirements.txt` và `RUN pip install` có thể được dùng lại từ
> cache. Layer `COPY app/ app/` bị thay đổi nên Docker phải tạo lại
> layer đó và các layer phụ thuộc phía sau nếu cần. Nếu đặt `COPY . .`
> trước `RUN pip install`, chỉ cần sửa một file source thì layer
> `COPY . .` thay đổi, cache của các layer sau bị invalid và
> `pip install` phải chạy lại dù dependency không đổi. Cách hiện tại
> giúp các lần build sau khi sửa code nhanh hơn.

------------------------------------------------------------------------

### Câu 5 --- Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ
hổng trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy
host", và lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng cho phép thực thi mã từ xa, kẻ tấn công
> trước hết chiếm quyền của process đang chạy trong container. Nếu
> process chạy bằng root, mã của họ cũng có quyền root bên trong
> container, làm hậu quả của lỗ hổng nghiêm trọng hơn và tăng khả năng
> lợi dụng thêm cấu hình sai hoặc lỗ hổng container/kernel để tác động
> tới host. Dockerfile của tôi tạo `appuser` và dùng `USER appuser`, nên
> ngay cả khi process Python bị chiếm quyền, kẻ tấn công trước tiên chỉ
> có quyền của user thường trong container. `USER` không tự loại bỏ mọi
> khả năng container escape, nhưng nó cắt bớt đặc quyền ngay tại process
> ứng dụng và giảm phạm vi thiệt hại.

------------------------------------------------------------------------

### Câu 6 --- Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm
theo phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa
bao nhiêu request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải
thích cách đạt được con số đó.

> Tối đa là **20 request**. Với fixed window theo phút, người dùng có
> thể gửi đủ 10 request ở cuối phút hiện tại, ví dụ quanh giây 59. Khi
> sang giây 00 của phút tiếp theo, bộ đếm reset và họ gửi ngay thêm 10
> request. Như vậy trong khoảng rất ngắn quanh ranh giới hai phút có thể
> có 20 request dù giới hạn ghi là 10 request/phút. Sliding window của
> tôi đếm các request trong 60 giây gần nhất bằng Redis Sorted Set nên
> không bị lỗ hổng ở ranh giới phút như vậy.

------------------------------------------------------------------------

### Câu 7 --- Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit
cho qua nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit của tôi giới hạn **tần suất request trong 60 giây**, còn
> cost guard giới hạn **chi phí tích lũy theo user trong tháng**. Hai cơ
> chế bảo vệ hai tài nguyên khác nhau.
>
> -   Rate limit cho qua nhưng cost guard chặn: một user chỉ gửi 1
>     request nên vẫn nằm dưới giới hạn request/phút, nhưng chi phí đã
>     ghi nhận trong tháng đã vượt ngân sách cấu hình. `guard.check()`
>     sẽ trả 402 trước khi gọi LLM.
> -   Cost guard cho qua nhưng rate limit chặn: một user vẫn còn ngân
>     sách tháng nhưng gửi request liên tục đến khi số request trong cửa
>     sổ 60 giây đạt giới hạn. Rate limiter sẽ trả 429 dù user vẫn còn
>     tiền trong budget.
>
> Trong `/ask`, tôi kiểm tra rate limiter rồi cost guard trước khi gọi
> mock LLM để request không hợp lệ bị chặn trước bước phát sinh thêm xử
> lý/chi phí.

------------------------------------------------------------------------

### Câu 8 --- /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra
với cụm 3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ
tự sự kiện.

> Nếu gộp liveness và readiness rồi để endpoint đó phụ thuộc Redis, thứ
> tự có thể xảy ra là: (1) Redis mất kết nối; (2) cả 3 agent vẫn còn
> process sống nhưng health check phụ thuộc Redis bắt đầu trả lỗi; (3)
> nếu orchestrator dùng endpoint đó làm liveness, nó hiểu nhầm agent đã
> chết và restart container; (4) Redis vẫn chưa phục hồi nên container
> vừa lên lại tiếp tục fail health check; (5) các instance có thể rơi
> vào vòng restart trong khi nguyên nhân thật nằm ở dependency Redis.
> Tách `/health` và `/ready` giúp phân biệt hai trạng thái: `/health`
> cho biết process agent còn sống, còn `/ready` kiểm tra agent có sẵn
> sàng nhận traffic và Redis có truy cập được hay không. Khi Redis lỗi,
> instance nên bị rút khỏi traffic qua readiness thay vì bị restart chỉ
> vì dependency tạm thời mất kết nối.

------------------------------------------------------------------------

### Câu 9 --- Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với
cùng một `X-User-Id`. Quan sát `history_length` trong response. Nếu lịch
sử được lưu trong một dict Python thay vì Redis, bạn sẽ thấy con số đó
thay đổi thế nào?

> Khi tôi scale `agent=3`, các instance dùng chung Redis nên lịch sử của
> cùng một `X-User-Id` vẫn được đọc lại giữa các request dù request có
> thể rơi vào container khác nhau. Tôi cũng kiểm tra Redis và thấy các
> message user/assistant của cùng user được lưu liên tiếp trong key lịch
> sử chung.
>
> Nếu thay Redis bằng một `dict` Python trong từng process, mỗi
> container sẽ có một bản lịch sử riêng. Khi load balancer phân phối
> request qua A, B, C, `history_length` sẽ không tăng nhất quán theo
> toàn bộ cuộc hội thoại: một request rơi vào container chưa từng phục
> vụ user có thể thấy lịch sử ngắn hoặc rỗng, còn request quay lại
> container cũ lại thấy lịch sử khác. Khi container restart, phần lịch
> sử trong RAM của container đó cũng mất. Đây là lý do state hội thoại
> phải đặt ngoài process.

------------------------------------------------------------------------

### Câu 10 --- Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health
check timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi
là gì, bạn tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp trên Railway là Uvicorn không nhận được giá trị số của
> cổng từ `$PORT`; log deploy cho thấy giá trị `$PORT` bị truyền nguyên
> văn vào tham số `--port`, khiến service không startup đúng và health
> check thất bại. Tôi tìm nguyên nhân bằng cách xem Deploy Logs và đối
> chiếu `railway.toml`. Khi đó `startCommand` gọi trực tiếp
> `uvicorn ... --port $PORT`, nhưng biến môi trường chưa được shell
> expand. Tôi sửa thành:
>
> `startCommand = "sh -c 'exec uvicorn app.main:app --host 0.0.0.0 --port $PORT'"`
>
> Sau khi commit và push, Railway deploy lại và `/health` trả HTTP 200.
> Tôi tiếp tục cấu hình `AGENT_API_KEY` và `REDIS_URL` trong Railway
> Variables; sau khi Redis được nối đúng, `/ready` cũng trả trạng thái
> ready với `redis: true`.
