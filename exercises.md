# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bên dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Phi Nhật               Mã học viên: 2A202602658

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `agent_api_key="changeme"`, khi deploy lên môi trường Cloud/Production mà quên thiết lập biến môi trường `AGENT_API_KEY` trong dashboard, ứng dụng vẫn sẽ khởi động bình thường mà không hề có cảnh báo. Khi đó, các bot quét Internet có thể dễ dàng dò ra key mặc định `"changeme"` và thoải mái gửi request để tiêu thụ tài nguyên LLM, dẫn đến rò rỉ dịch vụ và hóa đơn API tăng đột biến mà ta không hề hay biết cho tới cuối tháng. Việc không đặt giá trị mặc định buộc Pydantic ném lỗi `ValidationError` và làm app dừng hoạt động ngay lập tức lúc khởi động (fail-fast), giúp nhà phát triển phát hiện và cấu hình bổ sung secret ngay trong quá trình deploy thay vì để lọt lỗ hổng nghiêm trọng ra ngoài internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")` không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:56:01.458291+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00002745}`

Hai việc làm được với dòng log JSON này:
1. **Truy vấn và phân tích có cấu trúc (Structured Querying & Aggregation):** Các công cụ gom log tập trung (Datadog, Loki, Elasticsearch, CloudWatch) có thể tự động parse các trường JSON để lọc theo từng `user_id`, tính tổng chi phí phát sinh `cost_usd` theo từng giờ/ngày, hoặc thống kê phân phối số lượng token mà không cần viết regex phức tạp.
2. **Cảnh báo tự động dựa trên ngưỡng (Automated Metric Alerting):** Có thể thiết lập alerting rule tự động kích hoạt (trigger alert) khi phát hiện một `user_id` có `cost_usd` vượt quá ngưỡng cho phép trong 5 phút hoặc khi xuất hiện log với `level: "error"`, giúp đội ngũ vận hành phản ứng kịp thời trước các sự cố rò rỉ ngân sách hoặc lỗi hệ thống.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) bao gồm toàn bộ bộ công cụ biên dịch (build toolchain) như gcc, g++, make, các file C headers cần thiết cho quá trình build package, cùng với các tiện ích hệ thống đầy đủ, tài liệu man pages và cache của trình quản lý gói apt trong base image `python:3.11` tiêu chuẩn. Ở bản Multi-stage, stage builder chịu trách nhiệm tải và biên dịch dependencies vào thư mục trung gian `/install`, sau đó stage runtime chỉ sử dụng `python:3.11-slim` gọn nhẹ và copy phần thư viện thành phẩm sang `/usr/local`, loại bỏ hoàn toàn các trình biên dịch và file rác không cần thiết lúc chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại:
- Các layer từ đầu cho tới trước `COPY app ./app` (bao gồm base image, tạo appuser, copy thư viện từ builder stage, `COPY requirements.txt`) đều không bị ảnh hưởng và được lấy lại nguyên vẹn từ Docker cache (`CACHED`).
- Chỉ có layer `COPY app ./app` và các layer kế tiếp sau nó (`COPY utils ./utils`, `USER appuser`, `CMD`) bị mất cache và phải thực thi lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi có bất kỳ thay đổi nào trong mã nguồn (dù chỉ là 1 ký tự), Docker sẽ đánh dấu layer `COPY . .` bị thay đổi và hủy bỏ cache của toàn bộ các bước phía sau. Hệ quả là lệnh `RUN pip install` sẽ bị chạy lại từ đầu mỗi lần build, khiến quá trình build mất hàng phút tải lại dependencies thay vì chỉ mất 1-2 giây để copy code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công:
1. Kẻ tấn công khai thác lỗ hổng Remote Code Execution (RCE) trong ứng dụng Python (như deserialize pickle không an toàn, command injection qua `os.system`).
2. Mã độc được thực thi dưới quyền của tiến trình trong container. Nếu không đặt `USER`, tiến trình này sở hữu quyền `root` (UID 0) bên trong container namespace.
3. Kẻ tấn công tận dụng quyền root này để khai thác các lỗ hổng nhân Linux (kernel exploit), container runtime (như runc CVE-2019-5736, CVE-2024-21626) hoặc truy cập các socket/volume được mount từ host (như `/var/run/docker.sock`) để thoát khỏi container (container breakout).
4. Vì UID bên trong container ánh xạ trực tiếp tới UID 0 trên host, kẻ tấn công lập tức chiếm được quyền root trên toàn bộ máy host.

Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại Bước 2: Kẻ tấn công chỉ có quyền của một user không đặc quyền (`UID 10001`), không có quyền ghi vào các file nhị phân hệ thống, không có các Linux capabilities đặc quyền (`CAP_SYS_ADMIN`, `CAP_NET_RAW`), từ đó ngăn chặn hầu hết các vector tấn công leo thang và thoát container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Một người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp khi dùng fixed window (đếm theo phút đồng hồ).
Cách đạt được con số đó:
- Người dùng đợi đến giây cuối cùng của phút thứ nhất, ví dụ lúc 10:00:59, và gửi liền 10 request. Vì trong phút 10:00 chưa có request nào, hệ thống tính đây là 10 request hợp lệ.
- Ngay khi đồng hồ chuyển sang giây tiếp theo (10:01:00), cửa sổ phút mới bắt đầu và bộ đếm tự động reset về 0.
- Người dùng lập tức gửi thêm 10 request nữa lúc 10:01:00. Hệ thống lại ghi nhận 10 request này thuộc phút mới và cho qua.
Tổng cộng trong khoảng thời gian từ 10:00:59 đến 10:01:00 (vỏn vẹn 2 giây), người dùng đã gửi thành công 20 request, gấp đôi hạn mức 10 req/phút của hệ thống. Sliding window loại bỏ hoàn toàn kẽ hở này nhờ tính liên tục của khoảng thời gian 60 giây trượt.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt cốt lõi:
- Rate limit kiểm soát **số lượng request theo tần suất thời gian ngắn** (velocity/frequency - ví dụ 10 req/phút) để bảo vệ server khỏi quá tải tài nguyên mạng, CPU và tấn công DoS.
- Cost guard kiểm soát **tổng số tiền chi tiêu tích lũy theo chu kỳ tài chính** (budget/cumulative cost - ví dụ $10/tháng) để bảo vệ ngân sách trước các chi phí token của LLM.

Tình huống:
1. *Rate limit cho qua nhưng Cost guard chặn:* Người dùng chỉ gửi 1 request trong phút (hoàn toàn hợp lệ theo hạn mức 10 req/phút), nhưng trong tháng đó người dùng đã tiêu hết $10.0 ngân sách (hoặc chi phí ước tính của request này vượt quá budget còn lại). Cost guard sẽ phát hiện và chặn với mã lỗi HTTP 402 Payment Required.
2. *Cost guard cho qua nhưng Rate limit chặn:* Người dùng là tài khoản mới toanh đầu tháng, số dư chi tiêu hiện tại là $0.0 (chưa hề chạm ngưỡng $10.0). Tuy nhiên, người dùng dùng script gửi liên tục 15 request chỉ trong vòng 5 giây. Rate limit sẽ chặn từ request thứ 11 với mã lỗi HTTP 429 Too Many Requests để tránh nghẽn server, mặc dù về mặt chi phí người dùng vẫn còn rất nhiều tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện diễn ra:
1. Redis gặp sự cố mạng hoặc failover và mất kết nối trong 30 giây.
2. Bộ kiểm tra sức khỏe của orchestrator gọi vào endpoint gộp trên cả 3 container agent. Do phụ thuộc vào Redis, endpoint này trả về lỗi (503 hoặc timeout) trên toàn bộ 3 container.
3. Vì xem đây là liveness probe (chỉ báo sự sống), orchestrator kết luận rằng cả 3 tiến trình container đã hỏng và lập tức gửi tín hiệu tiêu diệt / restart toàn bộ 3 container.
4. Quá trình khởi động lại diễn ra, các container mới khởi động lại tiếp tục kiểm tra Redis và vẫn thấy Redis chưa phản hồi, dẫn đến việc container bị crashloop và restart liên tục.
5. Khi Redis phục hồi sau 30 giây, cụm container vẫn đang kẹt trong trạng thái restarting/unhealthy, khiến toàn bộ hệ thống bị downtime hoàn toàn. Thay vào đó, nếu tách riêng `/ready` (readiness probe), load balancer chỉ tạm thời ngừng đẩy traffic vào container trong 30 giây mà không hề restart container, và traffic sẽ tự động thông suốt trở lại ngay khi Redis sống lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu trong một dict Python nội bộ của process (stateful in-memory):
- Do load balancer phân phối traffic luân chuyển giữa 3 container A, B, C theo cơ chế round-robin, mỗi container sẽ duy trì một bộ nhớ dict hoàn toàn độc lập trong RAM của riêng nó.
- Khi người dùng gửi liên tiếp các câu hỏi:
  - Request 1 đến container A: `history_length` = 0 (A ghi nhận câu hỏi 1).
  - Request 2 đến container B: `history_length` = 0 (B chưa từng gặp user này).
  - Request 3 đến container C: `history_length` = 0 (C cũng chưa từng gặp user này).
  - Request 4 đến container A: `history_length` = 2 (A chỉ nhớ được câu hỏi 1).
  - Request 5 đến container B: `history_length` = 2 (B chỉ nhớ được câu hỏi 2).
- Ta sẽ thấy giá trị `history_length` nhảy loạn xạ không theo trật tự tăng dần, khiến agent có triệu chứng "mất trí nhớ ngẫu nhiên". Khi dùng Redis làm kho lưu trữ trung tâm, mọi instance đều đọc ghi cùng một nguồn dữ liệu nên `history_length` luôn tăng đều đặn và chính xác: 0 -> 2 -> 4 -> 6 -> 8...

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** Trên cloud dashboard (Render/Railway), deployment báo trạng thái `Health check timeout / Container failed to respond on port 8000` hoặc trang công khai trả về lỗi HTTP 502 Bad Gateway dù build thành công.
- **Nguyên nhân:** Nền tảng cloud tự động gán một cổng động thông qua biến môi trường `PORT` (ví dụ `PORT=10000` hoặc port ngẫu nhiên) và yêu cầu ứng dụng phải lắng nghe trên cổng đó, đồng thời phải bind vào địa chỉ `0.0.0.0`. Tuy nhiên trong Dockerfile ban đầu, lệnh uvicorn cố định `--port 8000`, khiến router của nền tảng không thể kết nối tới container trên cổng mà nó chỉ định.
- **Cách sửa:** Sửa lệnh `CMD` trong Dockerfile thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để uvicorn linh hoạt nhận cổng từ biến môi trường `$PORT` do platform cấp, đồng thời khai báo `port: int = 8000` trong Pydantic `Settings` để đảm bảo tương thích hoàn toàn.
