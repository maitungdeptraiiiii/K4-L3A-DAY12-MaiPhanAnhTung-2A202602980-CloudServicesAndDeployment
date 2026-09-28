# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời mẫu ở mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Mai Phan Anh Tùng          
> Mã học viên: 2A202602980

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> *Trong test thực tế, container vẫn khởi động và `/health` trả 200 dù thiếu `AGENT_API_KEY`, nhưng khi gọi `/ask` thì trả 500 và log ghi `ValidationError: 1 validation error for Settings ... agent_api_key Field required`. Điều này vẫn tốt hơn việc đặt mặc định `"changeme"` vì `Settings` không che giấu lỗi cấu hình bằng một API key giả: tôi phát hiện ngay ở request đầu tiên rằng biến môi trường bắt buộc đang bị thiếu. Nếu dùng `"changeme"`, service có thể chạy bình thường và che mất lỗi cấu hình cho đến khi request sử dụng credential giả.*

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> *Log JSON tôi thu được là: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T13:43:52.535693+00:00", "user_id": "sv02", "tokens_in": 269, "tokens_out": 52, "cost_usd": 7.155e-05}`. Từ log này, tôi có thể (1) lọc riêng các request của user `sv02` và (2) cộng dồn số request, tổng số token hoặc tổng chi phí `cost_usd` của user đó. Trong khi đó, `print("đã trả lời xong")` chỉ cho biết request đã hoàn thành, không có dữ liệu có cấu trúc để lọc hay thống kê.*


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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *1 stage (bản đầu): **1.73 GB**; Multi-stage: **310 MB**. Chênh lệch khoảng **1.4 GB**, chủ yếu do bản 1 stage dùng `python:3.11` đầy đủ thay vì `python:3.11-slim`, đồng thời đưa cả các công cụ và thư viện phục vụ build vào image cuối. Bản multi-stage chỉ copy môi trường `/opt/venv` cần thiết sang stage runtime và chỉ copy các thư mục source cần chạy, nên image nhỏ hơn đáng kể. Image nhỏ giúp pull và deploy nhanh hơn, đồng thời giảm các thành phần không cần thiết trong image.*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Khi sửa một ký tự trong `app/main.py`, các layer tạo môi trường và cài dependency (`FROM`, `RUN python -m venv`, `COPY requirements.txt`, `RUN pip install`) vẫn được dùng lại từ cache; chỉ layer `COPY app/` và các layer phía sau phải build lại. Với `Dockerfile.bad` dùng `COPY . .` trước `RUN pip install`, chỉ cần sửa `app/main.py` cũng làm layer `COPY . .` thay đổi, khiến `RUN pip install` phải chạy lại dù `requirements.txt` không đổi, làm build chậm hơn.*

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *Kết quả thực tế cho thấy bản `agent` chạy bằng user `app` (uid `10001`) nên không thể ghi vào `/etc/hacked`, trong khi bản `agent:single` chạy bằng `uid=0(root)` và ghi được. Chuỗi sự kiện là: một lỗ hổng trong code Python có thể cho phép kẻ tấn công thực thi lệnh trong container; nếu process chạy bằng root thì các lệnh đó có quyền root trong container, có thể đọc secret, sửa file hoặc cài công cụ. Do container dùng chung kernel với host, root trong container vẫn là uid 0 ở phía kernel, dù bị giới hạn bởi namespace và capability; vì vậy nếu có thêm lỗ hổng runtime/kernel hoặc cấu hình nguy hiểm như `docker.sock`, `--privileged` hay mount thừa, kẻ tấn công có thể tiếp tục khai thác để thoát container và có quyền cao trên host. Lệnh `USER 10001` cắt chuỗi ở bước quyền thực thi: nó không ngăn kẻ tấn công chạy lệnh khi khai thác được lỗ hổng, nhưng khiến các lệnh đó chạy với quyền thấp, nên kẻ tấn công phải tìm thêm một lỗ hổng leo thang quyền trước khi có thể tiếp tục.*

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Với cách đếm theo phút đồng hồ, người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp**. Cụ thể, gửi 10 request ngay trước giây 00 của một phút, sau đó gửi tiếp 10 request ngay sau giây 00 của phút kế tiếp. Hai nhóm request chỉ cách nhau khoảng 2 giây nhưng thuộc hai phút khác nhau nên đều được tính đủ 10 request.*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Rate limit giới hạn **số lượng request trong một khoảng thời gian**, còn cost guard giới hạn **tổng chi phí sử dụng** theo ngân sách. Ví dụ, một người dùng gửi rất ít request nên rate limit luôn cho qua, nhưng mỗi request đều đắt do prompt dài hoặc dùng model đắt, khiến tổng chi phí nhanh chóng vượt ngân sách; khi đó **request tiếp theo bị cost guard chặn (402)** dù chưa vượt rate limit. Ngược lại, người dùng có thể gửi **10 request nhỏ liên tiếp**, mỗi request gần như không tốn tiền nên cost guard vẫn cho qua, nhưng **request thứ 11 trong 60 giây bị rate limit chặn (429)**.*

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Giả sử cụm có **3 container agent** cùng phụ thuộc vào một Redis và healthcheck chạy mỗi **10 giây**, cần **3 lần fail liên tiếp** để orchestrator restart. Khi Redis mất kết nối, thứ tự xảy ra là: **(1)** Redis ngừng hoạt động → **(2)** cả 3 container agent đều kiểm tra Redis thất bại vì cùng phụ thuộc vào Redis đó → **(3)** sau khoảng **30 giây**, healthcheck của cả 3 đều fail → **(4)** orchestrator restart đồng loạt 3 container, các request đang xử lý có thể bị cắt → **(5)** nếu Redis vẫn chưa trở lại, các container mới khởi động lại tiếp tục fail healthcheck và rơi vào vòng lặp restart → **(6)** khi Redis trở lại, các container cần thêm thời gian để khởi động và vượt qua healthcheck, sau đó cụm mới phục hồi phục vụ bình thường. Kết quả thực tế của tôi cũng cho thấy khi Redis chạy thì `/health` trả **200** và `/ready` trả **200**, khi Redis bị stop thì `/health` vẫn **200** nhưng `/ready` trả **503**, và sau khi Redis start lại thì `/ready` trở lại **200**. Điều này cho thấy `/health` nên chỉ kiểm tra process còn sống, còn `/ready` kiểm tra service đã sẵn sàng nhận request.*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Kết quả thực tế cho thấy 6 request cùng một `X-User-Id` có `history_length` lần lượt là **0, 2, 4, 6, 8, 10**, và cả **3 instance agent** đều nhận request, cho thấy lịch sử được lưu trong Redis dùng chung nên cả 3 container cùng nhìn thấy một state nhất quán. Nếu thay Redis bằng một `dict` Python trong mỗi process, với 3 container nhận request theo round-robin thì mỗi container sẽ có một bộ nhớ riêng; giả sử cả 3 dict ban đầu rỗng, dãy `history_length` dự kiến sẽ là **0, 0, 0, 2, 2, 2**: request thứ 1, 2, 3 lần lượt vào ba container nên mỗi container mới thấy user lần đầu; request thứ 4 quay lại container đầu tiên nên nó thấy lịch sử của request 1 và trả `2`, tương tự các request tiếp theo. Điều này cho thấy state trong RAM không nhất quán khi scale ngang và sẽ mất hoàn toàn khi container restart, trong khi Redis giữ state dùng chung giữa các instance.*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Một lỗi tôi gặp khi deploy lên cloud là cấu hình `REDIS_URL` sai khiến service không sẵn sàng. Sau khi thay đổi `REDIS_URL`, tôi gọi endpoint `/ready` và nhận HTTP 503 với response `{"status":"not ready","redis":false}`. Log của `/ready` chỉ cho thấy request trả về 503, không có thông báo chi tiết về lỗi kết nối Redis vì `store.ping()` đã bắt exception và trả về `False`. Để tìm nguyên nhân, tôi gọi thử `/ask` và kiểm tra traceback của request, sau đó đối chiếu giá trị `REDIS_URL` trong Variables của Railway với biến kết nối của service Redis. Tôi xác định `REDIS_URL` đang trỏ sai, sửa lại thành `${{Redis.REDIS_URL}}`, deploy lại và kiểm tra `/ready`. Sau khi sửa, endpoint trả HTTP 200 với `{"status":"ready","redis":true}`, xác nhận service đã kết nối lại được với Redis.*
