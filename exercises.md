# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder mặc định bằng câu trả lời thật của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Duy Hoàng  Mã học viên: 2A202601147

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway/Render, nếu quên set `AGENT_API_KEY` thì app sẽ crash
> ngay lúc start và dashboard báo lỗi rõ. Nếu mặc định là `"changeme"`, service
> vẫn lên xanh, bot Internet quét được URL công khai và gọi `/ask` bằng khóa
> mặc định đó — mình chỉ phát hiện khi hóa đơn LLM tăng hoặc log đầy request lạ.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ dòng log:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:15:22.123456+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 40, "cost_usd": 0.0001}`
>
> Hai việc làm được: (1) lọc/đếm theo `user_id` hoặc `event` trên dashboard
> cloud để biết ai gọi nhiều nhất; (2) cộng `cost_usd` theo khung thời gian để
> cảnh báo khi chi phí tăng bất thường — `print` thuần chữ không parse được.

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
| 1 stage (bản đầu) | ~900–1100 MB (python:3.11 đầy đủ) |
| Multi-stage | ~150–250 MB (python:3.11-slim + deps) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu là toolchain/compiler và lớp hệ thống của image đầy đủ
> (`build-essential`, header, công cụ build) chỉ cần ở stage builder. Stage
> runtime chỉ nhận site-packages đã cài sẵn trên base slim, nên image cuối nhẹ hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với thứ tự hiện tại (COPY requirements → pip install → COPY app/utils): layer
> cài dependency được cache lại; chỉ các layer sau khi COPY code phải build lại.
> Nếu `COPY . .` đứng trước `pip install`, mọi thay đổi nhỏ trong code đều hủy
> cache và buộc Docker cài lại toàn bộ thư viện — build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng (ví dụ RCE qua input) → process trong container chạy với quyền root
> → nếu có thêm lỗ hổng escape container hoặc mount nhầm volume hệ thống, kẻ
> tấn công thao tác file/host với quyền root. Lệnh `USER appuser` cắt chuỗi
> ngay từ đầu: dù thoát được app, tiến trình vẫn chỉ có quyền user thường,
> không phải root trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong ~2 giây: gửi 10 request ở 10:00:59 (hết quota phút
> cũ), rồi ngay lúc 10:01:01 gửi thêm 10 request (quota phút mới vừa reset).
> Cửa sổ theo phút đồng hồ không nhìn liên tục qua ranh giới phút, nên bị “burst”
> gấp đôi. Sliding window 60 giây gần nhất sẽ chặn burst này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất** (số request/phút); cost guard giới hạn **tiền**
> (USD/tháng). Ví dụ rate limit cho qua nhưng cost guard chặn: user chỉ gọi
> 5 lần/phút nhưng mỗi lần ~50k token, hết ngân sách tháng. Ngược lại: user còn
> nhiều tiền tháng nhưng spam 15 request trong 10 giây → rate limit trả 429,
> cost guard chưa vượt.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → cả 3 container báo unhealthy vì probe phụ thuộc Redis →
> orchestrator restart cả 3 cùng lúc → trong lúc restart không còn instance phục
> vụ → khi Redis sống lại cũng mất thời gian chờ container lên lại. Sự cố nhỏ
> thành outage toàn cụm. Tách `/ready` thì LB chỉ rút traffic, không restart hàng loạt.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, `history_length` tăng dần ổn định dù request rơi vào container khác.
> Nếu dùng dict trong RAM, mỗi instance giữ lịch sử riêng nên số sẽ nhảy lung
> tung (ví dụ 0 → 2 → 0 → 4) tùy request đi vào container nào — agent “mất trí nhớ”.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thường gặp: `/ready` trả 503 sau khi deploy. Log/dashboard cho thấy app
> không ping được Redis vì `REDIS_URL` chưa gắn vào service agent (chỉ có Redis
> add-on riêng). Cách sửa: gắn biến `REDIS_URL` từ add-on vào web service, redeploy,
> rồi gọi lại `/ready` đến khi trả 200. (Nếu chưa deploy cloud được: dùng
> `LOCAL_FALLBACK=true` + `docker compose up -d` và ghi rõ lý do trong DEPLOYMENT.md.)
