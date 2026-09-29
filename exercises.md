# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.

Họ và tên: Trần Anh Đăng  Mã học viên: 2A202602992

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Việc không để mặc định "changeme" giúp ứng dụng phát hiện lỗi cấu hình ngay khi vừa khởi động (fail-fast). Tình huống cụ thể: Khi deploy lên môi trường sản xuất, nếu quên set biến môi trường `AGENT_API_KEY`, container sẽ crash ngay từ lúc startup và báo lỗi rõ ràng trong log. Nếu để mặc định `"changeme"`, app vẫn khởi động bình thường nhưng bất kỳ ai cũng có thể dùng key mặc định `"changeme"` để truy cập API của bạn và đốt sạch ngân sách LLM.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON:
`{"event": "ask_completed", "level": "INFO", "service": "day12-agent", "timestamp": "2026-09-29T10:49:33.123456+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}`

Hai việc làm được:
1. Đẩy log tự động vào hệ thống tập trung (Elasticsearch/Datadog/CloudWatch) để truy vấn, lọc log theo field (`user_id`, `cost_usd`) và alert tự động khi chi phí tăng đột biến.
2. Thống kê, vẽ biểu đồ dashboard tổng lượng token tiêu thụ và tổng chi phí theo thời gian mà không cần viết regex để parse text.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1010 MB |
| Multi-stage | ~180 MB |

Giải thích: phần dung lượng chênh lệch (~830 MB) là các công cụ build, gcc/g++, pip cache, wheel cache, các file header C/C++ và các file rác trung gian hình thành trong quá trình biên dịch dependency ở stage builder đã bị loại bỏ, không copy sang stage runtime final image (`python:3.11-slim`).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa 1 ký tự trong `app/main.py` và build lại:
- Các layer trước `COPY app ./app` (như `FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install`) đều được dùng lại từ Docker cache (CACHED).
- Chỉ từ layer `COPY app ./app` trở đi mới bị invalidate cache và phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa bất kỳ dòng code nào, Docker sẽ bỏ cache và phải chạy lại lệnh `RUN pip install` tốn nhiều phút để tải lại tất cả dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Kẻ tấn công lợi dụng lỗ hổng RCE (Remote Code Execution) trong app Python để thực thi lệnh shell. Nếu container chạy với quyền `root`, kẻ tấn công có quyền root trong container, từ đó có thể tìm cách escape container (thông qua mount socket docker hoặc lỗ hổng kernel) để chiếm quyền root máy host. Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay từ bước đầu: kẻ tấn công bị giới hạn trong quyền của user thường, không thể truy cập các file hệ thống nhạy cảm hay thực hiện các thao tác đòi hỏi đặc quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Một người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Giải thích: Nếu đếm theo phút đồng hồ, user có thể gửi 10 request ở 10:00:59 (cuối phút thứ nhất) và gửi tiếp 10 request ở 10:01:01 (đầu phút thứ hai). Hệ thống reset counter ở 10:01:00 nên cả 2 đợt đều hợp lệ theo luật từng phút riêng biệt, nhưng trong thực tế user đã bắn 20 request chỉ trong khoảng thời gian 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt: Rate limit giới hạn **tần suất/số lượng request** trong khoảng thời gian ngắn (ví dụ: request/phút). Cost guard giới hạn **tổng chi phí tài chính (USD)** trong khoảng thời gian dài (tháng).
- Tình huống Rate Limit cho qua nhưng Cost Guard chặn: User chỉ gửi 1 request/phút (dưới rate limit 10 req/min), nhưng prompt quá dài khiến chi phí $2/request và tổng chi tiêu tháng của user đó vượt $10.
- Tình huống ngược lại: User mới tiêu $0.10 trong tháng (chưa vượt ngân sách $10), nhưng gửi liên tục 20 request trong 5 giây -> bị Rate Limit chặn 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Orchestrator (Docker/K8s) gọi `/health` thấy trả về 503 -> tưởng rằng container agent bị crash/deadlock.
2. Orchestrator lập tức kill và restart lại toàn bộ cụm 3 container agent.
3. Vì Redis vẫn chưa sống lại trong 30s, các container agent sau khi restart lại tiếp tục bị kill & restart vòng lặp liên tục (crash loop backoff), làm gián đoạn hệ thống nghiêm trọng dù bản thân mã nguồn agent không có lỗi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử được lưu trong dict Python local: Khi scale ra 3 container, mỗi request của user sẽ rơi vào ngẫu nhiên 1 trong 3 container. User sẽ thấy `history_length` nhảy nhót bất thường (ví dụ request 1 rơi vào A -> len=0, request 2 rơi vào B -> len=0, request 3 rơi vào A -> len=2, request 4 rơi vào C -> len=0). Agent trông như bị "mất trí nhớ" do state không được chia sẻ giữa các instance.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải khi deploy: `Error: ConnectionRefusedError / REDIS_URL invalid`.
Nguyên nhân: Môi trường cloud mặc định dùng `REDIS_URL=redis://localhost:6379/0` làm agent không kết nối được tới service Redis container/add-on riêng biệt.
Cách sửa: Thêm biến môi trường `REDIS_URL=redis://redis:6379/0` (hoặc URL của Redis Addon trên Railway/Render) trong phần Environment Variables của dashboard platform.
