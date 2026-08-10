# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Trần Trung Dũng  Mã học viên: 2A202601785

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Khi deploy lên Railway, nếu quên truyền biến `AGENT_API_KEY`, app sẽ bị crash ngay từ lúc start và bị đánh dấu là "deploy failed". Mình sẽ phát hiện ngay lập tức để sửa. Ngược lại, nếu để mặc định `"changeme"`, app vẫn khởi động được, hệ thống tưởng app đang "healthy" và cho nhận traffic. Khi đó mọi request của khách hàng đều sẽ gọi AI bằng key rác `"changeme"` và trả về lỗi, mình sẽ không hề hay biết cho tới khi khách hàng phàn nàn. Việc "chết sớm" đã chặn đứng lỗi đó ngay từ cửa CI/CD.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"timestamp": "2026-08-10T04:47:35.817441+00:00", "level": "info", "event": "ask_completed", "user_id": "sv01", "tokens_in": 471, "tokens_out": 47, "cost_usd": 9.885e-05}`
> 
> Hai việc làm được:
> 1. Dễ dàng dùng các tool như ELK, Datadog lọc ra toàn bộ request theo `user_id = "sv01"` để theo dõi hoặc tính tổng cộng `cost_usd` họ đã tiêu trong ngày.
> 2. Có thể set cảnh báo (alert) tự động để gửi thông báo về Slack/Email ngay khi `tokens_out` hoặc `cost_usd` của một request vượt quá ngưỡng bất thường.

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
| 1 stage (bản đầu) | 1672 MB (với python:3.11 full) |
| Multi-stage | 266 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Ở bản 1-stage (dùng base python:3.11 full), nó chứa cả các công cụ build hệ thống (như gcc, make, build-essential), cache của apt, cache của pip và vô số thư viện hệ thống thừa thãi. Khi chuyển sang Multi-stage, những thứ rác đó chỉ nằm ở stage `builder` và sẽ bị vứt đi. Ở stage `runtime` (slim), ta chỉ copy sang những dependency đã cài đặt xong xuôi và mã nguồn Python, nhờ vậy tiết kiệm được hơn 700MB rác không cần thiết.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Hiện tại: Layer `COPY requirements.txt` và `RUN pip install` vẫn được tái sử dụng (CACHE), Docker chỉ chạy lại từ layer `COPY app/ app/` trở xuống. Tốc độ build sẽ rất nhanh (vài giây).
> - Nếu đặt `COPY . .` LÊN TRƯỚC: Mọi thay đổi nhỏ ở file `.py` sẽ phá vỡ cache của layer `COPY . .`. Hệ quả là lệnh `RUN pip install` đứng ngay sau đó cũng sẽ không được cache, buộc Docker phải tải và cài lại toàn bộ thư viện tốn rất nhiều thời gian mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng RCE (cho phép chèn mã thực thi), hacker có thể chạy lệnh Bash bên trong container. Do container chạy bằng quyền root, hacker có quyền cao nhất để chỉnh sửa hệ thống container, cài cắm mã độc và tìm cách lợi dụng lỗ hổng kernel để "thoát" (escape) ra ngoài thao tác trên hệ điều hành của máy host thật.
> 
> Lệnh `USER appuser` cắt đứt chuỗi này bằng cách hạ cấp đặc quyền. Tiến trình app chỉ chạy với user thường, nên dù hacker có chui vào được container cũng không có quyền cài đặt phần mềm, sửa đổi cấu hình hệ thống hay thoát ra ngoài host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Gửi tối đa **20 request** trong 2 giây.
> Cách đạt được: Người dùng gửi 10 request vào lúc 10:00:59 (thuộc hạn mức của phút 10:00). Ngay 1 giây sau, đồng hồ chuyển sang 10:01:00, biến đếm bị reset, người dùng có thể gửi tiếp 10 request nữa (thuộc hạn mức của phút 10:01). Như vậy trong vòng 2 giây (10:00:59 đến 10:01:00) họ đã spam được 20 request, phá vỡ giới hạn bảo vệ. 

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - Sự khác biệt: Rate Limit giới hạn **tần suất** request (ví dụ: request/phút) để chống spam làm quá tải máy chủ. Cost Guard giới hạn **ngân sách** (tiền USD/tháng) để tránh nguy cơ phá sản do gọi AI quá nhiều.
> - Rate Limit cho qua nhưng Cost Guard chặn: Một người vừa mới gọi request đầu tiên trong tháng này (thoả mãn tần suất Rate Limit), nhưng tháng trước đó họ đã xài quá ngưỡng 10$, hoặc request đầu tiên này chứa một prompt văn bản dài khổng lồ lên tới 10$ tiền token, thì Cost Guard sẽ chặn lại.
> - Rate Limit chặn nhưng Cost Guard cho qua: Người dùng spam liên tục 20 request trong 10 giây với câu hỏi "Hello". Tổng tiền 20 request này mới có 0.001$ (rất xa giới hạn Cost Guard), nhưng vì gọi quá nhanh vượt hạn mức 10req/phút nên Rate Limit sẽ chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Khi Redis mất kết nối, endpoint `/health` (do bị gộp chung) của cả 3 container sẽ đều báo lỗi 503 cho hệ thống (VD: Kubernetes/Docker Swarm).
> 2. Hệ thống lầm tưởng rằng process bên trong 3 container này đang bị kẹt/treo, nên nó tự động **restart/kill** cả 3 container cùng lúc.
> 3. Trong 30s sau khi Redis sống lại, toàn bộ các container vẫn đang bị kill hoặc đang trong quá trình start up. Hậu quả là toàn bộ hệ thống bị sập (downtime 100%), không còn container nào để hứng request của người dùng. Một sự cố mạng nhỏ đã bị phóng đại thành sập toàn server.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> `history_length` sẽ tăng một cách loạn xạ và không liên tục (ví dụ: gọi 4 lần ra kết quả là 1, 1, 2, 1). Lý do là vì Load Balancer (Nginx) sẽ phân phát ngẫu nhiên các request vào 3 container A, B, C. Nếu mỗi container lưu dict Python ở RAM riêng lẻ, chúng không hề biết State của nhau. Lần thứ nhất gọi trúng A, lần hai gọi trúng B (B lại đếm từ 1), khiến bot AI như bị "mất trí nhớ" bất thường đối với người dùng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Thông báo lỗi: `Multiple services found. Please specify a service via the --service flag.` khi CI/CD chạy `railway up`. 
> - Nguyên nhân tìm ra: Do trong project Railway tôi đã tạo cả Web service và Database Redis. Lệnh CLI của Github Actions không biết phải đẩy code mới đè lên cái nào.
> - Cách sửa: Khai báo thêm Secret Variable `RAILWAY_SERVICE_NAME` trên Github và sửa file action CI thành `railway up --service ${{ vars.RAILWAY_SERVICE_NAME }}` để chỉ đích danh service. Ngoài ra tôi còn phát hiện test local bị 404 là do file `DEPLOYMENT.md` vô tình ghi thừa đuôi `/ask` trong URL public, tôi đã xoá nó.
