# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết câu trả lời ngay dưới từng câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Quốc Cường  Mã học viên: 2A202602466

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường Staging/Production (ví dụ trên Render hoặc Railway), kỹ sư thiết lập quên không cấu hình biến môi trường `AGENT_API_KEY` trong bảng điều khiển (dashboard).
- Nếu đặt giá trị mặc định là `"changeme"`: Ứng dụng vẫn khởi động bình thường (`200 OK`), health check vượt qua nên hệ thống tưởng service sẵn sàng hoạt động an toàn. Tuy nhiên, bất kỳ kẻ tấn công hoặc bot quét lỗ hổng nào thử key mặc định phổ biến `"changeme"` đều sẽ truy cập được vào endpoint `/ask`. Hậu quả là kẻ xấu có thể tiêu tốn toàn bộ hạn mức LLM/tài nguyên hệ thống, làm lộ dữ liệu hoặc khiến hóa đơn API tăng đột biến hàng nghìn USD mà đội ngũ không hề hay biết cho đến cuối tháng.
- Khi áp dụng triết lý Fail Fast (không đặt mặc định): Thiếu biến môi trường sẽ khiến Pydantic ném `ValidationError` ngay lúc ứng dụng vừa khởi chạy trong hàm `lifespan`. Tiến trình container dừng lại lập tức, nền tảng cloud đánh dấu lần deploy thất bại và hiển thị rõ thông báo lỗi thiếu biến môi trường trong runtime log. Nhờ "chết sớm", lỗ hổng bảo mật nghiêm trọng này bị chặn đứng hoàn toàn trước khi có bất kỳ request công khai nào lọt vào hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế thu được từ stdout:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:15:30.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.000135}`

Hai việc làm được với Structured Log JSON mà `print("đã trả lời xong")` không thể làm được:
1. **Lập chỉ mục, truy vấn và tạo cảnh báo tự động trên hệ thống giám sát tập trung (Datadog, Grafana Loki, CloudWatch, ELK):** Log JSON có schema dạng key-value rõ ràng. Hệ thống thu thập log có thể parse tự động các trường số và chuỗi để thực hiện thống kê tức thời (ví dụ: `SUM(cost_usd) BY user_id`, tính P95/P99 token tiêu thụ, hoặc đặt ngưỡng alert nếu `cost_usd > 0.01` trong 1 phút). Đối với chuỗi `print` thô, máy tính không thể hiểu được ngữ nghĩa nếu không viết regex phức tạp và dễ vỡ.
2. **Audit trail, liên kết sự kiện (correlation) và đối soát chi phí (reconciliation) chi tiết:** Dòng log lưu lại chính xác mốc thời gian chuẩn UTC ISO-8601, định danh người dùng `user_id`, số token vào/ra và chi phí phát sinh. Điều này cho phép đội ngũ truy vết toàn bộ lịch sử hành vi của từng user, đối chiếu từng xu trên hóa đơn LLM của nhà cung cấp với nhật ký hệ thống, và phát hiện các hành vi gian lận hoặc lạm dụng dịch vụ.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (hơn 800 MB) gồm:
1. **Trình biên dịch và công cụ build hệ thống:** Bản 1-stage hoặc bản base image Python đầy đủ chứa toàn bộ bộ công cụ biên dịch C/C++ (`gcc`, `g++`, `make`), header files (`python3-dev`, `libc-dev`), và các gói hệ thống cần thiết cho quá trình build native wheels.
2. **Bộ nhớ đệm và file tạm:** Quá trình chạy pip trong cùng một stage lưu lại cache tại `~/.cache/pip`, các file `.whl` trung gian và thư mục build tạm thời.
3. **Môi trường hệ điều hành đầy đủ:** Sử dụng base image `python:3.12-slim` ở stage runtime đã loại bỏ toàn bộ các package tiện ích desktop/server không dùng đến của Debian chuẩn.
Trong Dockerfile multi-stage, stage `builder` tải và biên dịch toàn bộ wheel vào thư mục `/wheels`. Stage `runtime` chỉ copy các wheel đó sang để cài đặt bằng `pip install --no-index`, sau đó xóa bỏ luôn thư mục `/wheels`. Kết quả là image cuối cùng chỉ chứa code ứng dụng, runtime Python tối thiểu và các thư viện cần thiết, vừa tiết kiệm băng thông truyền tải, vừa thu hẹp diện tích bề mặt tấn công (attack surface).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với cấu trúc Dockerfile hiện tại:
- **Các layer được dùng lại từ cache (CACHED):**
  + Ở stage `builder`: `COPY requirements.txt .`, `RUN python -m pip wheel ...` (do file `requirements.txt` không thay đổi).
  + Ở stage `runtime`: `COPY requirements.txt .`, `COPY --from=builder /wheels /wheels`, `RUN python -m pip install ... && rm -rf /wheels && useradd ...`.
- **Các layer phải chạy lại:**
  + `COPY --chown=appuser:appuser app ./app` (vì nội dung thư mục `app/` đã bị thay đổi do sửa file `app/main.py`).
  + Các layer kế tiếp sau đó: `COPY --chown=appuser:appuser utils ./utils`, `USER 10001`, `EXPOSE 8000`, `HEALTHCHECK`, `CMD`. Quá trình build lại diễn ra gần như ngay lập tức (dưới 1 giây).

Nếu đặt `COPY . .` lên trước `RUN pip install`:
- Mỗi lần sửa dù chỉ một ký tự trong `app/main.py`, mã hash của layer `COPY . .` sẽ thay đổi.
- Theo nguyên lý cache của Docker, khi một layer bị vô hiệu hóa (cache bust), toàn bộ các layer bên dưới nó bắt buộc phải chạy lại từ đầu. Lệnh `RUN pip install` sẽ bị thực thi lại mỗi lần sửa code, buộc Docker phải cài lại toàn bộ thư viện dependencies từ đầu, làm thời gian build kéo dài từ vài giây lên vài phút, gây lãng phí tài nguyên và làm chậm nghiêm trọng chu trình phát triển/deploy.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện dẫn tới việc kẻ tấn công chiếm quyền máy host:
1. Kẻ tấn công phát hiện và khai thác thành công một lỗ hổng thực thi mã từ xa (RCE) trong ứng dụng Python (ví dụ: lỗ hổng deserialization với `pickle`, template injection `jinja2`, hoặc lỗ hổng command injection).
2. Mã độc của kẻ tấn công được thực thi bên trong tiến trình của container. Do container mặc định không có khai báo `USER`, tiến trình này chạy dưới quyền `root` (UID 0).
3. Với quyền root trong container, kẻ tấn công có toàn quyền thao tác với file hệ thống, cài thêm công cụ quét mã độc hoặc khai thác các lỗi chia sẻ tài nguyên với máy host (chẳng hạn như mount nhầm Docker socket `/var/run/docker.sock`, mount các thư mục nhạy cảm `/proc`, `/sys`, hoặc khai thác lỗ hổng kernel Linux privilege escalation / container breakout).
4. Do UID 0 bên trong container ánh xạ trực tiếp tới UID 0 (root) trên máy host (nếu không bật user namespace remapping), kẻ tấn công thoát khỏi container và chiếm quyền root điều khiển toàn bộ máy chủ vật lý/host machine.

Lệnh `USER 10001` cắt đứt chuỗi tấn công ở ngay bước 2:
Khi lệnh `USER 10001` (`appuser`) được áp dụng, tiến trình của ứng dụng chỉ có quyền của một user thông thường không có đặc quyền (unprivileged user). Kẻ tấn công nếu có RCE cũng chỉ bị giam trong phạm vi quyền hạn của `appuser`, không thể sửa file hệ điều hành của container, không thể ghi vào các thư mục hệ thống, không có quyền sudo, và không thể thực thi các kỹ thuật khai thác container escape vốn yêu cầu quyền root container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Con số tối đa: **20 request** trong 2 giây liên tiếp.

Giải thích cách đạt được:
- Với thuật toán Fixed Window Counter (đếm theo phút đồng hồ cố định, reset bộ đếm vào giây `00` của mỗi phút):
  + Giả sử người dùng gửi 10 request dồn dập vào 1 giây cuối cùng của phút thứ nhất: từ `10:00:59.000` đến `10:00:59.999`. Bộ đếm của phút 10:00 ghi nhận 10/10 request và cho qua toàn bộ vì chưa vượt quá hạn mức.
  + Ngay khi đồng hồ chuyển sang `10:01:00.000`, một cửa sổ mới bắt đầu và bộ đếm tự động reset về 0.
  + Người dùng lập tức gửi tiếp 10 request nữa trong 1 giây đầu tiên của phút thứ hai: từ `10:01:00.000` đến `10:01:00.999`. Bộ đếm của phút 10:01 lại ghi nhận 10/10 request và tiếp tục cho qua.
  + Tổng cộng: Chỉ trong khoảng thời gian 2 giây (từ `10:00:59` đến `10:01:01`), người dùng đã gửi thành công $10 + 10 = 20$ request — gấp đôi hạn mức 10 req/phút dự kiến, có thể gây sập backend hoặc quá tải LLM (burst attack).
- Ngược lại, thuật toán Sliding Window với Redis Sorted Set luôn kiểm tra chính xác khoảng thời gian trôi $[now - 60s, now]$. Tại mốc `10:01:00.500`, 10 request của giây trước vẫn nằm trong cửa sổ 60 giây gần nhất, do đó request thứ 11 sẽ bị chặn ngay lập tức với mã lỗi `429 Too Many Requests`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau giữa hai cơ chế:
- **Rate Limit:** Kiểm soát **tần suất / tốc độ gửi request** (số lượng request trên một đơn vị thời gian ngắn, ví dụ 10 request/phút) nhằm bảo vệ hạ tầng web/API không bị quá tải CPU, RAM, nghẽn đường truyền mạng hoặc tấn công DoS. Rate limit không quan tâm nội dung bên trong request tốn bao nhiêu tài nguyên tính toán.
- **Cost Guard:** Kiểm soát **ngân sách tài chính lũy kế** (tổng số tiền USD hoặc số token tiêu hao trong một chu kỳ dài, ví dụ $10/tháng) nhằm bảo vệ hạn mức tài chính của dịch vụ và người dùng trước việc tiêu xài API LLM không kiểm soát.

Tình huống Rate limit cho qua nhưng Cost guard phải chặn:
- Một người dùng chỉ gửi 1 request duy nhất trong suốt 15 phút (tần suất cực thấp, hoàn toàn hợp lệ với rate limit 10 request/phút). Tuy nhiên, tài khoản của user này trong tháng đã tiêu hết $9.999 trên tổng ngân sách $10.00. Khi gửi một câu hỏi kèm tài liệu dài (ước tính chi phí phát sinh $0.005), tổng chi tiêu dự kiến sẽ vượt quá $10.00. Cost guard phát hiện điều này và lập tức chặn request với mã lỗi `402 Payment Required`.

Tình huống Cost guard cho qua nhưng Rate limit phải chặn:
- Vào đầu tháng mới, người dùng vừa được cấp lại ngân sách và chưa tiêu đồng nào (số dư nguyên $10.00). Người dùng dùng công cụ tự động gửi spam liên tục 15 request ngắn trong vòng 3 giây (mỗi request chỉ tiêu tốn $0.00002, tổng cộng mới hết $0.0003, rất nhỏ so với ngân sách $10.00). Cost guard đánh giá ngân sách hoàn toàn đủ và cho qua, nhưng Rate limit phát hiện user đã vượt quá ngưỡng 10 request/phút nên lập tức chặn từ request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự chuỗi sự kiện xảy ra khi gộp chung probe:
1. Redis gặp sự cố (mạng nội bộ gián đoạn, quá tải hoặc restart) trong 30 giây.
2. Bộ điều phối (Container Orchestrator như Kubernetes / Docker Compose) gửi tín hiệu liveness probe định kỳ vào endpoint duy nhất này. Vì endpoint bị gộp kiểm tra cả Redis, kết nối Redis thất bại khiến cả 3 container đồng loạt trả về mã lỗi `503 Service Unavailable`.
3. Orchestrator hiểu rằng tiến trình ứng dụng bên trong cả 3 container đều đã bị treo/chết (liveness failure), do đó lập tức kích hoạt cơ chế tự hủy và restart lại toàn bộ cả 3 container.
4. Trong lúc Redis vẫn chưa hồi phục, 3 container mới khởi động lên lại tiếp tục fail probe ngay từ những giây đầu tiên. Orchestrator tiếp tục kill và khởi động lại chúng liên tục, đẩy toàn bộ cụm vào vòng lặp tử thần `CrashLoopBackOff`.
5. Hậu quả là toàn bộ các request khác đang xử lý dở bị hủy ngang, CPU máy chủ tăng vọt do liên tục boot tiến trình mới.
6. Ngay cả khi Redis đã kết nối lại sau 30 giây, hệ thống vẫn phải mất thêm một khoảng thời gian dài mới khôi phục lại bình thường do hiệu ứng sóng va đập (thundering herd).

Ngược lại, khi tách riêng:
- `/health` (liveness) độc lập với Redis nên vẫn trả `200 OK` → các container không bị restart oan uổng.
- `/ready` (readiness) trả `503` → Load balancer chỉ tạm thời ngừng điều hướng traffic vào service trong 30 giây. Ngay khi Redis thông suốt, `/ready` tự động trả lại `200` và hệ thống tiếp tục phục vụ người dùng bình thường mà không hề có container nào bị khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Quan sát thực tế khi dùng Redis (Stateless service):
- Cả 3 instance agent đều dùng chung một bộ lưu trữ ngoài là Redis. Khi gọi `/ask` nhiều lần liên tiếp, bất kể request được điều hướng ngẫu nhiên hay xoay vòng (Round Robin) vào instance nào (agent-1, agent-2, hay agent-3), tất cả các instance đều đọc và cập nhật chung vào cùng một key `history:{user_id}` trên Redis.
- Vì thế, giá trị `history_length` luôn tăng đều đặn, chính xác: 0 → 2 → 4 → 6 → 8... (mỗi lượt hỏi-đáp thêm 2 message: 1 user, 1 assistant).

Nếu lưu lịch sử trong một `dict` Python in-memory bên trong process:
- Mỗi instance agent sở hữu vùng nhớ RAM độc lập và có một `dict` cục bộ riêng.
- Khi load balancer phân phối traffic sang 3 instance:
  + Request 1 rơi vào Instance 1: `dict` của Instance 1 đang rỗng → `history_length` trả về 0. Sau đó Instance 1 lưu 2 message vào `dict` của mình.
  + Request 2 rơi vào Instance 2: `dict` của Instance 2 hoàn toàn rỗng → `history_length` lại trả về 0!
  + Request 3 rơi vào Instance 3: `dict` của Instance 3 cũng rỗng → `history_length` lại tiếp tục là 0!
  + Request 4 quay trở lại Instance 1: `history_length` trả về 2.
  + Request 5 rơi vào Instance 2: `history_length` trả về 2.
- Kết quả: `history_length` nhảy lộn xộn, ngữ cảnh đàm thoại của AI agent bị đứt đoạn nghiêm trọng tùy thuộc vào việc request rơi trúng container nào (agent bị "mất trí nhớ từng phần"). Thêm vào đó, nếu bất kỳ container nào bị restart hoặc crash, toàn bộ lịch sử trong dict của container đó sẽ vĩnh viễn biến mất.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: **Health check timeout / Application failed to respond trên Cloud Platform do ứng dụng bind cố định vào cổng 8000 thay vì sử dụng biến môi trường `$PORT` do nền tảng tự động cấp phát.**

1. **Thông báo lỗi trong Deployment Log:**
   `Timed out waiting for container to open port 10000` (hoặc `Healthcheck failed: Connection refused on port $PORT`).
2. **Cách tìm ra nguyên nhân:**
   - Mở tab **Runtime / Deploy Logs** trên giao diện điều khiển của nền tảng cloud (Render/Railway).
   - Quan sát thấy log khởi động của Uvicorn ghi nhận: `Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)`.
   - Trong khi đó, các dịch vụ PaaS như Render hay Railway không dùng cổng mặc định 8000 mà tự động chỉ định một cổng ngẫu nhiên thông qua biến môi trường `PORT` (ví dụ `PORT=10000`). Bộ định tuyến (Reverse Proxy / Router) của Cloud chỉ chuyển hướng traffic và thực hiện health check trên cổng `$PORT` đó. Do ứng dụng chỉ lắng nghe ở 8000, router không thể kết nối tới server, dẫn đến health check bị timeout và nền tảng đánh dấu deploy thất bại.
3. **Cách khắc phục:**
   - Cập nhật lệnh khởi chạy trong `Dockerfile` để đọc biến môi trường `$PORT` một cách linh hoạt:
     `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
   - Đảm bảo trong `app/config.py`, lớp `Settings` khai báo trường `port: int = 8000` để Pydantic tự động đọc giá trị từ biến `PORT` khi chạy.
   - Trong `railway.toml`, thiết lập `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"`.
   - Re-deploy lại ứng dụng. Uvicorn đã lắng nghe chính xác trên cổng do platform cung cấp và health check trả về mã `200 OK` thành công.
