# BF06 — Business Flow: Quét QR & Check-in Tại Cổng Sự Kiện (QR Scanning & Gate Check-in)

## 1. Mục Đích

Tài liệu mô tả chi tiết luồng nghiệp vụ **Soát Vé Tại Cổng Sự Kiện** — bao gồm check-in online, chế độ offline dự phòng, phân quyền cổng theo phân khu và xử lý vé giả mạo.

---

## 2. Luồng Check-in Online (Kết Nối Mạng Bình Thường)

```mermaid
sequenceDiagram
    actor KH as Khán giả
    actor NV as Nhân viên soát vé (Staff)
    participant APP as Gate Staff App
    participant AUTH as Auth Service
    participant CHECKIN as Check-in API
    participant TICKET as Ticket Service
    participant WS as WebSocket (Real-time Stats)

    Note over NV,APP: Đầu ca — Staff đăng nhập

    NV->>APP: Mở Gate Staff App\nĐăng nhập: {staff_username, password}
    APP->>AUTH: POST /auth/login {credentials}
    AUTH-->>APP: Staff JWT Token\n{roles: ["ROLE_STAFF"],\nassigned_concert: concert_id,\nassigned_gates: ["GATE_A_VIP"]}
    APP-->>NV: Hiển thị: "Xin chào [Tên NV]\nCổng: VIP-A | Concert: [Tên concert]"

    Note over KH,NV: Khán giả đến cổng
    KH->>NV: Trình điện thoại hiển thị mã QR vé

    NV->>APP: Hướng camera quét QR code
    APP->>APP: Decode QR → Extract JWT string
    APP->>APP: Verify JWT signature locally\n(RSA Public Key đã cài sẵn)

    alt JWT không hợp lệ (chữ ký sai hoặc format lỗi)
        APP-->>NV: 🔴 VÉ GIẢ MẠO — Báo cáo an ninh
        APP->>CHECKIN: POST /checkin/report-fraud\n{qr_data_raw, gate_id, staff_id}
    else JWT hợp lệ nhưng đã hết hạn (> 60s)
        APP-->>NV: ⚠️ QR ĐÃ HẾT HẠN\n"Yêu cầu khách mở app làm mới mã QR\n(Tự động refresh sau vài giây)"
    else JWT hợp lệ và còn hiệu lực
        APP->>CHECKIN: POST /api/v1/checkin\nBody: {ticket_id, concert_id, gate_id, staff_id, scanned_at}\nHeader: Authorization: Bearer {staff_jwt}

        CHECKIN->>CHECKIN: Xác thực Staff token + Gate permission:\nstaf.assigned_gates.includes(gate_id)?
        CHECKIN->>TICKET: Truy vấn ticket.status

        alt ticket.status = ORDERED
            TICKET->>TICKET: UPDATE tickets\nSET status='CHECKED',\nchecked_in_at=NOW(),\nchecked_by_staff_id=staff_id,\nchecked_at_gate=gate_id

            CHECKIN-->>APP: 200 OK ✅\n{result: "VALID",\nholder_name: "NGUYEN VAN A",\nzone: "VIP_A", seat: "A12",\nchecked_in_at: "2026-10-24T19:35:22+07:00"}

            APP->>APP: Phát âm báo xanh (beep)\nHiển thị màn xanh 2 giây
            APP-->>NV: ✅ HỢP LỆ\n"Nguyễn Văn A — Ghế A12 — Khu VIP-A"

            APP->>WS: Cập nhật stats: +1 checked_in
            WS-->>NV: Cập nhật: "Đã check-in: {N}/{Total}"

        else ticket.status = CHECKED
            CHECKIN-->>APP: 200 OK ❌\n{result: "ALREADY_CHECKED",\nfirst_checkin_at: "19:35:22",\nfirst_gate: "VIP-B"}

            APP->>APP: Phát âm còi đỏ (alert beep)\nHiển thị màn đỏ
            APP-->>NV: ❌ VÉ ĐÃ ĐƯỢC QUÉT\n"Check-in lúc 19:35 tại Cổng VIP-B\n→ Kiểm tra giấy tờ tùy thân"

        else ticket.status = CANCELLED
            CHECKIN-->>APP: 200 OK ❌\n{result: "TICKET_CANCELLED"}
            APP-->>NV: ❌ VÉ ĐÃ BỊ HỦY

        else Sai cổng / Sai concert
            CHECKIN-->>APP: 403 Forbidden\n{result: "WRONG_GATE", message: "Vé này dành cho Cổng Standard-B"}
            APP-->>NV: ❌ SAI CỔNG VÀO\n"Vé này dành cho Cổng Standard-B\nVui lòng hướng dẫn khách đến đúng cổng"
        end
    end
```

---

## 3. Luồng Check-in Offline (Mất Kết Nối Mạng Tại Cổng)

```mermaid
sequenceDiagram
    participant APP as Gate Staff App
    participant LOCAL_DB as Local Encrypted Cache<br/>(SQLite on device)
    participant API as Check-in API (Remote)

    Note over APP: Phát hiện mất kết nối internet
    APP->>APP: Chuyển sang ⚡ OFFLINE MODE\nHiển thị banner: "Đang hoạt động ngoại tuyến"

    Note over APP,LOCAL_DB: Khi có kết nối lần đầu, app đã tải trước
    Note over LOCAL_DB: Pre-downloaded data:\n- Danh sách ticket_id hợp lệ của concert\n- RSA Public Key để verify JWT\n- Danh sách ticket đã checked (sync trước)

    loop Khán giả xếp hàng vào cổng (Offline Mode)
        APP->>APP: Quét QR → Decode JWT
        APP->>APP: Verify JWT signature bằng\nPublic Key lưu local (không cần mạng)

        APP->>LOCAL_DB: SELECT FROM local_checkins\nWHERE ticket_id = scanned_ticket_id

        alt Chưa check-in (offline)
            LOCAL_DB-->>APP: Không có record
            APP->>LOCAL_DB: INSERT INTO local_checkins\n{ticket_id, gate_id, staff_id, ts: NOW(), synced: false}
            APP-->>Staff: ✅ HỢP LỆ (Offline Mode)\n[Hiển thị thông tin vé từ JWT payload]
        else Đã check-in offline tại máy này
            LOCAL_DB-->>APP: {check_time: "19:40:00", gate: "A"}
            APP-->>Staff: ❌ ĐÃ QUÉT (Offline Mode)\n"Quét lúc 19:40 tại cổng này"
        end
    end

    Note over APP,API: Khi có kết nối trở lại
    APP->>API: POST /checkin/sync\nBody: {offline_checkins: [\n  {ticket_id, gate_id, staff_id, scanned_at},\n  ...\n]}

    API->>API: Xử lý từng offline checkin:\n1. Verify ticket.status (lúc sync)
    API->>API: Phát hiện conflict:\nCùng ticket_id đã checked offline ở 2 máy khác nhau

    alt Conflict detected (Possible fraud)
        API->>API: Ghi SECURITY_BREACH_DUPLICATE_ENTRY\nAlert an ninh + Organizer
    else Không conflict
        API->>API: Batch UPDATE tickets status = CHECKED\ncho tất cả offline checkins hợp lệ
    end

    API-->>APP: Sync kết quả:\n{synced: N, conflicts: M, failed: K}
    APP-->>Staff: "Đã đồng bộ {N} vé. Phát hiện {M} vé cần kiểm tra."
```

---

## 4. Phân Quyền Cổng Soát Vé (Gate Permission Management)

```mermaid
sequenceDiagram
    actor ORG as Ban tổ chức
    participant PORTAL as Organizer Portal
    participant STAFF_API as Staff Management API

    ORG->>PORTAL: Vào mục "Quản lý nhân viên"\n→ Thêm tài khoản Staff mới

    ORG->>PORTAL: Nhập thông tin nhân viên:\n- Họ tên, SĐT\n- Email (dùng để đăng nhập app)\n- Cổng được phân công: [GATE_A_VIP, GATE_B_VIP]
    PORTAL->>STAFF_API: POST /organizer/concerts/{id}/staff\nBody: {name, email, assigned_gates:[...]}

    STAFF_API->>STAFF_API: Tạo tài khoản Staff\n(Auto-generate password)\nGhi vào DB với scope:\n{concert_id, allowed_gates}

    STAFF_API-->>PORTAL: Staff account created
    STAFF_API->>STAFF_API: Gửi email onboarding cho nhân viên:\n"Thông tin đăng nhập Gate Staff App:\nEmail: ...\nMật khẩu tạm: ..."
    PORTAL-->>ORG: "Đã tạo 3 tài khoản nhân viên"
```

---

## 5. Dashboard Check-in Real-time Cho Organizer

```mermaid
sequenceDiagram
    participant DASHBOARD as Organizer Dashboard
    participant API as Check-in Stats API
    participant WS as WebSocket

    DASHBOARD->>API: GET /organizer/concerts/{id}/checkin-stats
    API-->>DASHBOARD: {\n  total_tickets: 10000,\n  checked_in: 7234,\n  percentage: 72.34,\n  by_gate: [\n    {gate: "VIP_A", checked: 1200, total: 1500},\n    {gate: "STD_B", checked: 5800, total: 8000}\n  ],\n  last_updated: "19:45:30"\n}

    DASHBOARD->>WS: Subscribe wss://api/ws/checkin-stats/{concert_id}

    loop Mỗi khi có check-in mới
        WS-->>DASHBOARD: {\n  event: "CHECKIN_UPDATE",\n  checked_in: 7235,\n  percentage: 72.35,\n  latest_gate: "STD_B"\n}
        DASHBOARD->>DASHBOARD: Cập nhật số liệu real-time\n(Không cần F5)
    end
```

---

## 6. Xử Lý Các Tình Huống Đặc Biệt Tại Cổng

| Tình huống | Hiển thị trên App | Hành động nhân viên | BR liên quan |
|---|---|---|---|
| Vé hợp lệ lần đầu | ✅ Xanh: "HỢP LỆ — [Tên khách] — [Ghế] — [Zone]" | Cho phép vào | BR-TCK-003 |
| Vé đã quét trước đó | ❌ Đỏ: "ĐÃ QUÉT lúc [HH:mm] tại Cổng [X]" | Yêu cầu CCCD; Giữ lại người; Báo Tổ trưởng | BR-TCK-003 |
| Vé đã bị hủy | ❌ Cam: "VÉ ĐÃ BỊ HỦY — Lý do: [hoàn tiền/concert hủy]" | Không cho vào | BR-RFD-003 |
| Sai cổng vào | ⚠️ Vàng: "SAI CỔNG — Vé dành cho Cổng [X]" | Hướng dẫn khách đến đúng cổng | BR-TCK-004 |
| QR hết hạn 60s | ⚠️ Cam: "QR HẾT HẠN — Yêu cầu khách làm mới" | Yêu cầu khách mở app refresh | BR-TCK-002 |
| QR giả mạo (chữ ký sai) | 🔴 Đỏ đậm: "VÉ GIẢ MẠO — CẢNH BÁO AN NINH" | Giữ người lại; Gọi an ninh ngay; Không tự xử lý | BR-TCK-002 |
| Chưa đến giờ mở cửa | ⚠️ Xám: "CHƯA ĐẾN GIỜ MỞ CỬA — [HH:mm]" | Yêu cầu khách đợi | BR-TCK-007 |
| Mất mạng tại cổng | ⚡ Offline Mode kích hoạt tự động | Tiếp tục soát vé; Đồng bộ sau | BR-TCK-005 |
