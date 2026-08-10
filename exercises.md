# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dương Ngọc Tiến  Mã học viên: 2A202601401

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định `agent_api_key="changeme"`, khi deploy lên cloud mà quên cấu hình biến này, ứng dụng vẫn chạy bình thường. Kẻ xấu nếu đoán được khóa "changeme" có thể tự do gọi `/ask` miễn phí và đốt hết ngân sách gọi LLM đắt tiền của bạn. Nhờ cơ chế "fail fast", app sẽ chết ngay từ lúc khởi động, báo lỗi thiếu biến, giúp ta phát hiện và sửa lỗi ngay trước khi API vô tình bị công khai với mật khẩu yếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log: `{"timestamp": "2026-08-10T12:00:00.000Z", "level": "INFO", "event": "ask_completed", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0002}`
> 1. Đẩy log vào các hệ thống phân tích (như ELK, Datadog) để tạo biểu đồ tổng chi phí tự động.
> 2. Dùng các script tự động để trích xuất (parse) và đếm xem User ID nào sử dụng lượng token nhiều nhất mà không cần viết regex tách chuỗi thủ công.

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
| 1 stage (bản đầu) | ~ 1000 MB |
| Multi-stage | ~ 150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch đó là các bộ công cụ phát triển (như gcc, make...), các thư viện nguồn C++, header files và bộ nhớ đệm (cache) của lệnh `pip`. Multi-stage build đã bỏ lại toàn bộ những thứ dư thừa này ở stage 1 và chỉ mang thư viện đã biên dịch (chạy được) sang image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Nếu sửa code trong `app/main.py`, layer từ `FROM` đến `RUN pip install` vẫn giữ được cache, Docker chỉ build lại từ layer `COPY app/ app/` trở xuống, rất nhanh chóng. Nếu đặt `COPY . .` lên trước `RUN pip install`, mọi thay đổi trong mã nguồn (kể cả sửa một comment nhỏ) sẽ làm vô hiệu hóa cache của Docker, khiến lệnh `pip install` phải tải và cài lại toàn bộ các thư viện rất tốn thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Khi chạy dưới quyền root, nếu mã Python bị khai thác lỗ hổng thực thi mã từ xa (RCE), kẻ tấn công sẽ giành được quyền root bên trong container. Từ đây, họ có thể khai thác các công cụ như Docker socket để "vượt ngục" (breakout) và chiếm quyền root trên toàn bộ máy host thật. Việc dùng lệnh `USER appuser` giới hạn quyền kẻ tấn công ở mức người dùng thường ngay sau RCE, khiến chúng không thể can thiệp sâu vào hệ thống hay leo thang đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Bởi vì người dùng có thể căn gửi 10 request lúc 10:00:59, hệ thống cộng dồn thành 10 và cho qua. Ngay 1 giây sau, đồng hồ điểm 10:01:00, bộ đếm bị reset về 0, họ lập tức có thể gửi thêm 10 request lúc 10:01:01. Hạn mức bị phá vỡ một cách dễ dàng, thuật toán cửa sổ trượt ra đời để vá lỗ hổng tính giờ cố định này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm số *lần* request theo thời gian ngắn (phút), còn Cost guard đếm số *tiền* bị tiêu hao theo thời gian dài (tháng).
> - Rate limit cho qua nhưng cost guard chặn: Một user gửi 5 request (vẫn dưới 10 req/phút), nhưng hỏi những câu siêu phức tạp, tốn rất nhiều token và nhanh chóng đốt sạch hạn mức 10 USD của tháng.
> - Cost guard cho qua nhưng rate limit chặn: Một user viết bot ping /ask với câu "hi" 30 lần mỗi phút. Do câu rất ngắn nên chi phí rẻ (chưa hết 10 USD), nhưng tần suất quá nhanh bị Rate limit chặn lại để bảo vệ server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối.
> 2. Cả 3 container đều trả về lỗi ở /health.
> 3. Hệ thống quản lý (K8s/Docker Swarm) tưởng cả 3 container đang bị treo/chết vì healthcheck thất bại.
> 4. K8s quyết định ép dừng (kill) và khởi động lại toàn bộ 3 container cùng lúc.
> 5. Ứng dụng sập hoàn toàn và mọi request của người dùng đang được phục vụ bị gián đoạn oan uổng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu trong RAM bằng dict Python, biến `history_length` sẽ nhảy số bất thường (ví dụ: 1 -> 1 -> 2 -> 1). Vì có 3 instance độc lập A, B, C; request 1 đẩy vào A thì A nhớ 1, request 2 bị load balancer chuyển sang B thì B tạo mảng mới, đếm lại từ 1, khiến agent biểu hiện như "mất trí nhớ". Nhờ lưu trên Redis (state ngoài process), dữ liệu hội thoại trở thành tài nguyên chung (stateless backend) nên độ dài lịch sử sẽ tăng đều 1, 2, 3...

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi "Healthcheck failure" ngay sau khi Deploy lên Render. Thông báo lỗi trong Build Log hiện: `ValidationError: 1 validation error for Settings - agent_api_key (Field required)`. Tôi tìm ra nguyên nhân là do chưa set cấu hình trên cloud khiến pydantic "Fail Fast". Tôi đã vào tab Environment (Variables) trên giao diện quản lý của Render, thêm biến `AGENT_API_KEY` và `REDIS_URL`, sau đó Deploy lại thì thành công rực rỡ.
