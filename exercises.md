# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Các câu trả lời dưới đây dựa trên những gì tôi đã
> kiểm tra trong code, test và quá trình deploy.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Quang Hữu  Mã học viên: 2A202602756

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ, nếu tôi deploy mà quên đặt `AGENT_API_KEY`, ứng dụng dừng ngay lúc khởi động và log lỗi cấu hình. Tôi có thể sửa secret trước khi mở public URL. Nếu dùng mặc định `changeme`, service vẫn chạy và người ngoài có thể đoán khóa đó để gọi API, làm phát sinh chi phí mà tôi không biết.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng JSON tôi ghi nhận được khi kiểm tra `log_event`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:01:08.745046+00:00", "user_id": "sv-test", "tokens_in": 8, "tokens_out": 24, "cost_usd": 0.0003}
> ```
>
> Từ các field này, tôi có thể lọc event theo `user_id` hoặc `event`, và cộng `cost_usd`/token để theo dõi chi phí. Một câu `đã trả lời xong` không cho máy biết user nào gọi hay request đó tốn bao nhiêu.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Content size | Disk usage hiển thị bởi Docker |
|-----|--------------|---------------------------------|
| 1 stage (bản đầu) | 447 MB | 1.73 GB |
| Multi-stage | 63.9 MB | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build hai image bằng Docker Desktop và ghi cả hai cột mà Docker 29.5.2 hiển thị. Theo `CONTENT SIZE`, bản một stage là 447 MB, bản multi-stage là 63.9 MB, giảm khoảng 383.1 MB (xấp xỉ 86%). Theo `DISK USAGE`, các số lần lượt là 1.73 GB và 271 MB. Bản multi-stage dùng `python:3.11-slim`, không giữ pip cache, và chỉ chuyển dependencies đã cài cùng source cần chạy sang runtime. Image multi-stage dưới giới hạn 500 MB theo cả hai số đo.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer cài requirements vẫn cache vì `requirements.txt` không đổi. Layer `COPY app/` bị build lại; các bước filesystem sau nó cũng được thực hiện lại. Nếu `COPY . .` nằm trước `pip install`, sửa bất kỳ file nào trong build context sẽ làm layer copy đổi, khiến pip install chạy lại và tốn thời gian tải/cài dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu có lỗ hổng cho phép chạy lệnh trong ứng dụng, tiến công trước chiếm quyền trong process/container. Khi process là root, lệnh đó cũng có quyền root bên trong container; nếu cấu hình container yếu hoặc quyền truy cập host bị lộ, nguy cơ ảnh hưởng host tăng cao. `USER appuser` làm process chạy với UID thường, nên cắt quyền root ngay trong container. Đây là giảm quyền, không phải bảo đảm tuyệt đối rằng mọi kiểu thoát container đều bị chặn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với bộ đếm reset lúc phút mới, tôi có thể gửi 10 request ở giây 59 và thêm 10 request ngay ở giây 00 của phút kế tiếp. Như vậy có tới 20 request trong khoảng hai giây dù giới hạn ghi là 10/phút. Sliding window tính các request trong 60 giây gần nhất nên không có lỗ hổng do ranh giới phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm số lần gọi trong 60 giây; cost guard cộng tiền đã dùng trong tháng UTC. Một user còn quota request nhưng chi phí tháng đã chạm ngân sách thì rate limit vẫn cho qua còn cost guard trả 402. Ngược lại, user mới chưa tốn tiền nên cost guard cho qua, nhưng nếu đã gửi đủ 10 request trong phút vừa rồi thì rate limit chặn request kế tiếp bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint health chung kiểm tra Redis, Redis mất kết nối thì health check của cả ba container sẽ lỗi. Orchestrator có thể hiểu nhầm cả ba process đã chết và restart chúng; Redis vẫn hỏng nên health tiếp tục lỗi, tạo vòng restart không sửa được nguyên nhân. Tách endpoint giúp `/ready` báo không nhận traffic, còn `/health` vẫn báo process còn sống để orchestrator không restart nhầm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi request mới thấy lịch sử của user dù request trước được xử lý ở container khác; `history_length` tăng theo các message đã lưu (0 ở lần đầu, rồi 2, 4, ...). Nếu dùng dict trong RAM, mỗi container có bản riêng: khi request chuyển sang instance chưa từng thấy user, `history_length` có thể lại là 0; kết quả thay đổi tùy request được load balancer gửi tới đâu.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi kiểm tra `/ask` bằng `curl.exe` trong PowerShell, tôi gặp `422` với `JSON decode error` và thông báo `Expecting property name enclosed in double quotes`. Response body giúp tôi xác định request body bị biến dạng bởi cách PowerShell truyền dấu nháy cho chương trình native `curl.exe`, chứ request chưa tới bước xác thực. Dùng `Invoke-RestMethod` với JSON trong chuỗi nháy đơn cho request không có key đã trả đúng `401`. Với `curl.exe`, cách an toàn là ghi JSON ra file UTF-8 không BOM rồi gửi bằng `--data-binary @file`; cần xác nhận request có key trả `200` trước khi đánh giá rate limit `429`.
