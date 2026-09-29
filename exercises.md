# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Văn Sang  Mã học viên: 2A202602391

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là khi deploy nhưng quên đặt `AGENT_API_KEY`: ứng dụng dừng ngay
> với lỗi validation nên tôi biết cấu hình cloud còn thiếu. Nếu dùng khóa mặc
> định `"changeme"`, service vẫn public và bot có thể đoán khóa để gọi `/ask`,
> làm phát sinh chi phí trước khi tôi phát hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi quan sát được là
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T02:49:50.090025+00:00","user_id":"sv-cp4","tokens_in":3,"tokens_out":37,"cost_usd":2.265e-05}`.
> Từ log này tôi có thể lọc/tổng hợp chi phí theo `user_id`, và tính số request
> hoặc tỷ lệ lỗi theo khoảng thời gian. Dòng `print("đã trả lời xong")` không có
> trường dữ liệu ổn định để máy thực hiện hai việc đó.

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
| 1 stage (bản đầu) | khoảng 1700 MB (`docker images` hiển thị 1.7 GB) |
| Multi-stage | 274 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build và đo thật được bản một stage 1.7 GB, còn bản multi-stage dùng
> `python:3.11-slim` là 274 MB. Phần chênh lệch chủ yếu đến từ base image Python
> đầy đủ chứa nhiều package/hệ thống phục vụ phát triển; bản cuối chỉ giữ slim
> runtime và các dependency được copy từ builder, không mang toàn bộ môi trường
> build sang image chạy production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi chỉ sửa source, layer `COPY requirements.txt` và layer cài dependency
> được dùng lại từ cache; các layer copy `app/` và những layer đứng sau nó phải
> tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, bất kỳ thay đổi code nào
> cũng làm mất cache của layer cài dependency, khiến pip tải/cài lại toàn bộ gói.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng thực thi lệnh, kẻ tấn công trước hết chạy được
> lệnh với UID của process trong container. Khi process là root, lỗi cấu hình
> mount hoặc một lỗ hổng container runtime có thể biến quyền đó thành quyền cao
> trên host. `USER appuser` cắt chuỗi tại bước đầu: mã bị khai thác chỉ có quyền
> của user thường và không thể tùy ý sửa file hệ thống trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong khoảng 2 giây: gửi 10 request ở cuối phút, ví dụ
> `10:00:59`, rồi gửi tiếp 10 request ngay sau khi bộ đếm reset ở `10:01:00`.
> Sliding window nhìn lại đúng 60 giây nên vẫn thấy cả hai nhóm và chặn nhóm sau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong 60 giây, còn cost guard giới hạn
> tổng tiền theo user trong tháng UTC. Một user gửi ít request nhưng prompt rất
> lớn có thể qua rate limit nhưng bị cost guard chặn. Ngược lại, user gửi nhiều
> request rẻ trong vài giây có thể bị 429 dù tổng chi phí tháng vẫn rất thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng ping Redis, khi Redis mất kết nối cả ba container đồng loạt
> trả 503. Orchestrator coi cả ba process đã chết và restart chúng, dù code ứng
> dụng vẫn sống; Redis vẫn chưa phục hồi nên các container mới lại fail, tạo vòng
> lặp restart. Tách `/ready` giúp load balancer tạm ngừng gửi traffic mà không
> restart process; `/health` vẫn 200 và service tự nhận traffic lại khi Redis ổn.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Lần thử scale thật của tôi dừng ở instance thứ hai vì Compose map cố định
> `8000:8000`, nên Docker báo `Bind for 0.0.0.0:8000 failed: port is already
> allocated`. Tôi vẫn xác nhận tính stateless bằng cách gửi một lượt, restart
> container rồi gửi tiếp: response sau restart có `history_length: 2`, vì history
> nằm trong Redis. Nếu dùng dict Python, mỗi instance có dict riêng nên cùng user
> sẽ thấy `history_length` tăng giảm hoặc quay về 0 tùy request rơi vào instance
> nào; restart cũng làm mất toàn bộ history.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi deploy thật tôi gặp là `/ready` và `/ask` trả 500. Log Railway ghi
> `ValueError: Redis URL must specify one of the following schemes (redis://,
> rediss://, unix://)`. Tôi kiểm tra biến theo tên và định dạng bằng Railway CLI,
> phát hiện `REDIS_URL` của agent có scheme không hợp lệ. Tôi lấy đúng
> `REDIS_URL` từ service `day12-redis`, gắn vào `day12-agent` rồi chờ redeploy;
> sau đó `/ready` trả 200, request có key trả 200 và rate limit trả 429 từ lần 11.
