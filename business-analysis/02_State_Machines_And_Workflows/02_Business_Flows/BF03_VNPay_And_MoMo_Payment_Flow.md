# BF03 — Business Flow: Luồng Thanh Toán VNPay & MoMo (Payment Flow)

## 1. Mục Đích

Tài liệu mô tả chi tiết luồng tích hợp thanh toán qua hai cổng **VNPay** và **MoMo**, bao gồm khởi tạo giao dịch, xử lý IPN Webhook an toàn, và tất cả các trường hợp ngoại lệ.

---

## 2. Tổng Quan Luồng Thanh Toán

```mermaid
flowchart LR
    A["Khách hàng\ntạo đơn hàng\n(order = PENDING)"] --> B["Chọn phương thức\nthanh toán"]
    B --> C{VNPay hay MoMo?}
    C -->|VNPay| D["Tạo VNPay Payment URL\n+ chuyển hướng khách hàng"]
    C -->|MoMo| E["Tạo MoMo Payment Request\n+ chuyển hướng / deep link"]
    D --> F["Khách hàng hoàn tất\nthanh toán tại cổng"]
    E --> F
    F --> G["Cổng gửi IPN Webhook\nvề hệ thống"]
    G --> H["Xác thực HMAC +\nKhớp số tiền"]
    H --> I{"Hợp lệ?"}
    I -->|Thành công| J["Cập nhật: order=PAID\nticket=ORDERED\nseat=SOLD"]
    I -->|Thất bại / Gian lận| K["Ghi Alert\nKhông cập nhật đơn"]
```

---

## 3. Luồng Chi Tiết VNPay

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant APP as Web/Mobile App
    participant PAY_API as Payment API
    participant VNPAY as VNPay Gateway
    participant DB as Database

    C->>APP: Chọn "Thanh toán qua VNPay"\nClick nút Tiến hành thanh toán
    APP->>PAY_API: POST /payments/initiate\nBody: {order_id, gateway:"VNPAY", return_url, ip_addr}

    PAY_API->>DB: INSERT payment_transactions\n{status:"INITIATED", gateway:"VNPAY", amount:X}

    PAY_API->>PAY_API: Tạo VNPay Payment URL:\nvnp_Amount = order.total_amount * 100\nvnp_TxnRef = payment_transaction_id\nvnp_OrderInfo = "Thanh toan ve concert"\nvnp_ReturnUrl = "{system}/payments/vnpay/return"\nvnp_IpAddr = user_ip\nvnp_CreateDate = NOW()\nvnp_SecureHash = HMACSHA512(secret_key, query_params)

    PAY_API-->>APP: {payment_url: "https://sandbox.vnpayment.vn/paymentv2/...", expires_in: 600}

    APP-->>C: Chuyển hướng đến trang thanh toán VNPay
    C->>VNPAY: Khách hàng nhập thông tin thanh toán\n(OTP ngân hàng, PIN ví, quét QR...)
    VNPAY-->>C: Kết quả thanh toán tại cổng

    Note over VNPAY,PAY_API: VNPay gửi IPN đồng thời 2 kênh

    par Kênh 1: IPN Server-to-Server (Bất đồng bộ - tin cậy)
        VNPAY->>PAY_API: POST /webhooks/vnpay/ipn\n{vnp_TxnRef, vnp_Amount, vnp_ResponseCode,\nvnp_TransactionNo, vnp_SecureHash, ...}
        PAY_API->>PAY_API: Xác thực HMAC SHA512
        PAY_API->>DB: Cập nhật trạng thái đơn hàng
        PAY_API-->>VNPAY: {"RspCode":"00","Message":"Confirm Success"}
    and Kênh 2: Return URL (Đồng bộ - hiển thị kết quả cho user)
        VNPAY-->>APP: Redirect về {return_url}?vnp_ResponseCode=00&...
        APP->>PAY_API: GET /payments/vnpay/return?{query_params}
        PAY_API-->>APP: Redirect tới trang kết quả mua vé
        APP-->>C: "Mua vé thành công! Vé của bạn đã sẵn sàng."
    end
```

---

## 4. Luồng Chi Tiết MoMo

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant APP as Web/Mobile App
    participant PAY_API as Payment API
    participant MOMO as MoMo Gateway

    C->>APP: Chọn "Thanh toán qua MoMo"
    APP->>PAY_API: POST /payments/initiate\nBody: {order_id, gateway:"MOMO"}

    PAY_API->>MOMO: POST https://payment.momo.vn/v2/gateway/api/create\n{\n  partnerCode: "PARTNER_CODE",\n  requestId: payment_txn_id,\n  amount: order.total_amount,\n  orderId: order_id,\n  orderInfo: "Mua ve concert",\n  redirectUrl: "{system}/payments/momo/return",\n  ipnUrl: "{system}/webhooks/momo/ipn",\n  requestType: "payWithMethod",\n  signature: HMAC_SHA256\n}

    MOMO-->>PAY_API: {payUrl: "https://payment.momo.vn/...", deeplink: "momo://...", qrCodeUrl}

    PAY_API-->>APP: {pay_url, deeplink, qr_code_url}

    alt Trên Mobile App
        APP-->>C: Deep link mở ứng dụng MoMo tự động\n(momo://)
    else Trên Web
        APP-->>C: Hiển thị QR code MoMo\nHOẶC chuyển hướng sang trang MoMo
    end

    C->>MOMO: Xác nhận thanh toán trong app MoMo

    MOMO->>PAY_API: POST /webhooks/momo/ipn\n{orderId, requestId, amount, resultCode:0,\ntransId, signature, ...}
    PAY_API->>PAY_API: Verify signature HMAC_SHA256
    PAY_API->>PAY_API: Xử lý kết quả (xem BF03 - Webhook Processing)
    PAY_API-->>MOMO: HTTP 204 No Content (hoặc 200)
```

---

## 5. Xử Lý IPN Webhook An Toàn (Idempotent Webhook Handler)

```mermaid
flowchart TD
    START["Nhận IPN Webhook\n(VNPay hoặc MoMo)"] --> IP["1. Whitelist IP Check\n(Chỉ nhận từ IP VNPay/MoMo)"]
    IP --> IP_OK{IP hợp lệ?}
    IP_OK -->|Không| BLOCK["Trả 403\nGhi Alert"]
    IP_OK -->|Có| PARSE["2. Parse Payload:\nExtract txn_ref, amount,\ngateway_txn_no, status_code, signature"]

    PARSE --> HMAC["3. Tính HMAC SHA512/256\nbằng secret key được chia sẻ"]
    HMAC --> HMAC_OK{Signature khớp?}
    HMAC_OK -->|Không| FRAUD["Ghi SECURITY_ALERT\nAlert Slack/PagerDuty\nTrả 200 OK (không báo lỗi để tránh timing attack)"]
    HMAC_OK -->|Có| IDEM["4. Kiểm tra Idempotency:\nSELECT FROM payment_transactions\nWHERE gateway_txn_id = {txn_no}"]

    IDEM --> IDEM_OK{Đã xử lý?}
    IDEM_OK -->|Rồi| SKIP["Trả 200 OK ngay lập tức\nGhi log DUPLICATE_IPN\n(Không xử lý lại)"]
    IDEM_OK -->|Chưa| STATUS["5. Kiểm tra status_code:\nVNPay: vnp_ResponseCode = '00'\nMoMo: resultCode = 0"]

    STATUS --> STATUS_OK{Thanh toán thành công?}
    STATUS_OK -->|Không| FAILED_TXN["UPDATE payment_transaction\nstatus = FAILED\norder vẫn PENDING đến khi OUT_TIME"]

    STATUS_OK -->|Có| AMOUNT["6. Khớp số tiền:\nSELECT total_amount FROM orders\nWHERE id = order_id"]
    AMOUNT --> AMT_OK{amount_paid\n= order.total_amount?}
    AMT_OK -->|Không khớp| FRAUD2["Ghi SUSPICIOUS_PAYMENT_FRAUD\nAlert tức thì\nKHÔNG cập nhật order"]

    AMT_OK -->|Khớp| ORDER_CHECK["7. Kiểm tra trạng thái đơn hàng:\nSELECT status, expires_at\nFROM orders WHERE id = order_id"]
    ORDER_CHECK --> ORDER_STATUS{order.status?}

    ORDER_STATUS -->|PENDING và chưa hết hạn| SUCCESS_FLOW["8. Atomic Transaction:\n- payment_transaction.status = SUCCESS\n- order.status = PAID\n- seats → SOLD\n- Tạo tickets (ORDERED)\n- Publish OrderPaidEvent"]
    SUCCESS_FLOW --> NOTIFY["Trả 200 OK\nNotification Service\ngửi email + push notification"]

    ORDER_STATUS -->|OUT_TIME hoặc CANCELLED| LATE["9. Late Payment:\nKiểm tra ghế còn trống không?"]
    LATE --> SEAT_OK{Ghế vẫn còn AVAILABLE?}
    SEAT_OK -->|Có| RECOVER["Phục hồi đơn hàng:\norder.status = PAID\nPhát hành vé\n(Hiếm gặp)"]
    SEAT_OK -->|Không - Đã mất| UNFULFILLED["payment_transaction.status = SUCCESS_UNFULFILLED\nKích hoạt hoàn tiền 100% ngay lập tức\nGửi email xin lỗi + mã hoàn tiền"]
```

---

## 6. Các Trường Hợp Thanh Toán Thất Bại & Xử Lý

| Mã lỗi VNPay | Mã lỗi MoMo | Nguyên nhân | Xử Lý Hệ Thống | Thông Báo Người Dùng |
|---|---|---|---|---|
| `07` | `1006` | Thẻ/Tài khoản bị khóa | Ghi `FAILED`; đơn giữ `PENDING` | "Thẻ của bạn bị khóa. Vui lòng liên hệ ngân hàng." |
| `09` | `1005` | Thẻ chưa đăng ký Internet Banking | Ghi `FAILED` | "Vui lòng đăng ký dịch vụ thanh toán online với ngân hàng." |
| `10` | `1002` | Xác thực OTP quá số lần cho phép | Ghi `FAILED`; khóa tạm thời retry | "Nhập OTP sai quá nhiều lần. Vui lòng thử lại sau 15 phút." |
| `11` | `1003` | Hết hạn chờ thanh toán | Ghi `FAILED` → Scheduler xử lý `OUT_TIME` | "Phiên thanh toán đã hết hạn. Vui lòng đặt vé lại." |
| `12` | `1001` | Thẻ/Tài khoản không đủ số dư | Ghi `FAILED` | "Tài khoản không đủ số dư. Vui lòng nạp tiền hoặc dùng phương thức khác." |
| `24` | `1007` | Khách hàng hủy giao dịch | Ghi `FAILED` | "Bạn đã hủy giao dịch. Đơn hàng vẫn còn hiệu lực trong {X} phút." |
| `99` | `9000` | Lỗi không xác định từ ngân hàng | Ghi `FAILED`; Alert team | "Đã xảy ra lỗi kỹ thuật. Vui lòng thử lại hoặc liên hệ hỗ trợ." |

---

## 7. API Endpoints Thanh Toán

| Method | Endpoint | Mô tả |
|---|---|---|
| `POST` | `/api/v1/payments/initiate` | Khởi tạo giao dịch, chọn gateway |
| `GET` | `/api/v1/payments/vnpay/return` | Nhận return URL từ VNPay (hiển thị kết quả) |
| `GET` | `/api/v1/payments/momo/return` | Nhận return URL từ MoMo |
| `POST` | `/api/v1/webhooks/vnpay/ipn` | Nhận IPN từ VNPay (server-to-server) |
| `POST` | `/api/v1/webhooks/momo/ipn` | Nhận IPN từ MoMo (server-to-server) |
| `GET` | `/api/v1/payments/{txn_id}/status` | Kiểm tra trạng thái giao dịch (polling fallback) |
