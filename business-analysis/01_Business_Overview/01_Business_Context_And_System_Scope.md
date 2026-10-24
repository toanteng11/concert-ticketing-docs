# Bối Cảnh Kinh Doanh & Phạm Vi Hệ Thống (Business Context & System Scope)

## 1. Bối Cảnh Kinh Doanh (Business Context)

### 1.1 Vấn Đề Thị Trường (Market Problem)

Thị trường âm nhạc trực tiếp (Live Entertainment) tại Việt Nam đang bước vào giai đoạn tăng trưởng mạnh sau đại dịch COVID-19. Các sự kiện âm nhạc quy mô lớn — từ concert nghệ sĩ nội địa hàng đầu đến show diễn nghệ sĩ quốc tế — thu hút hàng chục ngàn khán giả tham dự và hàng trăm nghìn người mua vé. Tuy nhiên, hạ tầng bán vé hiện tại đang bộc lộ những vấn đề nghiêm trọng:

| Vấn đề | Biểu hiện thực tế | Hệ quả |
|---|---|---|
| **Quá tải hệ thống tại thời điểm mở bán** | Server sập, trang web không phản hồi trong 5–15 phút đầu khi mở bán | Khách hàng mất vị trí mua vé, trust nền tảng giảm mạnh |
| **Thiếu công bằng trong xếp hàng** | Người biết kỹ thuật dùng bot tự động hoặc F5 liên tục để giành ghế | Bất công với khách hàng thực sự |
| **Trải nghiệm người dùng kém** | Giao diện chậm, không cập nhật sơ đồ ghế thời gian thực | Khách chọn ghế đã bán, bực bội, bỏ giỏ hàng |
| **Vé giả & đầu cơ vé chợ đen** | Mã QR không bảo mật, bị sao chép; một vé nhiều người cùng vào cửa | Thiệt hại doanh thu, kiện cáo pháp lý |
| **Phân mảnh kênh bán vé** | Mỗi concert bán qua các kênh khác nhau (Shopee, MoMo App, trang riêng của BTC) | Không có dữ liệu tập trung, khó quản lý |
| **Thiếu công cụ cho Ban tổ chức** | BTC không có Dashboard real-time, không có công cụ check-in chuyên nghiệp | Lộn xộn tại cổng vào, check-in thủ công chậm chạp |

### 1.2 Lý Do Xây Dựng Hệ Thống (Rationale for Building)

Từ các vấn đề trên, bài toán đặt ra là xây dựng một **nền tảng bán vé concert trực tuyến tập trung, quy mô lớn, công bằng và bảo mật**, với kiến trúc đủ mạnh để phục vụ đồng thời **2 triệu người dùng đăng ký** và chịu được các **đợt traffic đỉnh** (peak load) hàng trăm nghìn request mỗi giây vào thời điểm mở bán.

Hệ thống phải giải quyết được 3 thách thức cốt lõi:
1. **Scalability (Khả năng mở rộng):** Chịu tải đỉnh khi hàng trăm nghìn người đăng nhập cùng lúc.
2. **Fairness (Tính công bằng):** Cơ chế phòng chờ ảo loại bỏ lợi thế của bot và F5 liên tục.
3. **Integrity (Tính toàn vẹn):** Zero double-booking, vé QR mã hóa động chống sao chép.

### 1.3 Mục Tiêu Kinh Doanh Cụ Thể (Business Goals)

| ID | Mục tiêu | Chỉ số đo lường thành công |
|---|---|---|
| BG-01 | Nền tảng bán vé tập trung dành cho thị trường Việt Nam | ≥ 10 concert quy mô lớn (>10,000 vé) mỗi tháng trong năm đầu |
| BG-02 | Đảm bảo công bằng cho khách hàng qua phòng chờ ảo | Tỷ lệ khiếu nại "mua vé không công bằng" < 1% |
| BG-03 | Hệ thống không sập trong đợt mở bán đỉnh | Uptime ≥ 99.95% trong giờ mở bán; P99 latency < 2 giây |
| BG-04 | Zero double-booking | Tỷ lệ lỗi trùng ghế = 0% |
| BG-05 | Vé điện tử bảo mật, chống giả mạo | Tỷ lệ vé giả bị phát hiện qua cổng = 0% |
| BG-06 | Tạo doanh thu nền tảng qua phí dịch vụ | Phí dịch vụ 5–10% trên mỗi vé bán thành công |
| BG-07 | Công cụ vận hành chuyên nghiệp cho Ban tổ chức | BTC tự vận hành được không cần hỗ trợ kỹ thuật trong ngày diễn |

---

## 2. Context Diagram — Ranh Giới Hệ Thống & Các Tác Nhân Bên Ngoài

```mermaid
C4Context
    title System Context Diagram — Concert Ticketing Platform

    Person(customer, "Khách Hàng<br/>(Customer)", "Người dùng cá nhân mua vé\ntrên Web, iOS, Android")
    Person(organizer, "Ban Tổ Chức<br/>(Organizer)", "Tạo concert, thiết lập\nsơ đồ ghế, theo dõi doanh thu")
    Person(admin, "Quản Trị Viên<br/>(Admin)", "Phê duyệt concert,\ngiám sát vận hành")
    Person(staff, "Nhân Viên Soát Vé<br/>(Gate Staff)", "Quét QR kiểm tra vé\ntại cổng vào sự kiện")

    System_Boundary(platform, "Concert Ticketing Platform") {
        System(cts, "Concert Ticketing System", "Hệ thống bán vé concert\ntrực tuyến tập trung")
    }

    System_Ext(vnpay, "VNPay", "Cổng thanh toán nội địa:\nThẻ ATM, QR Pay, Visa/Mastercard")
    System_Ext(momo, "MoMo", "Ví điện tử &\nCổng thanh toán MoMo")
    System_Ext(fcm_apns, "FCM / APNs", "Dịch vụ thông báo đẩy\nAndroid (FCM) & iOS (APNs)")
    System_Ext(email_svc, "Email Service\n(SendGrid / SES)", "Gửi email xác nhận,\nvé điện tử, thông báo")
    System_Ext(sms_svc, "SMS Gateway\n(Twilio / VNPT SMS)", "Gửi OTP xác thực,\nthông báo SMS")
    System_Ext(id_provider, "Identity Provider\n(Google / Apple ID)", "Đăng nhập OAuth2/OIDC\ntừ bên thứ ba")
    System_Ext(cdn, "CDN\n(Cloudflare)", "Phục vụ tài nguyên tĩnh,\nWAF, Waiting Room Edge")

    Rel(customer, cts, "Xem concert, vào phòng chờ,\nchọn ghế, thanh toán, nhận vé QR", "HTTPS / WebSocket")
    Rel(organizer, cts, "Tạo concert, thiết kế sơ đồ,\ntheo dõi dashboard", "HTTPS")
    Rel(admin, cts, "Phê duyệt, giám sát,\nxử lý sự cố", "HTTPS")
    Rel(staff, cts, "Quét vé QR tại cổng\ntheo ca sự kiện", "HTTPS / Offline QR")

    Rel(cts, vnpay, "Khởi tạo giao dịch thanh toán\n& nhận IPN webhook", "HTTPS API")
    Rel(cts, momo, "Khởi tạo giao dịch thanh toán\n& nhận IPN webhook", "HTTPS API")
    Rel(cts, fcm_apns, "Gửi push notification:\nhàng chờ, thanh toán, vé", "HTTPS API")
    Rel(cts, email_svc, "Gửi email xác nhận đơn hàng,\nvé điện tử, hoàn tiền", "SMTP / API")
    Rel(cts, sms_svc, "Gửi OTP đăng ký,\nthông báo hết vé/hủy concert", "HTTP API")
    Rel(id_provider, cts, "Cung cấp OAuth2 Token\nđể xác thực người dùng", "OAuth2 / OIDC")
    Rel(cdn, cts, "Proxy request, WAF filter,\nWaiting Room Edge Logic", "TCP / HTTPS")
```

---

## 3. Mô Tả Tổng Quan Luồng Tương Tác Chính (High-Level Interaction Flows)

### Luồng 1: Khách Hàng → Mua Vé → Nhận Vé QR

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant WR as Waiting Room<br/>(Phòng chờ ảo)
    participant SS as Seat Selection<br/>(Chọn ghế)
    participant OR as Order Service
    participant PG as Payment Gateway<br/>(VNPay / MoMo)
    participant TK as Ticket Service
    participant NT as Notification Service

    Note over C,WR: T-15 phút trước giờ mở bán
    C->>WR: Truy cập concert → Vào phòng chờ
    WR-->>C: Hiển thị đồng hồ đếm ngược

    Note over WR,SS: Đúng giờ mở bán (T0)
    WR->>WR: Shuffle ngẫu nhiên toàn bộ người trong phòng
    WR-->>C: Cấp Queue Position + Access Token

    Note over C,SS: Đến lượt (≤ 5.000 người đồng thời)
    C->>SS: Chọn ghế trên sơ đồ real-time
    SS->>SS: Lock ghế (Redis Distributed Lock, TTL 10 phút)
    SS-->>C: Xác nhận giữ ghế thành công

    C->>OR: Tạo đơn hàng (ghế + thông tin cá nhân)
    OR-->>C: Trả về thông tin đơn hàng + countdown 10 phút

    C->>PG: Chọn VNPay hoặc MoMo → Chuyển hướng thanh toán
    PG-->>C: Màn hình thanh toán của Gateway

    C->>PG: Hoàn tất thanh toán (nhập PIN/OTP)
    PG->>OR: IPN Webhook thông báo thanh toán thành công
    OR->>OR: Xác thực HMAC chữ ký + khớp số tiền
    OR->>SS: Chốt ghế → seat.status = SOLD
    OR->>TK: Tạo vé điện tử (ticket.status = ORDERED)
    TK->>TK: Sinh Dynamic QR Code (JWT/HMAC)
    TK->>NT: Trigger gửi thông báo

    NT-->>C: Email + Push Notification: "Vé của bạn đã sẵn sàng"
    TK-->>C: Hiển thị vé + QR Code trên App/Web
```

### Luồng 2: Ban Tổ Chức → Tạo Concert → Mở Bán

```mermaid
flowchart LR
    A["Organizer tạo hồ sơ concert\n(Thông tin, sơ đồ, giá vé)"] --> B["Gửi hồ sơ\n→ concert.status = PENDING_REVIEW"]
    B --> C{"Admin xét duyệt\ntrong 48 giờ"}
    C -->|Phê duyệt| D["concert.status = PUBLISHED\nHiển thị công khai, chưa mở bán"]
    C -->|Yêu cầu bổ sung| E["Organizer chỉnh sửa\nvà gửi lại"]
    E --> C
    D --> F["T-15 phút: Waiting Room kích hoạt tự động"]
    F --> G["T0: concert.status = ON_SALE\nShuffle ngẫu nhiên hàng chờ"]
    G --> H{"Vé còn không?"}
    H -->|Hết vé| I["concert.status = SOLD_OUT\nHàng chờ đóng + Thông báo"]
    H -->|Còn vé| G
    I --> J["T+24h sau khi biểu diễn xong\nconcert.status = COMPLETED"]
```

### Luồng 3: Nhân Viên Soát Vé → Check-in tại Cổng

```mermaid
sequenceDiagram
    actor S as Nhân viên (Staff)
    participant APP as Gate Staff App
    participant API as Check-in API
    participant TK as Ticket Service

    S->>APP: Đăng nhập bằng tài khoản Staff<br/>(được Organizer cấp)
    APP->>API: Xác thực token + gán ca / cổng vào

    Note over S,APP: Khách hàng trình vé QR
    S->>APP: Quét mã QR của khán giả
    APP->>API: POST /checkin {qr_payload, gate_id, staff_id}
    API->>API: Xác thực chữ ký QR + Kiểm tra TTL
    API->>TK: Truy vấn ticket.status

    alt Vé hợp lệ (status = ORDERED)
        TK-->>API: Trả về thông tin vé
        API->>TK: UPDATE ticket SET status = CHECKED,<br/>checked_in_at = NOW()
        API-->>APP: ✅ HỢP LỆ — Tên: Nguyen Van A | Hạng: VIP | Ghế: A12
        APP-->>S: Hiển thị màn hình xanh + tiếng bíp xanh
    else Vé đã check-in trước đó (status = CHECKED)
        TK-->>API: Đã check-in lúc 19:35:22 tại Cổng B
        API-->>APP: ❌ ĐÃ QUÉT — Check-in lúc 19:35 tại Cổng VIP-B
        APP-->>S: Hiển thị màn hình đỏ + tiếng còi cảnh báo
    else Vé bị hủy hoặc không hợp lệ
        API-->>APP: ❌ VÉ KHÔNG HỢP LỆ — Đã bị hủy
        APP-->>S: Hiển thị màn hình cam + cảnh báo
    end
```

---

## 4. Định Nghĩa Ranh Giới Hệ Thống (System Boundary Definition)

### 4.1 Những gì NẰM TRONG phạm vi hệ thống (In Scope)

| Phạm vi | Mô tả |
|---|---|
| **Phòng chờ ảo & Xếp hàng** | Cơ chế Waiting Room T-15 phút, Shuffle ngẫu nhiên T0, FIFO người đến sau, Giới hạn 5.000 CCU trong khu chọn ghế |
| **Sơ đồ ghế tương tác** | Hiển thị sơ đồ khán phòng thời gian thực, màu sắc phân biệt AVAILABLE/HELD/SOLD/BLOCKED, cập nhật qua WebSocket |
| **Đặt vé & Thanh toán** | Tạo đơn hàng, tích hợp VNPay và MoMo, xử lý IPN webhook, Idempotency Key |
| **Vé điện tử & QR Code** | Phát hành vé tức thì sau thanh toán, Dynamic QR TOTP/JWT, PDF vé gửi email |
| **Check-in cổng vào** | App Staff mobile (iOS/Android), quét QR online/offline, đồng bộ dữ liệu check-in |
| **Quản lý Concert** | Organizer Portal: Tạo concert, thiết kế sơ đồ ghế, giá vé, Dashboard real-time |
| **Phê duyệt & Vận hành** | Admin Dashboard: Phê duyệt concert, giám sát hệ thống, xử lý sự cố |
| **Hoàn tiền & Hủy vé** | Quy trình hủy đơn, hoàn tiền theo chính sách phân bậc, hủy concert hàng loạt |
| **Đánh giá & Nhận xét** | Chức năng review sau concert dành cho khách đã tham dự |
| **Thông báo đa kênh** | Push notification, Email, SMS cho các sự kiện: vé, thanh toán, hủy, nhắc nhở |
| **Báo cáo & Quyết toán** | Báo cáo doanh thu cho Organizer, đối soát giao dịch Admin, giải ngân sau concert |

### 4.2 Những gì KHÔNG NẰM TRONG phạm vi hệ thống (Out of Scope)

| Phạm vi | Lý do loại trừ |
|---|---|
| **Ứng dụng phát trực tiếp (Live Streaming)** | Không phải dịch vụ nền tảng phân phối nội dung — thuộc về Organizer tự cấu hình kênh phát sóng riêng |
| **Quản lý thiết bị vật lý tại sự kiện** | Camera an ninh, âm thanh, ánh sáng — thuộc chuỗi vận hành offline của Organizer |
| **Nền tảng mua bán vé thứ cấp (Secondary Market / Resale)** | Chưa trong phạm vi phiên bản 1; sẽ đánh giá thêm ở roadmap sau |
| **Chức năng bảo hiểm vé (Ticket Insurance)** | Sản phẩm tài chính, cần đối tác bảo hiểm riêng |
| **Quản lý hợp đồng nghệ sĩ** | Quan hệ hợp đồng giữa BTC và nghệ sĩ nằm ngoài phạm vi hệ thống công nghệ |
| **Dịch vụ đặt lịch vận chuyển, lưu trú đi kèm** | Nằm ngoài scope; có thể tích hợp đối tác qua deep link trong tương lai |
| **Hệ thống kế toán - ERP nội bộ của Organizer** | Nền tảng cung cấp báo cáo doanh thu chuẩn hóa; Organizer tự import vào ERP của mình |

### 4.3 Ranh Giới Tích Hợp Hệ Thống Bên Ngoài (External Integration Boundary)

```mermaid
graph TB
    subgraph IN["Trong phạm vi hệ thống (In Scope)"]
        direction LR
        CTS["Concert Ticketing System\n(Core Platform)"]
    end

    subgraph EXT["Hệ thống bên ngoài (Out of Scope — External Dependencies)"]
        direction LR
        VNP["VNPay API"]
        MOM["MoMo API"]
        FCM["Firebase FCM / APNs"]
        SG["SendGrid / AWS SES"]
        SMS["Twilio / VNPT SMS"]
        GG["Google / Apple OAuth2"]
        CF["Cloudflare CDN / WAF"]
    end

    CTS <-->|"Thanh toán:\nGọi API + Nhận IPN Webhook"| VNP
    CTS <-->|"Thanh toán:\nGọi API + Nhận IPN Webhook"| MOM
    CTS -->|"Gửi push notification"| FCM
    CTS -->|"Gửi email vé & thông báo"| SG
    CTS -->|"Gửi SMS OTP & nhắc nhở"| SMS
    GG -->|"Cung cấp Access Token\nOAuth2 OIDC"| CTS
    CF -->|"Proxy & WAF & Edge Waiting Room"| CTS
```

---

## 5. Các Ràng Buộc & Giả Định Quan Trọng (Constraints & Assumptions)

### 5.1 Ràng Buộc Kỹ Thuật

| ID | Ràng buộc | Chi tiết |
|---|---|---|
| TC-01 | Quy mô người dùng | 2 triệu tài khoản đăng ký; đỉnh CCU đồng thời trong giờ mở bán có thể lên đến 500.000 người |
| TC-02 | Giới hạn khu chọn ghế | Tối đa 5.000 CCU trong khu chọn ghế của 1 concert tại 1 thời điểm |
| TC-03 | SLA Uptime | Uptime tối thiểu 99.95% đối với luồng mua vé; 99.9% với các tính năng phụ |
| TC-04 | Độ trễ giữ ghế | API khóa ghế phải hoàn thành trong < 500ms ở P95 dưới tải đỉnh |
| TC-05 | Đa nền tảng | Hỗ trợ đồng thời Web (PWA-ready), iOS App native và Android App native |
| TC-06 | Múi giờ | Toàn bộ dữ liệu lưu trữ UTC; hiển thị UI theo múi giờ ICT (UTC+7) |
| TC-07 | Tiền tệ | Chỉ hỗ trợ VNĐ trong phiên bản 1.0 |

### 5.2 Giả Định Dự Án

| ID | Giả định |
|---|---|
| AS-01 | Organizer đã ký hợp đồng hợp tác với nền tảng trước khi được cấp tài khoản và tạo concert |
| AS-02 | Cổng thanh toán VNPay và MoMo đã hoàn tất thủ tục ký kết và cấp API credentials cho môi trường Production |
| AS-03 | Hệ thống không phải cơ quan cấp vé giấy vật lý; toàn bộ vé là vé điện tử (E-Ticket) kèm mã QR |
| AS-04 | Mạng internet tại địa điểm tổ chức sự kiện được đảm bảo đủ băng thông cho check-in; cơ chế Offline là dự phòng khẩn cấp |
| AS-05 | Khán giả có điện thoại thông minh hiển thị màn hình; tính năng in vé PDF là phương án dự phòng cho người cao tuổi |
| AS-06 | Admin sẽ xử lý phê duyệt concert trong SLA 48 giờ; hệ thống cảnh báo nếu quá hạn |

