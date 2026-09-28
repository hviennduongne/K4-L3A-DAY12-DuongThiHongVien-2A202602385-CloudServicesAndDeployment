# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dương Thị Hồng Viên  Mã học viên: 2A202602385

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ khi deploy mà quên đặt `AGENT_API_KEY`, fail fast làm container dừng ngay và log báo thiếu cấu hình. Tôi có thể sửa biến môi trường trước khi service nhận request. Nếu dùng mặc định `changeme`, service vẫn chạy và người ngoài có thể đoán được khóa để gọi `/ask`.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật trên Railway: `event="ask_completed" timestamp="2026-09-28T09:57:23.230426+00:00" user_id="sv-deploy-check" tokens_in=3 tokens_out=35 cost_usd=0.00002145`. Từ các trường này tôi có thể lọc hoặc đếm request theo `user_id`, đồng thời cộng `cost_usd` để theo dõi chi phí. Một câu `print("đã trả lời xong")` không có dữ liệu có cấu trúc để làm hai việc đó.

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
| 1 stage (bản đầu) | khoảng 1.120 MB |
| Multi-stage | 247 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu đến từ base image `python:3.11` đầy đủ, công cụ hệ thống và dữ liệu trung gian dùng lúc cài package. Bản multi-stage dùng `python:3.11-slim` và chỉ chép dependency đã cài cùng mã nguồn sang runtime nên không mang toàn bộ môi trường build vào image chạy thật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt` và `RUN pip install` vẫn lấy từ cache; từ layer `COPY app ./app` trở đi phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi mã nguồn làm layer COPY đổi, vì vậy Docker phải chạy cài toàn bộ dependency lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗi cho phép thực thi lệnh, kẻ tấn công có thể chạy lệnh với quyền của process trong container. Khi process là root, một lỗi thoát container hoặc volume/socket được gắn sai có thể cho quyền rất cao trên host. `USER appuser` làm process bị khai thác chỉ có UID thường từ trước lúc Uvicorn chạy, nhờ đó giảm quyền mà kẻ tấn công nhận được.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây: gửi 10 request ngay trước giây 00, rồi gửi thêm 10 request ngay sau giây 00 khi bộ đếm phút mới đã reset. Sliding window nhìn lại đúng 60 giây nên vẫn thấy nhóm request trước và chặn cách lách ranh giới này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ request trong một khoảng ngắn, còn cost guard giới hạn tổng tiền đã tiêu trong tháng. Một user gọi chậm dưới 10 request/phút nhưng mỗi request rất đắt vẫn được rate limit cho qua và bị cost guard chặn khi hết ngân sách. Ngược lại, đầu tháng còn nhiều ngân sách nhưng gửi 11 request liên tiếp thì cost guard vẫn cho phép về mặt chi phí, còn rate limit chặn request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Khi Redis mất kết nối, endpoint gộp sẽ trả lỗi. Liveness probe xem container là hỏng và lần lượt restart cả ba container. Các container mới vẫn không nối được Redis nên tiếp tục bị đánh dấu lỗi và restart, dù bản thân web process vẫn chạy được. Tách `/health` giúp container không bị restart oan, còn `/ready` loại instance khỏi nhận traffic cho tới khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi lưu bằng Redis, cả ba container đọc cùng một lịch sử nên `history_length` tăng đều theo các lần gọi của cùng `X-User-Id`. Nếu dùng dict Python, load balancer có thể chuyển request sang container khác; mỗi container có dict riêng nên số có thể nhảy như 0, 2, 0, 4 hoặc giảm khi request tới instance chưa thấy lịch sử trước đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi deployment báo thành công, gọi URL công khai vẫn nhận `502 Application failed to respond`. Tôi đọc Railway logs và thấy Uvicorn đang nghe ở `0.0.0.0:8080`, trong khi domain đã được tạo với target port 8000. Tôi cập nhật target port của domain thành 8080; sau đó `/health` và `/ready` đều trả 200, còn `/ask` không có key trả 401.
