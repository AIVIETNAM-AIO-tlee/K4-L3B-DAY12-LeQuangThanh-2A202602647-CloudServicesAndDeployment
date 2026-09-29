# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `*Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Quang Thành Mã học viên: 2A202602647

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên môi trường Production/Staging, lập trình viên có thể quên cấu hình biến môi trường `AGENT_API_KEY` trên dashboard quản lý. Nếu để giá trị mặc định là `"changeme"`, service vẫn âm thầm khởi động bình thường; kẻ tấn công hoặc bot quét Internet có thể dễ dàng đoán được key mặc định này để gửi hàng loạt request gọi LLM, gây cạn kiệt ngân sách hoặc lộ dữ liệu trước khi bạn kịp nhận ra. Ngược lại, việc "chết sớm" (Fail Fast) với lỗi `ValidationError` ngay lúc khởi động sẽ khiến quá trình deploy báo thất bại lập tức, buộc lập trình viên phải cấu hình secret hợp lệ trước khi hệ thống mở cửa tiếp nhận traffic từ bên ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:35:55.906785+00:00", "user_id": "sv01", "tokens_in": 14, "tokens_out": 32, "cost_usd": 0.00018}
```

Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn, lọc và tổng hợp số liệu tự động (Structured Aggregation):** Các hệ thống thu thập log (như Grafana Loki, Datadog, CloudWatch) có thể parse trực tiếp các trường JSON để tính toán tổng chi phí theo ngày (`SUM(cost_usd)`), phân tích số lượng token tiêu thụ theo từng `user_id`, hoặc thống kê phân vị độ trễ.
2. **Thiết lập cảnh báo tự động và kiểm toán truy vết (Alerting & Auditing):** Có thể cài đặt rule cảnh báo ngay lập tức nếu một request tiêu thụ token/chi phí bất thường vượt ngưỡng, đồng thời trường `timestamp` chuẩn UTC và `user_id` cung cấp bằng chứng kiểm toán chính xác để truy vết hành vi người dùng khi xảy ra sự cố.

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
| 1 stage (bản đầu) | ~885 MB |
| Multi-stage | ~195 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch gần 700 MB bao gồm:
1. Các công cụ và phần mềm biên dịch (`gcc`, `g++`, `make`, `build-essential`, `python3-dev`) cần thiết trong quá trình biên dịch các thư viện C-extension ở stage builder.
2. Thư mục cache của pip (`~/.cache/pip`) lưu trữ các file nén `.whl` và tarball được tải về.
3. Các file header C, tài liệu hướng dẫn (manpages), và các package hệ điều hành phụ trợ chỉ dùng trong lúc build nhưng hoàn toàn không cần thiết cho quá trình runtime. Stage `runtime` chỉ copy thư mục dependency hoàn chỉnh (`/install`) sang base image `python:3.11-slim`, giúp loại bỏ hoàn toàn rác thải build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với thứ tự tối ưu hiện tại:** Các layer từ đầu cho tới `COPY requirements.txt .` và `RUN pip install ...` đều được **dùng lại hoàn toàn từ cache** (`CACHED`) vì file `requirements.txt` không hề thay đổi. Docker chỉ bắt đầu thực thi lại từ layer `COPY app ./app` trở đi, thời gian build lại chỉ mất 1-2 giây.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:** Khi sửa dù chỉ một ký tự trong `app/main.py`, layer `COPY . .` sẽ bị thay đổi checksum, làm vỡ cache (cache invalidation) của Docker. Toàn bộ các layer phía sau nó bao gồm `RUN pip install` sẽ bị buộc phải chạy lại từ đầu, khiến Docker phải tải và cài lại toàn bộ danh sách thư viện mỗi lần sửa code, làm chậm nghiêm trọng quy trình phát triển và CI/CD.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện khi chạy bằng root:**
  1. Ứng dụng Python tồn tại một lỗ hổng bảo mật (ví dụ: Command Injection, Remote Code Execution qua deserialization hoặc dependency có lỗ hổng).
  2. Kẻ tấn công gửi payload khai thác thành công và chiếm quyền điều khiển tiến trình Python. Vì tiến trình này chạy dưới user `root` (UID 0), kẻ tấn công sở hữu quyền root bên trong container.
  3. Từ quyền root trong container, kẻ tấn công khai thác tiếp các lỗ hổng container breakout (như khai thác kernel host chưa vá, lạm dụng Linux capabilities chưa bị drop, hoặc truy cập file nhạy cảm nếu có mount volume từ host). Do UID 0 trong container mặc định ánh xạ trực tiếp tới UID 0 (root) của host, kẻ tấn công chiếm toàn quyền kiểm soát máy chủ host.
- **Lệnh `USER appuser` cắt đứt chuỗi ở:** Ngay tại **bước 2**. Khi tiến trình chạy dưới một user phi đặc quyền (non-root, ví dụ UID 10001), kẻ tấn công sau khi khai thác RCE chỉ có quyền hạn cực kỳ hạn chế: không thể sửa đổi file hệ thống, không thể cài phần mềm, không thể truy cập socket của Docker daemon, và không có các Linux capabilities đặc quyền để thực hiện breakout ra host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong vòng 2 giây liên tiếp.
Cách đạt được:
- Vào giây `10:00:59` (giây cuối cùng của phút thứ nhất), người dùng gửi liên tiếp **10 request**. Hệ thống kiểm tra phút 10:00 thấy mới có 10 request nên cho qua toàn bộ.
- Ngay ở giây tiếp theo `10:01:00` (bắt đầu phút mới), bộ đếm cố định bị reset về 0. Người dùng lập tức gửi tiếp **10 request** nữa, và hệ thống vẫn chấp nhận vì quota phút 10:01 vừa được làm mới.
- Kết quả: Trong khoảng thời gian từ `10:00:59` đến `10:01:01` (chỉ vỏn vẹn 2 giây), người dùng đã gửi thành công 20 request mà không bị chặn, gây đột biến tải (traffic spike). Sliding window (cửa sổ trượt) giải quyết triệt để lỗ hổng này bằng cách luôn tính tổng số request trong 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau:**
  - **Rate Limit:** Giới hạn **số lượng request (throughput)** trong một đơn vị thời gian ngắn (ví dụ: 10 request/phút) để ngăn chặn tấn công DoS/spam và bảo vệ tính sẵn sàng của hạ tầng máy chủ.
  - **Cost Guard:** Giới hạn **tổng chi phí tài chính (financial budget)** trong một chu kỳ dài hơn (ví dụ: $10/tháng) dựa trên lượng token LLM thực tế tiêu thụ để tránh thâm hụt ngân sách API.
- **Tình huống Rate limit cho qua nhưng Cost guard chặn:** Người dùng chỉ gửi duy nhất 1 request trong phút (hoàn toàn hợp lệ theo rate limit 10 req/phút), nhưng request này kèm tài liệu prompt khổng lồ tiêu tốn 100.000 token, chi phí ước tính vượt quá số dư ngân sách còn lại trong tháng của user. Cost guard sẽ chặn ngay lập tức với mã `402 Payment Required`.
- **Tình huống Cost guard cho qua nhưng Rate limit chặn:** Đầu tháng, người dùng chưa tiêu đồng nào (ngân sách còn nguyên $10.0). Người dùng dùng script spam 15 request "Xin chào" siêu ngắn trong 3 giây. Tổng chi phí chỉ vài phần nghìn cent (chưa thấm vào ngân sách), nhưng tần suất gọi quá nhanh vi phạm hạn mức 10 req/phút nên Rate limit chặn với mã `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. **Giây 0 – 5:** Redis gặp sự cố mạng hoặc khởi động lại, tạm thời không phản hồi ping trong 30 giây.
2. **Giây 5 – 10:** Orchestrator (Kubernetes/Docker) định kỳ thăm dò liveness probe (`/health`). Vì endpoint này gộp chung việc kiểm tra Redis nên lập tức trả về lỗi hoặc timeout trên cả 3 container agent.
3. **Giây 10 – 20:** Liveness probe thất bại liên tiếp vượt quá số lần retry cho phép. Orchestrator suy đoán sai lầm rằng các container ứng dụng đã bị treo (deadlock), do đó ra lệnh **kill và restart toàn bộ cả 3 container agent cùng một lúc**.
4. **Giây 20 – 30:** Các container agent rơi vào vòng lặp CrashLoopBackOff: khởi động lên -> kiểm tra Redis thất bại -> bị kill và restart lại liên tục.
5. **Giây 30 trở đi:** Dù Redis đã hoàn toàn hồi phục, cụm agent vẫn đang trong quá trình restart lộn xộn, dẫn đến toàn bộ hệ thống bị gián đoạn dịch vụ hoàn toàn (outage 100%), biến một lỗi tạm thời ở tầng phụ thuộc thành sự cố sập cả hệ thống. Trong khi đó, nếu tách bạch, `/ready` sẽ chỉ tạm ngưng đẩy traffic vào agent mà không hề restart container, giữ cho hệ thống tự phục hồi ngay khi Redis sống lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi lưu trên Redis (Stateless - chuẩn hiện tại):** Mọi instance agent đều đọc/ghi vào cùng một cụm Redis tập trung, do đó dù request được Load Balancer điều phối vào bất kỳ container nào (agent-1, agent-2 hay agent-3), `history_length` luôn **tăng dần đều đặn** sau mỗi lượt tương tác (0 -> 2 -> 4 -> 6 -> ...).
- **Nếu lưu trong dict Python (Stateful - trong bộ nhớ RAM của từng tiến trình):** Vì mỗi container là một process độc lập với vùng nhớ RAM riêng biệt, các request gửi tới sẽ luân chuyển round-robin qua các container khác nhau. Người dùng sẽ thấy `history_length` nhảy **thất thường, ngắt quãng và không đồng nhất** (ví dụ câu 1 vào agent-1 có history=0, câu 2 vào agent-2 có history=0, câu 3 vào agent-3 có history=0, câu 4 quay lại agent-1 mới thấy history=2). Agent sẽ bị "mất trí nhớ ngẫu nhiên", làm vỡ hoàn toàn ngữ cảnh cuộc hội thoại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Health check timeout / Service không nhận traffic sau khi deploy lên Cloud (Railway/Render).
- **Thông báo lỗi:** `Application failed to respond on port 8000` hoặc `Deployment failed: Healthcheck timed out after 30s`.
- **Cách tìm ra nguyên nhân:** Kiểm tra log container (`railway logs`) và cấu hình mạng của nền tảng cloud. Nền tảng cloud không mở cổng cố định 8000 mà tự động cấp phát một cổng ngẫu nhiên và truyền vào ứng dụng qua biến môi trường `$PORT`. Do Dockerfile ban đầu fix cứng lệnh chạy trên port 8000 (`uvicorn ... --port 8000`), container lắng nghe ở port 8000 trong khi Cloud Router lại thăm dò liveness ở port `$PORT`, dẫn đến kết nối bị từ chối và timeout.
- **Cách sửa:** Cập nhật lệnh CMD trong Dockerfile để đọc linh hoạt biến môi trường `$PORT` với fallback:
  ```dockerfile
  CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
  ```
  Đồng thời trong `app/config.py`, dùng pydantic-settings để tự động ánh xạ biến môi trường `PORT` vào cấu hình của ứng dụng.
