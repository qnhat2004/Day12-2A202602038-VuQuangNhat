# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mặc định bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Quang Nhật  Mã học viên: 2A202602038

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi triển khai ứng dụng lên Render/Railway, nếu để giá trị mặc định AGENT_API_KEY="changeme", ứng dụng sẽ vẫn khởi động thành công. Tuy nhiên, người lạ có thể tìm thấy Public URL và gọi API hoàn toàn miễn phí bằng khóa mặc định "changeme" này, làm phát sinh chi phí gọi LLM lớn. Khi ứng dụng "fail fast" không cho phép mặc định, app sẽ ngay lập tức sập lúc bật lên, buộc quản trị viên phải cung cấp biến môi trường thật trên Cloud Dashboard trước khi dịch vụ đi vào hoạt động.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:31:40.270299+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0000288}`

Hai việc làm được với log JSON:
1. Cho phép các công cụ monitoring (Grafana/Datadog) tự động parse các trường cost_usd, tokens_in, tokens_out để vẽ biểu đồ chi phí và đo lưu lượng theo thời gian thực.
2. Dễ dàng truy vấn, lọc lỗi theo từng user_id cụ thể trong hàng triệu dòng log mà không phải parse chuỗi thô.

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
| 1 stage (bản đầu) | ~1.01 GB |
| Multi-stage | 117 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~830MB) bao gồm: các công cụ biên dịch C/C++ (GCC, Make), bộ nhớ đệm pip cache, bộ cài Python full chuẩn Debian và các file thư viện tạm chỉ cần lúc build chứ không cần lúc chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa app/main.py: Các layer từ FROM python:3.11-slim AS builder tới COPY --from=builder /install /usr/local đều được dùng lại từ cache. Layer COPY app/ app/ và các lệnh đứng sau nó phải chạy lại.
Nếu đặt COPY . . lên trước RUN pip install: Mỗi lần sửa code, layer COPY . . bị thay đổi sẽ làm hủy cache của toàn bộ các lệnh đứng sau, khiến RUN pip install phải tải lại và cài đặt lại toàn bộ thư viện từ đầu, tốn thời gian build rất lâu.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Giả sử code Python có lỗ hổng RCE (Remote Code Execution) thông qua một thư viện bị lỗi -> Hacker gửi payload độc hại để thực thi câu lệnh hệ thống trong container -> Do container mặc định chạy quyền root, hacker có đầy đủ quyền hạn tối cao bên trong container -> Hacker lợi dụng các kĩ thuật container escape để khai thác nhân Linux và chiếm quyền kiểm soát máy host bên ngoài.
Lệnh USER appuser cắt đứt chuỗi tấn công ngay từ bước 3: Hacker khi xâm nhập vào chỉ sở hữu quyền hạn của user thường appuser (không có quyền cài phần mềm, không can thiệp được vào file hệ thống hay giao tiếp với nhân máy host).

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa gửi được 20 request trong 2 giây liên tiếp.
Cách đạt được: Người dùng gửi 10 request ở giây thứ 59 của phút 10:00 (hạn mức 10/phút vừa tròn). Ngay sau đó ở giây 00 của phút 10:01, hệ thống đếm theo phút đồng hồ tự động reset đếm về 0 -> Người dùng gửi tiếp 10 request ở giây thứ 01. Tổng cộng gửi được 20 request chỉ từ giây 59 đến giây 01 (khoảng thời gian 2 giây).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt: Rate limit quản lý tốc độ gọi (số request/đơn vị thời gian) để bảo vệ hạ tầng web. Cost guard quản lý tổng chi phí tài chính (số tiền tiêu trong tháng) để bảo vệ ngân sách.
- Rate limit cho qua nhưng Cost guard chặn: User gửi 1 request/phút (rất thong thả), nhưng câu hỏi đính kèm file tài liệu 200 trang làm phát sinh chi phí $5.00 -> Vượt ngân sách tháng $10 -> Cost guard trả lỗi 402.
- Cost guard cho qua nhưng Rate limit chặn: User mới bắt đầu tháng (còn dư nguyên ngân sách $10), dồn dập gửi 20 request ngắn ("Hi", "Hello") chỉ trong 2 giây -> Ngân sách còn dư nhiều nhưng bị Rate limit chặn với lỗi 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện khi Redis mất kết nối 30 giây:
1. Cả 3 container đều báo /health hỏng (HTTP 503) vì /health đi check kết nối Redis.
2. Orchestrator (Docker/Kubernetes) nhận thấy /health hỏng nên coi như 3 container đã bị treo/chết, tự động kill và ra lệnh khởi động lại (restart) cả 3 container.
3. Khi 3 container bật lại, Redis vẫn đang mất kết nối -> /health lại trả về 503 -> Docker tiếp tục kill và restart liên tục (vòng lặp crashloop).
4. Các request của người dùng đang được xử lý bị ngắt đột ngột giữa chừng và cả cụm ứng dụng bị sập hoàn toàn.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu dùng dict Python: Giá trị history_length trả về sẽ nhảy ngẫu nhiên và không tăng liên tục (ví dụ: 0, 0, 0, 2, 2...). Lý do là 3 container có 3 bộ nhớ RAM riêng biệt. Nginx điều phối round-robin đẩy 5 request lần lượt qua 3 container khác nhau. Container 2 không nhìn thấy lịch sử lưu trong RAM của Container 1, khiến agent bị "mất trí nhớ" tùy thuộc vào request rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Dockerfile parse error on line 46: unknown instruction: 1:{port}/health'.
Thông báo lỗi: Docker build không nhận diện được lệnh 1:{port}/health'.
Cách tìm nguyên nhân: Đọc thông báo log của docker build chỉ rõ dòng 45-46 trong Dockerfile. Phát hiện câu lệnh CMD trong HEALTHCHECK bị ngắt xuống dòng giữa chừng tại địa chỉ IP 127.0.0.1.
Cách sửa: Đưa toàn bộ câu lệnh CMD của HEALTHCHECK về nằm trên cùng một dòng trong Dockerfile.
