# BF01 — Business Flow: Phòng Chờ Ảo & Hàng Chờ Ảo (Waiting Room & Virtual Queue)

## 1. Mục Đích

Tài liệu mô tả chi tiết luồng nghiệp vụ của cơ chế **Phòng Chờ Ảo (Virtual Waiting Room)** và **Hàng Chờ Ảo (Virtual Queue)** — trái tim của hệ thống đảm bảo công bằng và chống quá tải khi mở bán vé concert.

Luồng này là căn cứ trực tiếp để:
- Vẽ **Sequence Diagram** chi tiết cho Queue Service và Seat Selection Service.
- Thiết kế kiến trúc Redis Queue + WebSocket Server + CDN Edge.
- Xác định các API Endpoints cần thiết cho luồng phòng chờ.

---

## 2. Tổng Quan Pha (Phase Overview)

Luồng Phòng Chờ gồm **4 pha** liên tiếp:

```mermaid
timeline
    title Vòng Đời Hàng Chờ Mua Vé Concert

    section T-15 phút
        Phòng chờ mở: Khách hàng vào phòng chờ ảo
        Hiển thị countdown: Đếm ngược đến giờ mở bán

    section T0 (Giờ mở bán chính thức)
        Shuffle ngẫu nhiên: Tất cả người trong phòng chờ được xáo trộn thứ tự
        Cấp Queue Position: Mỗi user nhận 1 số thứ tự duy nhất

    section Sau T0
        FIFO Queue: Người đến sau xếp cuối hàng theo thứ tự thời gian
        Kiểm soát vào khu chọn ghế: Tối đa 5.000 người đồng thời

    section Đóng hàng chờ
        Hết vé hoặc Admin đóng: Hàng chờ đóng lại
        Thông báo toàn bộ: Push notification đến tất cả đang chờ
```

---

## 3. Luồng Chi Tiết Theo Từng Pha

### Pha 1: T-15 Phút — Mở Phòng Chờ

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant CDN as CDN Edge<br/>(Cloudflare Waiting Room)
    participant WR as Waiting Room Service
    participant AUTH as Auth Service
    participant REDIS as Redis Cluster

    Note over WR: Scheduler phát hiện:<br/>current_time >= sale_start_time - 15m<br/>→ Kích hoạt phòng chờ concert_id
    WR->>REDIS: SET waitingroom:{concert_id}:active = true EX {15m+duration}
    WR->>CDN: Cấu hình Waiting Room rule cho concert endpoint

    C->>CDN: GET /concerts/{id}/waiting-room
    CDN->>CDN: Kiểm tra Waiting Room rule
    CDN-->>C: HTML Waiting Room Page<br/>(Đồng hồ đếm ngược, số người đang chờ)

    C->>AUTH: Xác thực JWT Token (nếu chưa đăng nhập → redirect login)
    AUTH-->>C: JWT hợp lệ

    C->>WR: POST /queue/{concert_id}/join<br/>Header: Authorization: Bearer {jwt}
    WR->>REDIS: SETNX queue:user:{concert_id}:{user_id} = "WAITING" EX {15m+sale_duration}
    Note over REDIS: NX = chỉ set nếu chưa có<br/>→ Đảm bảo mỗi user chỉ join 1 lần
    WR->>REDIS: SADD queue:{concert_id}:pre_sale_pool {user_id}
    WR-->>C: 200 OK<br/>{status: "IN_WAITING_ROOM", position: null, countdown: 840}
```

### Pha 2: T0 — Shuffle Ngẫu Nhiên & Phân Phát Số Thứ Tự

```mermaid
sequenceDiagram
    participant SCHED as Scheduler<br/>(Cron job)
    participant WR as Waiting Room Service
    participant REDIS as Redis Cluster
    participant WS as WebSocket Server
    participant C1 as Khách hàng 1
    participant C2 as Khách hàng 2
    participant Cn as Khách hàng N

    Note over SCHED: current_time = concert.sale_start_time
    SCHED->>WR: TriggerSaleOpen(concert_id)

    WR->>REDIS: SMEMBERS queue:{concert_id}:pre_sale_pool
    REDIS-->>WR: [user_1, user_2, ..., user_N] (danh sách tất cả user trong phòng chờ)

    WR->>WR: Fisher-Yates Shuffle:<br/>Xáo trộn ngẫu nhiên mảng user_ids

    loop Với mỗi user_id sau khi shuffle
        WR->>REDIS: ZADD queue:{concert_id}:sorted_queue {queue_position} {user_id}
        Note over REDIS: Sorted Set: score = số thứ tự (1, 2, 3...)
    end

    WR->>REDIS: RENAME queue:{concert_id}:pre_sale_pool → [expired]<br/>DEL pre_sale_pool

    WR->>WS: BroadcastToAll(concert_id):<br/>{"event": "SALE_OPENED", "message": "Đã đến giờ mở bán!"}

    WS-->>C1: {"event":"QUEUE_POSITION_ASSIGNED","your_position": 1, "total_queue": N}
    WS-->>C2: {"event":"QUEUE_POSITION_ASSIGNED","your_position": 247, "total_queue": N}
    WS-->>Cn: {"event":"QUEUE_POSITION_ASSIGNED","your_position": N, "total_queue": N}
```

### Pha 3: Kiểm Soát Vào Khu Chọn Ghế (Admission Control — Max 5.000 CCU)

```mermaid
flowchart TD
    A["Queue Service kiểm tra liên tục:\nSố người đang trong khu chọn ghế"] --> B{active_users < 5.000?}

    B -->|Còn chỗ| C["Lấy user_id tiếp theo từ Sorted Set:\nZRANGE queue:{concert_id}:sorted_queue 0 0"]
    C --> D["Kiểm tra user có còn online không\n(Heartbeat còn sống?)"]
    D --> E{User online?}
    E -->|Offline| F["Bỏ qua user này\nZREM khỏi sorted_queue\nChuyển sang user tiếp theo"]
    F --> C
    E -->|Online| G["Sinh Queue Access Token:\nJWT {user_id, concert_id, queue_pos, exp: now()+120s}"]
    G --> H["Gửi Access Token cho user qua WebSocket:\n{event: 'YOUR_TURN', access_token: '...', expires_in: 120}"]
    H --> I["Tăng active_user_count trên Redis:\nINCR active:seat_selection:{concert_id}"]
    I --> J["User vào khu chọn ghế thành công"]

    B -->|Đã đủ 5.000| K["Chờ đến khi có người rời:\n- Hoàn tất thanh toán\n- Timeout 2 phút inactive\n- Hủy đơn & thoát ra\n\nSub Redis channel: seat_selection_left:{concert_id}"]
    K --> A
```

### Pha 4: Timeout Không Hoạt Động & Thu Hồi Quyền (Inactivity Revocation)

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant APP as Web/Mobile App
    participant API as API Gateway
    participant REDIS as Redis Cluster
    participant WS as WebSocket Server

    Note over C,APP: User đang trong khu chọn ghế (có Access Token)

    loop Mỗi 30 giây — Client Heartbeat
        APP->>API: POST /queue/heartbeat<br/>Header: X-Queue-Token: {access_token}
        API->>REDIS: EXPIRE user:activity:{concert_id}:{user_id} 120
        API-->>APP: 200 OK {status: "alive"}
    end

    Note over C,APP: Giả sử user đóng tab / mất mạng
    Note over REDIS: Sau 120 giây không nhận heartbeat<br/>Key user:activity:{concert_id}:{user_id} TỰ XÓA

    REDIS->>REDIS: Keyspace Notification:<br/>__keyevent@0__:expired → user:activity:*
    REDIS->>API: Session expired event

    API->>REDIS: DECR active:seat_selection:{concert_id}
    API->>REDIS: Xóa tất cả ghế đang HELD của user này
    API->>WS: Broadcast SeatReleasedEvent cho concert

    Note over WS,C: Nếu user quay lại sau 2 phút
    C->>APP: Trở lại website
    APP->>API: POST /queue/{concert_id}/join (join lại)
    API-->>APP: Vị trí mới = cuối hàng chờ hiện tại (BR-QUEUE-003)
    APP-->>C: "Phiên của bạn đã hết hạn do không hoạt động.\nBạn đang ở vị trí #{new_position} trong hàng chờ."
```

---

## 4. Bảo Vệ Chống Gian Lận & Bot (Anti-Fraud & Bot Protection)

### 4.1 Các hình thức gian lận và biện pháp đối phó

| Loại gian lận | Biểu hiện | Biện pháp đối phó |
|---|---|---|
| **F5 Spam / Click bot** | Gửi hàng nghìn request GET `/concerts/{id}` | CDN Rate Limiting (Cloudflare) + `429 Too Many Requests` |
| **Multi-tab farming** | Mở 10 tab để giữ nhiều vị trí | SETNX Redis: Mỗi `user_id` chỉ có 1 entry trong queue |
| **Multi-device** | Dùng điện thoại + laptop cùng lúc | Cùng `user_id` → cùng queue position, không được cộng thêm |
| **Account farming (nhiều nick)** | Tạo nhiều tài khoản ảo | Yêu cầu xác thực SĐT (OTP); 1 SĐT = 1 tài khoản; reCAPTCHA v3 khi đăng ký |
| **Queue Token theft** | Đánh cắp `X-Queue-Token` của người khác | Token binding `user_id + IP + device_fingerprint`; TTL ngắn 120s |
| **Automated checkout bot** | Bot tự động chọn ghế và điền form | Fingerprinting JS challenge (không CAPTCHA blocking để UX tốt) |

---

## 5. API Endpoints Liên Quan (Queue Service API Reference)

| Method | Endpoint | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/queue/{concert_id}/join` | Bearer JWT | Tham gia hàng chờ / phòng chờ |
| `GET` | `/api/v1/queue/{concert_id}/status` | Bearer JWT | Xem số thứ tự và vị trí hiện tại |
| `POST` | `/api/v1/queue/heartbeat` | Bearer JWT + X-Queue-Token | Gửi heartbeat duy trì phiên chọn ghế |
| `DELETE` | `/api/v1/queue/{concert_id}/leave` | Bearer JWT | Rời khỏi hàng chờ (tự nguyện) |
| `WS` | `wss://api.concert.vn/ws/queue/{concert_id}` | JWT query param | Kênh WebSocket nhận sự kiện real-time |

### WebSocket Events (Server → Client)

| Event | Payload | Ý nghĩa |
|---|---|---|
| `WAITING_ROOM_OPEN` | `{countdown: 900, total_waiting: N}` | Phòng chờ vừa mở |
| `QUEUE_POSITION_ASSIGNED` | `{your_position: N, total_queue: M}` | Số thứ tự được cấp sau shuffle |
| `YOUR_TURN` | `{access_token: "...", expires_in: 120}` | Đến lượt vào khu chọn ghế |
| `POSITION_UPDATED` | `{your_position: N}` | Vị trí cập nhật (người trước rời đi) |
| `SALE_SOLD_OUT` | `{message: "Đã hết vé"}` | Hết vé, hàng chờ đóng |
| `SESSION_EXPIRED` | `{reason: "INACTIVITY_TIMEOUT"}` | Phiên hết hạn do không hoạt động |

