# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dương Phương Hiếu  Mã học viên: L3B202603008

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy Render, nếu quên đặt `AGENT_API_KEY` mà code dùng khóa mặc định `changeme`, service vẫn chạy và endpoint `/ask` có thể bị người khác gọi bằng chính khóa mặc định đó. Không có default khiến `Settings` báo lỗi ngay lúc khởi động; tôi phát hiện thiếu biến trong dashboard trước khi public URL hoạt động, thay vì phát hiện sau khi API đã bị lộ.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log tôi thu được: `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T09:46:18.048277+00:00","user_id":"exercise-log","tokens_in":6,"tokens_out":38,"cost_usd":2.37e-05}`. Với JSON này tôi có thể lọc tất cả event `ask_completed` của một user để điều tra lỗi, và tính/tổng hợp `cost_usd` hoặc token theo thời gian để theo dõi chi phí. Một câu `print("đã trả lời xong")` không có field chuẩn để máy lọc, nhóm hay tính toán.

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
| 1 stage (bản đầu) | Không đo hoàn tất: Docker Desktop hết bộ nhớ khi pull/build `python:3.11` |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage đo được 271 MB. Khi tôi thử build lại Dockerfile one-stage ban đầu, Docker Desktop báo `fatal error: runtime: cannot allocate memory` trong lúc tải/build image `python:3.11`, nên tôi không ghi một con số ước lượng làm số đo thật. Sự cố này cũng cho thấy base image đầy đủ nặng hơn nhiều. Multi-stage chỉ giữ runtime `python:3.11-slim` và các package đã cài ở `/usr/local`; compiler, cache pip và lớp build không đi vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi chỉ sửa source, các layer `COPY requirements.txt` và `RUN pip install` của stage builder vẫn được cache; layer `COPY app`/`COPY utils` ở runtime cần chạy lại vì source đổi. Nếu đặt `COPY . .` trước `RUN pip install`, bất kỳ thay đổi nhỏ nào trong code cũng làm cache của layer COPY mất hiệu lực và buộc cài lại toàn bộ dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: lỗi Python bị khai thác cho phép chạy lệnh trong container; tiến trình đang là root nên lệnh đó đọc/sửa được mọi file mà root container truy cập được; nếu có volume, socket Docker hoặc cấu hình host bị mount sai, kẻ tấn công có thể mở rộng quyền ra host. `USER app` chuyển tiến trình ứng dụng sang user không đặc quyền, nên ngay sau bước thực thi lệnh trong container, thao tác cần quyền root sẽ bị chặn hoặc bị giới hạn đáng kể.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong khoảng 2 giây: gửi 10 request lúc 10:00:59, rồi đồng hồ sang 10:01:00 và bộ đếm theo phút bị reset, gửi tiếp 10 request nữa. Sliding window 60 giây của tôi không reset theo phút; 10 request đầu vẫn nằm trong cửa sổ nên các request tiếp theo bị 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất trong 60 giây, còn cost guard giới hạn tổng chi phí theo user trong tháng. Một user gọi chậm 1 request/phút nhưng đã tiêu gần hết hoặc vượt budget tháng có thể qua rate limit mà bị cost guard chặn. Ngược lại, user còn gần như toàn bộ budget nhưng gửi request thứ 11 trong cùng một phút sẽ bị rate limit chặn, dù mỗi request rất rẻ.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng ping Redis, Redis mất kết nối 30 giây sẽ làm health của cả 3 agent trả lỗi. Orchestrator hiểu nhầm từng process là chết và restart cả ba container; các request đang xử lý có thể bị ngắt và khi Redis trở lại các instance lại khởi động đồng thời. Tách `/health` chỉ kiểm tra process giúp không restart dây chuyền, còn `/ready` trả 503 để load balancer tạm ngừng gửi traffic vào instance không dùng được Redis.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, lần hỏi đầu có `history_length` là 0; lần kế tiếp cùng user thấy 2 message (câu hỏi và câu trả lời trước), kể cả khi Nginx có thể chuyển request sang agent khác trong 3 replica. Nếu dùng dict Python trong mỗi container, history_length sẽ lúc tăng ở replica nhận request trước, lúc quay về 0 khi request bị gửi sang replica khác; hội thoại sẽ mất ngữ cảnh không ổn định.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau deploy tôi mở URL gốc và nhận `{"detail":"Not Found"}`. Tôi không vội sửa Dockerfile vì dashboard Render vẫn báo Live; kiểm tra đúng `/health` và `/ready` đều trả 200, còn `/ask` không key trả 401. Nguyên nhân là FastAPI không có route `/`, không phải deploy thất bại. Tôi sửa cách kiểm tra và ghi tài liệu dùng các endpoint health/ready thay vì URL gốc.
