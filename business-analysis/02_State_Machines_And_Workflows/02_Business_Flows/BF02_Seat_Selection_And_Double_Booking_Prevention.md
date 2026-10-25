# BF02 — Business Flow: Chọn Ghế & Ngăn Chặn Double-Booking (Seat Selection & Double-Booking Prevention)

## 1. Mục Đích

Tài liệu mô tả chi tiết luồng nghiệp vụ **Chọn Ghế** trên sơ đồ khán phòng tương tác thời gian thực, bao gồm cơ chế **ngăn chặn đặt trùng ghế (Zero Double-Booking)** dưới áp lực tải cao đồng thời.

---

## 2. Tổng Quan Luồng (Flow Overview)

```mermaid
flowchart LR
    A["User nhận\nQueue Access Token"] --> B["Tải sơ đồ ghế\nreal-time"]
    B --> C["Chọn 1-4 ghế\ntrên sơ đồ"]
    C --> D["API giữ ghế\n(Hold)"]
    D --> E{Thành công?}
    E -->|Có| F["Tạo đơn hàng\n(Order)"]
    E -->|Không — Conflict| G["Thông báo ghế đã bị\nngười khác chọn"]
    G --> C
    F --> H["Thanh toán\n(Luồng BF03)"]
```

---

## 3. Chi Tiết Từng Bước

### Bước 1: Tải Sơ Đồ Ghế Thời Gian Thực

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant APP as Web/Mobile App
    participant API as Seat Map API
    participant CACHE as Redis Cache (Seat Map)
    participant DB as Database
    participant WS as WebSocket Server

    C->>APP: Vào trang chọn ghế\n(Đã có X-Queue-Token hợp lệ)
    APP->>API: GET /concerts/{id}/seat-map\nHeaders: X-Queue-Token: {token}

    API->>API: Xác thực Queue Access Token\n(JWT verify + Redis TTL check)
    API->>CACHE: GET seat_map:{concert_id}

    alt Cache hit (< 500ms stale)
        CACHE-->>API: Snapshot trạng thái ghế (JSON)
        API-->>APP: 200 OK — Seat Map Snapshot
    else Cache miss hoặc stale
        API->>DB: SELECT id, row, col, zone, status, price FROM seats\nWHERE concert_id = ?
        DB-->>API: Danh sách tất cả ghế + trạng thái hiện tại
        API->>CACHE: SETEX seat_map:{concert_id} 5 {snapshot}
        API-->>APP: 200 OK — Seat Map Snapshot
    end

    APP->>WS: Kết nối WebSocket\nwss://api/ws/seat-map/{concert_id}
    WS-->>APP: Subscribe thành công\nNhận updates real-time khi ghế thay đổi trạng thái
```

### Bước 2: Chọn Ghế và Khóa Ghế (Atomic Hold)

```mermaid
sequenceDiagram
    actor C as Khách hàng (Người dùng A)
    participant APP as Web App
    participant API as Seat Hold API
    participant REDIS as Redis Cluster
    participant DB as Database
    participant WS as WebSocket Server

    C->>APP: Click chọn ghế A12 trên sơ đồ
    APP->>API: POST /concerts/{id}/seats/hold\nBody: {seat_ids: ["seat_A12"]}\nHeaders: X-Queue-Token: {token}, Authorization: Bearer {jwt}

    Note over API: Xác thực đồng thời:<br/>1. JWT hợp lệ<br/>2. X-Queue-Token còn TTL<br/>3. Số ghế trong request ≤ giới hạn

    API->>REDIS: SET lock:seat:A12 {user_id} NX EX 600
    Note over REDIS: Atomic SET: Chỉ thành công<br/>nếu key chưa tồn tại (NX)

    alt Redis lock thành công
        REDIS-->>API: OK

        API->>DB: UPDATE seats\nSET status='HELD', hold_user_id='{user_id}',\n    hold_started_at=SYSUTCDATETIME(), version=version+1\nWHERE id='seat_A12'\n  AND status='AVAILABLE'\n  AND version = {current_version}
        Note over DB: Optimistic Lock:<br/>Bảo vệ tầng 2 chống race condition

        alt DB update thành công (1 row affected)
            DB-->>API: 1 row updated
            API->>WS: Publish SeatHeldEvent:\n{concert_id, seat_id:"A12", new_status:"HELD"}
            WS-->>APP: Broadcast tới tất cả user đang xem sơ đồ ghế
            API-->>APP: 200 OK\n{held_seats: [{seat_id:"A12", zone:"VIP", price:2000000}],\n expires_in: 600}
            APP-->>C: Ghế A12 đổi màu vàng trên sơ đồ\n"Đã giữ ghế A12 — còn 10:00 phút"
        else DB update thất bại (0 rows — version conflict)
            DB-->>API: 0 rows updated
            API->>REDIS: DEL lock:seat:A12 (dọn dẹp lock bị lỗi)
            API-->>APP: 409 Conflict\n{error:"SEAT_CONFLICT", message:"Ghế A12 vừa bị người khác chọn. Vui lòng chọn ghế khác."}
            APP-->>C: Sơ đồ tự cập nhật — Ghế A12 chuyển đỏ
        end

    else Redis lock thất bại (key đã tồn tại)
        REDIS-->>API: NIL
        API-->>APP: 409 Conflict\n{error:"SEAT_ALREADY_HELD", code:"BR-SEAT-001"}
        APP-->>C: "Ghế A12 đang được người khác giữ. Vui lòng chọn ghế khác."
    end
```

### Bước 3: Cập Nhật Sơ Đồ Real-time Cho Tất Cả Người Dùng

```mermaid
sequenceDiagram
    participant WS_SERVER as WebSocket Server<br/>(Pub/Sub)
    participant USER_1 as Người dùng 1<br/>(đang giữ ghế A12)
    participant USER_2 as Người dùng 2<br/>(đang xem sơ đồ)
    participant USER_5000 as Người dùng 5000<br/>(đang xem sơ đồ)

    Note over WS_SERVER: Nhận SeatHeldEvent từ Seat Hold API

    par Broadcast đến tất cả subscribers
        WS_SERVER-->>USER_1: {event:"seat.held", seat_id:"A12", status:"HELD"}
        WS_SERVER-->>USER_2: {event:"seat.held", seat_id:"A12", status:"HELD"}
        WS_SERVER-->>USER_5000: {event:"seat.held", seat_id:"A12", status:"HELD"}
    end

    Note over USER_2,USER_5000: Giao diện tự động cập nhật:<br/>Ghế A12 chuyển sang màu vàng<br/>Không thể click vào ghế này nữa
```

### Bước 4: Bỏ Chọn Ghế (Release Hold)

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant APP as Web App
    participant API as Seat Hold API
    participant REDIS as Redis
    participant DB as Database
    participant WS as WebSocket Server

    C->>APP: Click "Bỏ chọn" ghế A12
    APP->>API: DELETE /concerts/{id}/seats/hold/{seat_id}\nHeaders: Authorization: Bearer {jwt}

    API->>REDIS: GET lock:seat:A12
    REDIS-->>API: {user_id} (kiểm tra đúng chủ sở hữu lock)

    alt Đúng chủ sở hữu
        API->>REDIS: DEL lock:seat:A12
        API->>DB: UPDATE seats SET status='AVAILABLE',\nhold_user_id=NULL, hold_started_at=NULL\nWHERE id='seat_A12' AND hold_user_id='{user_id}'
        DB-->>API: 1 row updated
        API->>WS: Publish SeatReleasedEvent
        WS-->>APP: Broadcast — Ghế A12 → AVAILABLE
        API-->>APP: 200 OK
        APP-->>C: Ghế A12 chuyển xanh lá lại trên sơ đồ
    else Không phải chủ sở hữu (bảo mật)
        API-->>APP: 403 Forbidden
    end
```

---

## 4. Xử Lý Tự Động Hết Hạn Giữ Ghế (Auto-Release on Timeout)

```mermaid
flowchart TD
    A["Khách hàng giữ ghế A12\nRedis key: lock:seat:A12\nTTL = 600 giây"] --> B{{"600 giây đã trôi qua?"}}

    B -->|Khách hàng đã thanh toán trước| C["Thanh toán IPN thành công\n→ seat.status = SOLD\nDEL lock:seat:A12 (xóa chủ động)"]

    B -->|Hết 600s, chưa thanh toán| D["Redis Keyspace Notification:\nkey lock:seat:A12 EXPIRED"]

    D --> E["Seat Release Worker nhận event"]
    E --> F["UPDATE seats SET status='AVAILABLE',\nhold_user_id=NULL\nWHERE id=seat_id AND status='HELD'\nAND hold_user_id=expired_user_id"]
    F --> G["UPDATE orders SET status='OUT_TIME'\nWHERE seat ghế = seat_id\nAND order.status = 'PENDING'"]
    G --> H["Publish SeatReleasedEvent\n→ WebSocket broadcast\n→ Ghế xuất hiện lại trên sơ đồ của tất cả user"]
    H --> I["Notification: Gửi thông báo cho user\n'Phiên giữ ghế đã hết hạn. Vui lòng chọn lại.'"]

    C --> J["SOLD - Ghế thuộc về khách hàng vĩnh viễn"]
```

---

## 5. Giới Hạn Số Ghế Tối Đa Trên Mỗi Đơn Hàng (Per-Order Seat Limit Guard)

```mermaid
flowchart LR
    A["API nhận request giữ ghế\n{seat_ids: [A12, A13, A14, A15, A16]}"] --> B["Đếm số ghế trong request:\nCOUNT = 5"]
    B --> C{count <= concert.max_seats_per_order\n(mặc định 4)?}
    C -->|Có ≤ 4| D["Tiếp tục xử lý giữ ghế"]
    C -->|Không > 4| E["Trả lỗi ngay lập tức:\n400 Bad Request\n{error:'SEAT_LIMIT_EXCEEDED',\ncode:'BR-SEAT-004',\nmessage:'Bạn chỉ được chọn tối đa 4 ghế'}"]

    D --> F["Kiểm tra tổng vé đang sở hữu:\nSELECT COUNT(*) FROM tickets t\nJOIN orders o ON t.order_id = o.id\nWHERE o.customer_id = user_id\nAND o.concert_id = concert_id\nAND o.status IN ('PAID','PENDING')"]
    F --> G{total_owned + new_request <= max_per_account?}
    G -->|Vượt giới hạn| H["400 Bad Request\n{error:'ACCOUNT_SEAT_LIMIT_EXCEEDED',\ncode:'BR-ORD-003'}"]
    G -->|Trong giới hạn| I["Tiến hành lock từng ghế một\n(theo thứ tự seat_id để tránh deadlock)"]
```

---

## 6. Phòng Chống Double-Booking — Tóm Tắt Defense Layers

| Layer | Cơ chế | Độ trễ | Hiệu quả |
|---|---|---|---|
| **Layer 1 — CDN** | Rate limiting per IP/user | < 1ms | Chặn bot, F5 spam trước khi vào API |
| **Layer 2 — API Auth** | Queue Access Token validation (Redis TTL) | < 5ms | Chỉ user có lượt mới được gọi Seat Hold API |
| **Layer 3 — Redis Distributed Lock** | `SET NX EX` — Atomic; chỉ 1 user giành lock | < 5ms | Ngăn đồng thời ở tầng cache; phản hồi nhanh |
| **Layer 4 — Database Optimistic Lock** | `WHERE status='AVAILABLE' AND version=N` | < 20ms | Đảm bảo tính nhất quán dữ liệu dù Redis fail |
| **Layer 5 — Audit Log** | Ghi nhận tất cả attempt HOLD, thành công và thất bại | Async | Phục vụ điều tra tranh chấp và bảo mật |

