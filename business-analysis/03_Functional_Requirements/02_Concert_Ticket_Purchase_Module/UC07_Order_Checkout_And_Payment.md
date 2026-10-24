# UC07 — Đặt Vé & Thanh Toán (Order Checkout & Payment)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC07 |
| **Tên Use Case** | Đặt Vé & Thanh Toán |
| **Module** | 02 — Concert & Ticket Purchase Module |
| **Actor chính** | Khách hàng (Customer) |
| **Actor phụ** | VNPay Gateway, MoMo Gateway, Notification Service |
| **Mức độ ưu tiên** | 🔴 Critical |
| **Precondition** | Customer đang giữ ≥ 1 ghế (seat.status = HELD, hold_user_id = customer); đồng hồ 10 phút vẫn còn chạy |
| **BR liên quan** | BR-ORD-001 đến BR-ORD-008; BR-PAY-001 đến BR-PAY-008; BR-TCK-001 |

---

## 2. Tóm Tắt

Use case này bao gồm toàn bộ luồng từ khi customer xác nhận chọn ghế đến khi hoàn tất thanh toán và nhận vé điện tử. Đây là luồng tạo ra doanh thu của nền tảng và yêu cầu độ tin cậy tuyệt đối: không mất tiền, không mất ghế, không mất vé.

---

## 3. Điều Kiện Tiên Quyết (Preconditions)

- Tất cả ghế trong danh sách đang ở `HELD` với đúng `hold_user_id`.
- Đơn hàng chưa được tạo trước đó cho cùng phiên này (Idempotency Key chưa dùng).
- Customer đã xác thực số điện thoại (bắt buộc để thanh toán).

---

## 4. Điều Kiện Kết Thúc (Postconditions)

**Thành công:**
- `order.status = PAID`; `seat.status = SOLD`; `ticket.status = ORDERED`.
- Vé điện tử với mã QR động được phát hành trong < 3 giây.
- Email xác nhận và Push Notification được gửi đến customer.

**Thất bại:**
- Ghế giữ nguyên `HELD` cho đến khi hết TTL 10 phút.
- Không tạo vé; không trừ tiền (hoặc kích hoạt hoàn tiền tự động nếu tiền đã bị trừ nhầm).

---

## 5. Luồng Chính (Main Flow)

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Customer | Nhấn "Tiến hành đặt vé" từ panel ghế đã chọn | Hiển thị trang Checkout: Thông tin đơn hàng, danh sách ghế, bảng giá chi tiết |
| 2 | Customer | Kiểm tra thông tin đơn hàng (tên, ghế, giá, phí dịch vụ) | Hiển thị tổng tiền phải thanh toán (đã bao gồm VAT và phí dịch vụ) |
| 3 | Customer | Chọn phương thức thanh toán: VNPay hoặc MoMo | UI hiển thị các tùy chọn phương thức cụ thể trong cổng được chọn |
| 4 | Customer | Nhấn "Thanh toán ngay" | Hệ thống tạo đơn hàng với `Idempotency-Key`; khởi tạo Payment Transaction |
| 5 | System | Tạo URL thanh toán và chuyển hướng customer | Redirect đến cổng VNPay / mở deep link MoMo App |
| 6 | Customer | Thực hiện thanh toán tại cổng (nhập OTP, xác nhận trong app MoMo) | — |
| 7 | System | Nhận IPN Webhook từ cổng thanh toán | Xác thực HMAC; khớp số tiền; kiểm tra idempotency |
| 8 | System | Cập nhật hệ thống trong 1 transaction atomic | `order → PAID`; `seats → SOLD`; `payment_transaction → SUCCESS`; `tickets → ORDERED (với QR)` |
| 9 | System | Gửi thông báo | Email xác nhận đơn + vé điện tử; Push Notification "Vé của bạn đã sẵn sàng!" |
| 10 | Customer | Xem kết quả | Trang xác nhận với thông tin vé, hướng dẫn check-in |

---

## 6. Luồng Thay Thế (Alternative Flows)

### AF-01: Thanh Toán Thất Bại Tại Cổng

| Bước | Mô tả |
|---|---|
| 1 | IPN Webhook trả về mã lỗi (OTP sai, không đủ tiền, hết phiên...) |
| 2 | `payment_transaction.status = FAILED` |
| 3 | `order.status` giữ nguyên `PENDING`; ghế vẫn `HELD` cho đến khi hết TTL |
| 4 | Customer được redirect về trang checkout với thông báo lỗi và lý do cụ thể |
| 5 | Customer có thể thử lại với phương thức khác hoặc chờ đến khi hết TTL |

### AF-02: Thanh Toán Trễ — IPN Đến Sau Khi Ghế Hết Hạn (BR-PAY-003)

| Bước | Mô tả |
|---|---|
| 1 | IPN thành công đến sau khi `order.status = OUT_TIME` và ghế đã bị người khác mua |
| 2 | Hệ thống KHÔNG ghi đè ghế đã SOLD của người khác |
| 3 | `payment_transaction.status = SUCCESS_UNFULFILLED` |
| 4 | Tự động kích hoạt hoàn tiền 100% ngay lập tức |
| 5 | Gửi email + SMS xin lỗi với mã giao dịch hoàn tiền |

---

## 7. Luồng Ngoại Lệ (Exception Flows)

| ID | Điều kiện | Xử lý |
|---|---|---|
| EX-01 | Đồng hồ 10 phút hết hạn trong khi customer đang ở trang checkout (chưa bấm thanh toán) | Hiện cảnh báo ở giây 60 cuối; khi hết → Hiện "Phiên đã hết hạn"; redirect về sơ đồ ghế |
| EX-02 | Cổng thanh toán không phản hồi (timeout 30 giây) | Ghi `payment_transaction = FAILED`; thông báo "Cổng thanh toán tạm thời không khả dụng. Vui lòng thử lại hoặc chọn phương thức khác." |
| EX-03 | Số tiền IPN Webhook không khớp với `order.total_amount` | Ghi cờ `SUSPICIOUS_PAYMENT_FRAUD`; alert team an ninh; không cập nhật đơn hàng |
| EX-04 | Chữ ký HMAC không hợp lệ | Từ chối xử lý; ghi Security Alert; không trả lỗi ra ngoài (trả 200 OK) |
| EX-05 | Click "Thanh toán" nhiều lần (double submit) | `Idempotency-Key` đảm bảo chỉ tạo 1 đơn; các lần click sau trả về kết quả đơn đã tạo |
| EX-06 | Customer đóng tab ngay sau khi thanh toán tại cổng nhưng trước khi redirect về | IPN vẫn đến và xử lý đúng; lần sau customer truy cập vào "Đơn hàng của tôi" sẽ thấy đơn `PAID` |

---

## 8. Cấu Trúc Hiển Thị Trang Checkout

| Phần | Nội dung hiển thị |
|---|---|
| **Thông tin sự kiện** | Tên concert, nghệ sĩ, ngày giờ, địa điểm |
| **Danh sách ghế** | Từng ghế: Mã ghế, Zone, Giá ghế |
| **Bảng tổng tiền** | Giá vé x số lượng, Phí dịch vụ (%), VAT (nếu có), **Tổng cộng** |
| **Đồng hồ đếm ngược** | Còn X:XX phút để hoàn tất thanh toán (màu đỏ khi < 2 phút) |
| **Chọn phương thức** | VNPay (ATM nội địa, QR, Visa), MoMo (ví, thẻ liên kết) |
| **Thông tin người mua** | Họ tên, SĐT (từ hồ sơ tài khoản, có thể chỉnh sửa) |

---

## 9. User Stories & Acceptance Criteria

### US-07-01: Tạo Đơn Hàng Và Thanh Toán

> **As a** khách hàng đã giữ ghế,  
> **I want to** thanh toán đơn vé qua VNPay hoặc MoMo,  
> **So that** nhận được vé điện tử hợp lệ để vào sự kiện.

| AC | Mô tả | Loại |
|---|---|---|
| AC-07-01-01 | Trang checkout hiển thị bảng giá đầy đủ: giá gốc, phí dịch vụ, VAT, tổng cộng; không có phí ẩn | Functional |
| AC-07-01-02 | Đơn hàng được tạo với `Idempotency-Key`; click nhiều lần không tạo nhiều đơn | Functional |
| AC-07-01-03 | Từ khi IPN thành công đến khi vé và QR sẵn sàng: < 3 giây | Performance |
| AC-07-01-04 | Customer nhận email xác nhận có vé đính kèm trong < 30 giây sau thanh toán thành công | Performance |
| AC-07-01-05 | Thông tin ghế, giá vé trong đơn hàng khớp 100% với những gì hiển thị trên trang checkout | Functional (Critical) |

---

### US-07-02: Hiển Thị Đồng Hồ Cảnh Báo Khi Sắp Hết Hạn

> **As a** khách hàng,  
> **I want to** thấy cảnh báo rõ ràng khi thời gian giữ ghế sắp hết,  
> **So that** tôi biết cần hoàn tất thanh toán gấp hoặc chấp nhận mất ghế.

| AC | Mô tả | Loại |
|---|---|---|
| AC-07-02-01 | Đồng hồ đếm ngược hiển thị liên tục trên trang checkout; chuyển màu đỏ khi còn < 2 phút | Functional |
| AC-07-02-02 | Khi còn 60 giây: popup/toast cảnh báo "Còn 1 phút — Thanh toán ngay!" | UX |
| AC-07-02-03 | Khi hết 10 phút: thông báo rõ ràng "Đã hết thời gian giữ ghế. Ghế của bạn đã được trả về cho người khác." | Functional |

---

### US-07-03: Xem Vé Điện Tử Sau Thanh Toán

> **As a** khách hàng vừa thanh toán thành công,  
> **I want to** xem và lưu vé điện tử kèm mã QR,  
> **So that** chuẩn bị check-in tại cổng sự kiện.

| AC | Mô tả | Loại |
|---|---|---|
| AC-07-03-01 | Trang kết quả thanh toán hiển thị ngay lập tức mã QR của từng vé | Functional |
| AC-07-03-02 | Customer có thể tải vé dạng PDF hoặc lưu vào bộ sưu tập vé trong app | Functional |
| AC-07-03-03 | Vé hiển thị đầy đủ: Tên concert, Ngày giờ, Địa điểm, Tên chủ vé, Khu vực, Mã ghế, Mã QR | Functional |

---

## 10. Danh Sách Use Case Con & Quan Hệ Actor

| Use Case Con | Actor Chính | Actor Phụ | Ghi Chú |
|---|---|---|---|
| UC07.1 — Xem trang checkout | Customer | — | Hiển thị tổng tiền, chọn gateway |
| UC07.2 — Tạo đơn hàng (PENDING) | Customer | Order Service | Idempotency-Key bắt buộc |
| UC07.3 — Thanh toán qua VNPay | Customer | VNPay Gateway | Redirect URL + IPN Webhook |
| UC07.4 — Thanh toán qua MoMo | Customer | MoMo Gateway | Deep link App + IPN Webhook |
| UC07.5 — Xử lý IPN Webhook | System | VNPay / MoMo | Validate HMAC + khớp tiền + idempotency |
| UC07.6 — Phát hành vé điện tử | System | Ticket Service, QR Service | Sau khi IPN SUCCESS |
| UC07.7 — Gửi thông báo xác nhận | System | Email Service, Push Service | Email + Push trong < 30s |
