# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu quên khai báo `API_TOKEN`, ứng dụng sẽ dừng ngay và log chỉ rõ biến cấu hình bị thiếu. Nhờ vậy tôi sửa biến môi trường trước khi service nhận traffic. Nếu dùng mặc định `"changeme"`, service vẫn chạy và bất kỳ ai đoán được token đó có thể gọi `/chat`, gây lộ chức năng và phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log tôi thu được khi gọi `/chat`:
> `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T16:31:34.032400+00:00", "client_id": "exercise", "prompt_tokens": 2, "completion_tokens": 34, "usd_cost": 2.07e-05}`
>
> Hệ thống log có thể lọc hoặc đếm các sự kiện `chat_completed` theo `client_id` để theo dõi mức sử dụng của từng client. Nó cũng có thể cộng trường `usd_cost` hoặc tạo cảnh báo khi chi phí, số token, hay số lỗi vượt ngưỡng. Chuỗi `print("đã trả lời xong")` không có cấu trúc và không mang các dữ liệu này để máy truy vấn đáng tin cậy.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> | Bản | Dung lượng |
> |-----|-----------|
> | 1 stage (bản đầu) | 1.8 GB |
> | Multi-stage | 270 MB |
>
> Phần chênh lệch chủ yếu là compiler, header và thư viện phát triển cần để build dependency, cùng pip cache và các công cụ build khác. Multi-stage chỉ copy các package đã cài từ stage `builder` sang image runtime, nên runtime không mang những thành phần đó.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa một ký tự trong `app/main.py`, các layer `FROM`, `WORKDIR`, `COPY requirements.txt` và `RUN pip install` được dùng lại từ cache vì `requirements.txt` không đổi. Layer copy source `app` và các layer sau nó phải chạy lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm layer copy thay đổi, cache của `RUN pip install` bị mất và Docker phải cài lại dependency dù requirements không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng có thể cho kẻ tấn công thực thi lệnh trong process Python. Nếu container chạy bằng root, lệnh đó có quyền root trong container; kết hợp với cấu hình Docker, volume mount, socket Docker hoặc một lỗ hổng container escape, kẻ tấn công có thể đọc/sửa tài nguyên host với quyền rất cao. `USER appuser` khiến process ứng dụng và lệnh bị chiếm quyền chỉ có quyền của user thường, nên không thể trực tiếp ghi vào các vị trí chỉ root được phép; nó giảm đáng kể tác động ngay cả khi có RCE.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Header `WWW-Authenticate: Bearer` là yêu cầu của cơ chế HTTP authentication cho phản hồi 401: nó cho client biết endpoint yêu cầu Bearer token và cách xác thực cần dùng. Dùng cùng một lỗi cho thiếu header, sai scheme và sai token tránh tiết lộ token có tồn tại hay client đã sai ở bước nào. Thông tin chi tiết đó giúp kẻ tấn công dò cơ chế xác thực hoặc thử token, trong khi client hợp lệ vẫn biết cần gửi Bearer token.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Xô tối đa chỉ chứa 10 token, nên sau 10 phút im lặng client gửi liên tiếp được 10 request trước khi request tiếp theo nhận 429. Nếu xét client đã làm xô cạn rồi im lặng 10 phút và bỏ `min(capacity, ...)`, nó tích thêm 100 token (`10 request/phút x 10 phút`) và có thể bắn 100 request liên tiếp. `min` giữ số token không vượt quá sức chứa để thời gian im lặng không biến thành một burst không giới hạn.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng, sự cố từ 2h sáng có thể tiêu tối đa $30 cho client đó trước khi bị chặn; service chỉ tự có ngân sách lại khi sang chu kỳ tháng mới. Với $1/ngày, thiệt hại tối đa trong ngày là $1 và key chi tiêu dùng ngày UTC, nên sang ngày UTC tiếp theo client có ngân sách mới mà không cần can thiệp. Hạn mức ngày khoanh vùng thiệt hại của sự cố xuống nhỏ hơn nhiều.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Khi Redis mất kết nối, endpoint gộp trả lỗi vì kiểm tra `ping()`. Health check của platform coi cả ba container là không khỏe, lần lượt hoặc đồng thời rút chúng khỏi traffic và restart chúng. Redis vẫn đang mất kết nối nên container vừa khởi động lại lại fail health check, tiếp tục bị restart; trong khoảng 30 giây đó cụm có thể không còn instance nào phục vụ được liveness dù process Python vẫn bình thường. Tách `/healthz` giúp platform chỉ restart khi process lỗi, còn `/readyz` trả 503 để ngừng nhận traffic cho đến khi Redis hồi phục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Khi deploy lên cloud tôi thấy tôi bị lỗi build fail do setup Dockerfile bị sai và thiếu mất một sốthứ quan trọng như 
COPY --from=builder /install /usr/local
COPY --from=builder /app/app ./app
COPY --from=builder /app/utils ./utils*
