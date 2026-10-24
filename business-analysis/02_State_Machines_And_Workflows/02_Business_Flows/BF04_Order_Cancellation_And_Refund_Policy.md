# BF04 — Business Flow: Hủy Đơn & Chính Sách Hoàn Tiền (Order Cancellation & Refund Policy)

## 1. Mục Đích

Tài liệu mô tả chi tiết luồng nghiệp vụ **Hủy Đơn Hàng** và **Hoàn Tiền** — bao gồm hoàn tiền tự nguyện của khách hàng và hoàn tiền hàng loạt khi concert bị hủy.

---

## 2. Cây Quyết Định Hoàn Tiền (Refund Decision Tree)

```mermaid
flowchart TD
    A["Khách hàng muốn hủy vé\norder.status = PAID"] --> B{"Có vé nào trong đơn\nđã được CHECK-IN không?"}

    B -->|Có — Đã vào cửa| BLOCKED["❌ KHÔNG THỂ HOÀN TIỀN\n(BR-RFD-006)\nThông báo: Vé đã được sử dụng để vào sự kiện"]

    B -->|Không — Chưa ai check-in| C{"Số ngày còn lại đến\nngày biểu diễn?"}

    C -->|"> 7 ngày (> 168h)"| FULL["✅ HOÀN TIỀN 100%\ntrên giá vé\n(BR-RFD-001 — Tier 1)"]
    C -->|"3–7 ngày (72h–168h)"| HALF["✅ HOÀN TIỀN 50%\ntrên giá vé\n(BR-RFD-001 — Tier 2)"]
    C -->|"< 3 ngày (< 72h)"| NONE["❌ KHÔNG HOÀN TIỀN\n(BR-RFD-001 — Tier 3)\nHệ thống không cho phép gửi yêu cầu"]

    FULL --> PROCESS["Tạo Refund Request\nGửi lệnh hoàn tiền\ntới VNPay / MoMo"]
    HALF --> PROCESS
    PROCESS --> RELEASE["Vé → CANCELLED\nGhế → AVAILABLE (nếu concert còn ON_SALE)\norder.status = REFUND_PENDING"]
```

---

## 3. Luồng Hủy Đơn Tự Nguyện Của Khách Hàng

```mermaid
sequenceDiagram
    actor C as Khách hàng
    participant APP as Mobile/Web App
    participant API as Order API
    participant REFUND as Refund Service
    participant PAY_GW as Payment Gateway (VNPay/MoMo)
    participant NT as Notification Service

    C->>APP: Vào mục "Vé của tôi"\nChọn đơn hàng → Click "Yêu cầu hủy vé"

    APP->>API: POST /orders/{order_id}/cancel-request\nHeader: Authorization: Bearer {jwt}

    API->>API: Kiểm tra điều kiện:\n1. order.customer_id = current_user_id\n2. order.status = PAID\n3. Không có ticket nào đã CHECKED\n4. Tính (concert.event_date - now()) → Xác định tier hoàn tiền

    alt Điều kiện hợp lệ

        API->>API: Tính refund_amount:\n- > 7 ngày: 100% price\n- 3-7 ngày: 50% price\n- < 3 ngày: 0% → Từ chối

        API-->>APP: Hiển thị màn xác nhận:\n"Bạn sẽ được hoàn {refund_amount} VNĐ\n(về {payment_method} trong {SLA_time}).\nXác nhận hủy vé?"

        C->>APP: Click "Xác nhận hủy"
        APP->>API: POST /orders/{order_id}/cancel-confirm

        API->>API: Atomic Transaction:\n- order.status = REFUND_PENDING\n- tickets → CANCELLED (QR vô hiệu hóa)\n- seats → AVAILABLE

        API->>REFUND: CreateRefundRequest\n{order_id, amount: refund_amount, gateway}

        REFUND->>PAY_GW: Gọi API hoàn tiền\n(VNPay Refund API / MoMo Refund API)
        PAY_GW-->>REFUND: Xác nhận nhận lệnh hoàn tiền

        REFUND-->>API: payment_transaction.status = REFUND_PROCESSING

        API->>NT: Trigger thông báo
        NT-->>C: Email: "Yêu cầu hủy vé thành công.\nTiền hoàn {amount} VNĐ sẽ về trong {SLA_days} ngày."
        NT-->>C: Push Notification: "Vé đã hủy. Đang xử lý hoàn tiền..."

        API-->>APP: 200 OK — Hủy vé thành công
        APP-->>C: Màn hình xác nhận + Trạng thái đơn: "Đang hoàn tiền"

    else Vé đã check-in
        API-->>APP: 422 Unprocessable Entity\n{error:"TICKET_ALREADY_CHECKED", code:"BR-RFD-006"}
        APP-->>C: "Không thể hủy vé đã được sử dụng để vào sự kiện."

    else Quá gần ngày diễn (< 3 ngày)
        API-->>APP: 422 Unprocessable Entity\n{error:"REFUND_WINDOW_CLOSED", code:"BR-RFD-001"}
        APP-->>C: "Đã quá thời hạn được phép hủy vé\n(dưới 3 ngày trước ngày diễn)."
    end
```

---

## 4. Luồng Hoàn Tiền Hàng Loạt Khi Concert Bị Hủy

```mermaid
sequenceDiagram
    participant ADMIN as Admin
    participant CON_API as Concert API
    participant EVENT_BUS as Event Bus (Message Queue)
    participant ORDER_SVC as Order Service
    participant REFUND as Refund Service
    participant PAY_GW as Payment Gateway
    participant NT as Notification Service

    ADMIN->>CON_API: PUT /admin/concerts/{id}/cancel\nBody: {reason: "Nghệ sĩ hủy show do sức khỏe"}
    CON_API->>CON_API: UPDATE concerts SET status='CANCELLED',\ncancelled_at=NOW(), cancel_reason=?

    CON_API->>EVENT_BUS: Publish ConcertCancelledEvent\n{concert_id, reason, cancelled_at}

    EVENT_BUS->>ORDER_SVC: ConcertCancelledEvent received

    ORDER_SVC->>ORDER_SVC: SELECT * FROM orders\nWHERE concert_id = ? AND status = 'PAID'
    Note over ORDER_SVC: Có thể hàng nghìn đơn hàng

    loop Với mỗi PAID Order (batch processing)
        ORDER_SVC->>ORDER_SVC: order.status = CANCELLED
        ORDER_SVC->>REFUND: CreateRefundRequest\n{order_id, amount: 100% of order.total_amount, reason: "CONCERT_CANCELLED"}
        REFUND->>PAY_GW: Gọi API hoàn tiền (batch)
        PAY_GW-->>REFUND: Nhận lệnh xử lý
        ORDER_SVC->>ORDER_SVC: tickets → CANCELLED (batch)\nseats → AVAILABLE (batch)
    end

    REFUND->>EVENT_BUS: Publish RefundBatchInitiatedEvent

    EVENT_BUS->>NT: RefundBatchInitiatedEvent
    NT->>NT: Lấy danh sách email/SĐT của tất cả khách hàng bị ảnh hưởng

    par Gửi thông báo hàng loạt (async)
        NT-->>Customer_1: Email + Push Notification:\n"Concert [Tên] đã bị hủy. Chúng tôi xin lỗi.\nBạn sẽ được hoàn 100% tiền vé\n({amount} VNĐ) trong vòng {SLA_days} ngày làm việc."
        NT-->>Customer_2: (Tương tự)
        NT-->>Customer_N: (Tương tự)
    end

    CON_API-->>ADMIN: 200 OK\n{affected_orders: N, total_refund_amount: X_VND}
```

---

## 5. Luồng Cổng Thanh Toán Xác Nhận Hoàn Tiền (Refund IPN Processing)

```mermaid
sequenceDiagram
    participant PAY_GW as Payment Gateway
    participant REFUND_WEBHOOK as Refund Webhook Handler
    participant DB as Database
    participant NT as Notification Service

    PAY_GW->>REFUND_WEBHOOK: POST /webhooks/{gateway}/refund-ipn\n{refund_id, original_txn_id, amount, result_code, signature}

    REFUND_WEBHOOK->>REFUND_WEBHOOK: 1. Xác thực HMAC signature
    REFUND_WEBHOOK->>REFUND_WEBHOOK: 2. Kiểm tra Idempotency (refund_id chưa xử lý)
    REFUND_WEBHOOK->>REFUND_WEBHOOK: 3. Khớp refund_amount với lệnh hoàn tiền gốc

    alt Hoàn tiền thành công (result_code = 00 / 0)
        REFUND_WEBHOOK->>DB: UPDATE payment_transactions\nSET status = 'REFUNDED',\n    refunded_at = NOW()\nWHERE refund_id = ?

        REFUND_WEBHOOK->>DB: UPDATE orders\nSET status = 'REFUNDED'\nWHERE id = order_id

        REFUND_WEBHOOK->>NT: Trigger thông báo hoàn tiền thành công
        NT-->>Customer: Email: "Tiền đã hoàn về tài khoản của bạn.\nMã giao dịch hoàn: {refund_txn_id}"
        NT-->>Customer: Push Notification: "✅ Hoàn tiền thành công!"

        REFUND_WEBHOOK-->>PAY_GW: 200 OK

    else Hoàn tiền thất bại (result_code != 00)
        REFUND_WEBHOOK->>DB: UPDATE payment_transactions\nSET status = 'REFUND_FAILED',\n    retry_count = retry_count + 1

        alt retry_count < 3
            REFUND_WEBHOOK->>DB: Schedule retry trong 24h\n(Delayed Job Queue)
        else retry_count >= 3
            REFUND_WEBHOOK->>DB: status = 'REFUND_FAILED' (final)
            REFUND_WEBHOOK->>NT: Alert Admin + Finance team:\n"Hoàn tiền thất bại sau 3 lần retry\nOrder: {order_id} — Amount: {amount}\nCần xử lý thủ công"
        end

        REFUND_WEBHOOK-->>PAY_GW: 200 OK
    end
```

---

## 6. Thời Gian Hoàn Tiền Theo Phương Thức Thanh Toán (Refund SLA)

| Phương thức thanh toán gốc | Thời gian hoàn tiền | Ghi chú |
|---|---|---|
| Ví MoMo | 1 – 2 ngày làm việc | Hoàn về ví điện tử MoMo |
| Thẻ ATM nội địa (VNPay) | 3 – 5 ngày làm việc | Hoàn về tài khoản ngân hàng |
| Thẻ Visa / Mastercard (VNPay) | 7 – 15 ngày làm việc | Theo kỳ sao kê của ngân hàng phát hành |
| Thẻ tín dụng nội địa | 5 – 7 ngày làm việc | Tùy ngân hàng phát hành |

> **Lưu ý:** Thời gian tính từ khi hệ thống gửi lệnh hoàn tiền đến cổng thanh toán, không tính thời gian chờ duyệt yêu cầu.

---

## 7. Bảng Tóm Tắt Chính Sách Hoàn Tiền (Refund Policy Summary)

| Trường hợp | Điều kiện | Tỷ lệ hoàn | Ai kích hoạt | Thời hạn xử lý |
|---|---|---|---|---|
| **Hủy tự nguyện — Sớm** | Trước concert > 7 ngày | 100% giá vé | Khách hàng | 1–15 ngày (tùy phương thức) |
| **Hủy tự nguyện — Trung hạn** | Trước concert 3–7 ngày | 50% giá vé | Khách hàng | 1–15 ngày |
| **Hủy tự nguyện — Muộn** | Trước concert < 3 ngày | 0% — Không hoàn | Không cho phép | — |
| **Concert bị hủy bởi Admin** | Bất kỳ thời điểm nào | 100% giá vé | Hệ thống (Auto) | 1–15 ngày |
| **Late Payment Callback** | Tiền trừ nhưng ghế đã mất | 100% số tiền đã trừ | Hệ thống (Auto) | 1–15 ngày |
| **Vé đã check-in** | Bất kỳ lý do nào | 0% — Không hoàn | Không cho phép | — |

