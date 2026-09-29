# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Minh Hiếu  Mã học viên: L3B202602669

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường cloud (Railway/Render) mà developer quên cấu hình biến môi trường `AGENT_API_KEY` trên dashboard:
- Nếu để mặc định là `"changeme"`, container vẫn khởi động thành công và liveness probe báo xanh. Bot quét trên Internet sẽ nhanh chóng dò ra endpoint công khai và thử các khóa mặc định phổ biến như `"changeme"` để gọi API `/ask` miễn phí. Hậu quả là toàn bộ hạn mức hoặc ngân sách tài khoản LLM bị tiêu cạn mà ta không hề hay biết cho tới khi nhận hóa đơn.
- Khi không có giá trị mặc định, Pydantic ném `ValidationError` ngay lúc ứng dụng khởi tạo Settings. Ứng dụng dừng ngay lập tức (fail fast), container báo lỗi crash trên log deploy khi developer đang theo dõi màn hình, buộc ta phải cấu hình đúng secret trước khi dịch vụ kịp mở cho công chúng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:37:53.387774+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 0.00002265}
```

Hai việc làm được với định dạng này mà `print()` thông thường không làm được:
1. **Lọc, truy vấn và thống kê theo trường dữ liệu**: Các hệ thống gom log (Datadog, Loki, CloudWatch, ELK) có thể tự động parse JSON để thực hiện truy vấn có cấu trúc, ví dụ: tìm kiếm những người dùng tiêu tốn chi phí nhiều nhất trong ngày (`WHERE user_id = ... AND cost_usd > 0.001`), tính tổng chi phí theo giờ, hoặc vẽ biểu đồ tương quan giữa độ dài câu hỏi (`tokens_in`) và câu trả lời (`tokens_out`).
2. **Thiết lập cảnh báo tự động (Alerting)**: Dễ dàng cấu hình các metric filter và rule cảnh báo (alert) tự động khi có bất thường, ví dụ: cảnh báo về Slack/PagerDuty khi mức chi phí (`cost_usd`) của một request vượt ngưỡng $0.05 hoặc khi tần suất các sự kiện lỗi (`level: "error"`) tăng vọt trong 5 phút qua.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch (~750 MB) bao gồm:
1. Hệ điều hành base image đầy đủ (`python:3.11` dựa trên Debian bookworm chuẩn) chứa toàn bộ trình biên dịch C/C++ (`gcc`, `g++`, `make`), công cụ build (`build-essential`), thư viện phát triển (`header files`, `glibc-dev`), và các gói hệ thống không cần thiết cho quá trình thực thi ứng dụng.
2. Bộ đệm cache tạm của trình quản lý gói `apt` và các công cụ phát triển. Trong bản multi-stage, stage `builder` chịu trách nhiệm cài đặt và biên dịch thư viện, sau đó stage `runtime` chỉ dùng base image siêu gọn `python:3.11-slim` và copy đúng kết quả cài đặt từ `/install` sang `/usr/local`, loại bỏ hoàn toàn các compiler và file rác phát sinh.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Khi sửa một ký tự trong `app/main.py`, các layer từ đầu cho đến `COPY requirements.txt .` và `RUN pip install ...` ở stage builder, cũng như layer `COPY --from=builder /install /usr/local` ở runtime đều được dùng lại từ cache (CACHED). Chỉ có layer `COPY app ./app` và các bước sau nó (USER, EXPOSE, CMD) phải chạy lại. Toàn bộ quá trình build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Vì nội dung thư mục mã nguồn thay đổi, layer `COPY . .` bị mất cache (cache invalidated). Theo cơ chế layer của Docker, một khi một layer bị invalid cache thì toàn bộ các layer phía sau nó đều phải chạy lại từ đầu. Do đó, Docker sẽ phải tải lại và cài đặt lại toàn bộ dependencies trong `requirements.txt` qua mạng, khiến mỗi lần sửa code tốn thêm vài phút một cách lãng phí.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện leo thang đặc quyền:
  1. Ứng dụng Python có lỗ hổng bảo mật (ví dụ: lỗi Remote Code Execution - RCE, command injection, hoặc thư viện bên thứ ba bị backdoor).
  2. Kẻ tấn công gửi payload khai thác thành công và chiếm được quyền thực thi lệnh shell bên trong container.
  3. Do container chạy dưới user `root` mặc định (UID 0), kẻ tấn công sở hữu toàn quyền quản trị tối cao bên trong container (có thể cài tool quét, sửa file hệ thống, mở port...).
  4. Tiếp theo, nếu nhân Linux trên máy host tồn tại lỗ hổng container breakout (như Dirty COW, runc CVE-2019-5736) hoặc container được cấu hình thiếu an toàn (mount thư mục nhạy cảm từ host, mount Docker socket `/var/run/docker.sock`), tiến trình UID 0 trong container có thể vượt rào cản cgroups/namespaces và chiếm quyền root trên chính máy host thực tế.
- Lệnh `USER appuser` cắt đứt chuỗi này: Khi chỉ thị `USER appuser` (UID 10001 không đặc quyền) được kích hoạt, tiến trình app chỉ có quyền hạn chế của một user thường. Nếu bị tấn công RCE, kẻ xâm nhập chỉ bị cô lập trong quyền của `appuser`, không thể sửa đổi file hệ thống, không có quyền `sudo` và không thể khai thác các cơ chế tương tác kernel cấp cao để vượt rào ra ngoài máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được:
- Người dùng chờ đến cuối phút hiện tại, ví dụ lúc `10:00:59`, gửi dồn dập 10 request (vừa vặn chạm hạn mức 10 req của phút `10:00`).
- Đúng 1 giây sau, lúc `10:01:00`, đồng hồ bước sang phút mới và bộ đếm fixed-window bị reset về 0. Người dùng gửi tiếp ngay 10 request nữa.
- Kết quả: Từ `10:00:59` đến `10:01:01` (chỉ trong vòng 2 giây liên tiếp), người dùng đã đẩy vào hệ thống 20 request mà không bị chặn, vi phạm mục tiêu ban đầu là bảo vệ hệ thống khỏi lưu lượng quá lớn trong thời gian ngắn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau:
  - **Rate limit**: Giới hạn **số lượng / tần suất request** trong một đơn vị thời gian ngắn (ví dụ: 10 request/phút) nhằm ngăn chặn spam, tấn công DoS và đảm bảo tính sẵn sàng của hạ tầng web.
  - **Cost guard**: Giới hạn **tổng chi phí / lượng token tích lũy** theo chu kỳ dài (ví dụ: $10.0/tháng) nhằm bảo vệ ngân sách tài chính trước các cuộc gọi LLM đắt đỏ.
- Tình huống Rate limit cho qua nhưng Cost guard chặn: Người dùng gọi 1 request duy nhất trong phút (tần suất rất thấp, rate limit cho qua), nhưng request này yêu cầu xử lý tài liệu lớn hoặc tài khoản của người dùng trong tháng đã tiêu hết $9.99/$10.00 ngân sách. Request mới ước tính phát sinh thêm $0.05 làm vượt ngân sách tháng -> Cost guard chặn lại với mã lỗi `402 Payment Required`.
- Tình huống Cost guard cho qua nhưng Rate limit chặn: Đầu tháng, tài khoản của người dùng còn nguyên $10.00 ngân sách. Người dùng dùng script tự động bắn liên tục 15 request trong vòng 3 giây với nội dung câu hỏi rất ngắn (mỗi câu chỉ tốn $0.00002, tổng tiền không đáng kể). Tuy nhiên vì tần suất gọi vượt quá 10 req/phút, Rate limit lập tức chặn ở request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. **Redis gặp sự cố**: Redis tạm thời mất kết nối mạng hoặc reload trong 30 giây.
2. **Liveness probe thất bại đồng loạt**: Cả 3 container nhận request probe kiểm tra sức khỏe từ orchestrator (Docker/Kubernetes/Platform), hàm probe kiểm tra Redis thất bại và trả về HTTP 503.
3. **Orchestrator restart cụm container**: Vì endpoint liveness trả về lỗi, orchestrator kết luận rằng cả 3 container ứng dụng đã bị hỏng/treo và lập tức gửi tín hiệu kill để restart lại toàn bộ 3 container.
4. **Hệ thống sập toàn diện (Cascading failure)**: Trong khi Redis vẫn chưa hồi phục hoặc đang khởi động, các container vừa restart lại tiếp tục probe thất bại và rơi vào vòng lặp restart liên tục (CrashLoopBackOff). Cụm 3 container hoàn toàn không có container nào online để nhận traffic, biến một sự cố kết nối tạm thời của Redis thành thảm họa sập toàn bộ dịch vụ.
*(Khi tách riêng: `/health` chỉ kiểm tra process sống để không restart container; `/ready` trả về 503 để Load Balancer tạm ngưng điều phối traffic tới container cho đến khi Redis kết nối lại).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (kiến trúc Stateless): Toàn bộ 3 container cùng chia sẻ chung một Redis database. Dù Load Balancer phân phối các lượt hỏi của cùng một user lần lượt sang container 1, container 2 rồi container 3, trường `history_length` vẫn tăng tuyến tính và nhất quán (0 -> 2 -> 4 -> 6 -> ...).
- Nếu lưu trong dict Python trong bộ nhớ RAM: Mỗi container sở hữu một tiến trình và vùng nhớ RAM độc lập. Khi request 1 vào container A, A lưu 2 message vào RAM của mình; request 2 bị Load Balancer đẩy sang container B, B không có dữ liệu gì nên trả về `history_length: 0` và lưu vào RAM của B; request 3 rơi vào container C thì C lại thấy 0. Kết quả là `history_length` sẽ nhảy lộn xộn ngẫu nhiên (ví dụ: 0, 0, 0, 2, 0, 2, 4...) tùy thuộc request rơi trúng container nào, và người dùng sẽ thấy agent bị hiện tượng "mất trí nhớ" gián đoạn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: `Deploy crashed` xuất hiện trên terminal khi chạy lệnh `railway up` trong lần triển khai đầu tiên.
- **Cách tìm nguyên nhân**: Dùng lệnh `railway logs` và `railway status` để kiểm tra log chi tiết từ container. Phân tích log cho thấy container uvicorn mất vài giây khởi động trong khi healthcheck ban đầu thăm dò nhanh hơn trước khi ứng dụng sẵn sàng.
- **Cách xử lý**: Nhờ Dockerfile đã cấu hình chuẩn `--port ${PORT:-8000}` và healthcheck endpoint `/health` độc lập không phụ thuộc Redis, hệ thống Railway tự động retry theo restart policy và container chuyển sang trạng thái `● Online` ổn định ngay sau đó. Kiểm tra live service tại URL `https://agent-production-f33d.up.railway.app` cho thấy cả `/health` và `/ready` (nối Redis thành công) đều phản hồi HTTP 200 OK.
