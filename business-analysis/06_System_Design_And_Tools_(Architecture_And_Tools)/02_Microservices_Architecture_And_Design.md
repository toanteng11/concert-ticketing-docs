# 02 — Kiến Trúc & Thiết Kế Microservices (Microservices Architecture & Design)

## 1. Giới Thiệu & Triết Lý Thiết Kế (Design Philosophy)

Để đảm bảo hệ thống có thể mở rộng độc lập từng chức năng, cô lập lỗi và phục vụ hàng trăm nghìn người dùng đồng thời, kiến trúc hệ thống được thiết kế theo phương pháp luận **Phát triển Hướng Tên Miền (Domain-Driven Design - DDD)**:
1. **Nguyên Tắc Ranh Giới Ngữ Cảnh (Bounded Contexts):** Mỗi dịch vụ sở hữu trọn vẹn một miền logic nghiệp vụ cụ thể.
2. **Cơ Sở Dữ Liệu Riêng Biệt (Database-per-Service / Schema Isolation):** Nghiêm cấm việc một Microservice truy vấn trực tiếp vào bảng dữ liệu của Microservice khác; mọi tương tác bắt buộc phải thông qua API hợp đồng hoặc Sự kiện bất đồng bộ.
3. **Kiến Trúc Hướng Sự Kiện (Event-Driven Architecture):** Sử dụng Apache Kafka làm xương sống truyền thông tin bất đồng bộ, giảm tối đa sự phụ thuộc chặt chẽ (Loose Coupling) giữa các dịch vụ.

Tài liệu này đặc tả danh mục toàn bộ Microservices, ranh giới dữ liệu và các giao thức giao tiếp liên dịch vụ (Inter-Service Communication).

---

## 2. Bảng Danh Mục Các Microservices (Service Inventory & Boundaries)

| Tên Microservice | Ngôn ngữ phát triển | Trách nhiệm cốt lõi | Cơ sở dữ liệu sở hữu | Giao diện đồng bộ (Sync API) | Sự kiện phát ra (Events Published) | Sự kiện lắng nghe (Events Consumed) |
|---|---|---|---|---|---|---|
| **1. Auth & Account Service** | Java (Spring Boot 3) | Đăng ký, đăng nhập, JWT RS256, OAuth2, hồ sơ cá nhân, phân quyền RBAC | MySQL: `users`, `user_credentials`, `user_sessions` | REST: `/api/v1/auth/*`, `/api/v1/users/*` | `user.registered`, `user.password_changed`, `user.locked` | — |
| **2. Waiting Room & Queue Service** | Java (Spring Boot / Netty) | Quản lý hàng chờ ảo, xếp hàng ngẫu nhiên, cấp mã Queue Access Token | Redis Cluster (Sorted Sets) | REST: `/api/v1/queue/status`, `/api/v1/queue/enter` | `queue.admitted`, `queue.expired` | `concert.on_sale_started` |
| **3. Concert & Catalog Service** | Java (Spring Boot 3) | Quản lý danh mục concert, nghệ sĩ, sơ đồ khán đài, duyệt sự kiện | MySQL: `concerts`, `ticket_zones`, `venues`, `artists` | REST: `/api/v1/concerts/*` + gRPC nội bộ | `concert.published`, `concert.cancelled`, `concert.updated` | — |
| **4. Booking & Reservation Service** | Java (Spring Boot 3) | Giữ ghế phân tán (Redis Lock), timeout 10 phút, cập nhật sơ đồ ghế real-time | Redis (Seat Locks) + MySQL: `seat_holds`, `seats` | REST: `/api/v1/seats/hold`, `/api/v1/seats/release` + WebSocket | `seat.held`, `seat.released`, `seat.sold` | `order.created`, `order.cancelled`, `payment.failed` |
| **5. Order & Checkout Service** | Java (Spring Boot 3) | Quản lý vòng đời đơn hàng, áp dụng voucher, tính toán hóa đơn | MySQL: `orders`, `order_items`, `vouchers` | REST: `/api/v1/orders/*` | `order.created`, `order.paid`, `order.cancelled` | `payment.success`, `payment.failed`, `seat.held` |
| **6. Payment & Settlement Service** | Java (Spring Boot 3) | Tích hợp cổng VNPay/MoMo, tiếp nhận webhook IPN, hoàn tiền, đối soát | MySQL: `payment_transactions`, `refunds`, `settlements` | REST: `/api/v1/payments/*` (Webhook callback) | `payment.success`, `payment.failed`, `refund.completed` | `order.created`, `order.cancelled` |
| **7. Ticket & Check-In Service** | Java (Spring Boot 3) | Phát hành Dynamic QR vé điện tử, giải mã JWT soát vé tại cổng, hỗ trợ offline | Redis + MySQL: `tickets`, `checkin_logs` + SQLite Offline | REST: `/api/v1/tickets/*`, `/api/v1/check-in` | `ticket.issued`, `ticket.checked_in`, `ticket.fraud_detected` | `order.paid`, `concert.cancelled` |
| **8. Review & Social Service** | Java (Spring Boot 3) | Đánh giá sao, bình luận của khán giả thực tế (Verified Attendees) | MySQL: `reviews`, `review_reports` | REST: `/api/v1/reviews/*` | `review.created`, `review.flagged` | `ticket.checked_in` |
| **9. Notification Engine** | Java (Spring Boot Workers) | Định tuyến thông báo đa kênh (Email, Push, SMS, In-App), khử trùng lặp | MySQL: `notification_logs`, `templates` + Redis Dedup | REST nội bộ | `notification.sent`, `notification.bounced` | Lắng nghe toàn bộ sự kiện nghiệp vụ của hệ thống |
| **10. Analytics & Reporting Service** | Java (Spring Boot / POI) | Tổng hợp số liệu doanh thu, tỷ lệ lấp đầy ghế, xuất báo cáo Excel | MySQL Read Replica / ClickHouse | REST: `/api/v1/reports/*` | `report.generated` | `order.paid`, `refund.completed`, `ticket.checked_in` |

---

## 3. Giao Thức Giao Tiếp Liên Dịch Vụ (Inter-Service Communication)

Hệ thống kết hợp hài hòa cả 2 phương thức giao tiếp để tối ưu hóa hiệu năng:

### 1. Giao Tiếp Đồng Bộ (Synchronous Communication - gRPC & REST)
- **Áp dụng cho:** Các truy vấn đọc nội bộ đòi hỏi kết quả tức thì (ví dụ: `Order Service` gọi `Auth Service` để kiểm tra thông tin người dùng; hoặc `Booking Service` gọi `Concert Service` để lấy cấu hình Zone).
- **Giao thức khuyến nghị:** **gRPC qua HTTP/2 với Protocol Buffers (Protobuf)**:
  - Tải trọng nhị phân nhỏ hơn $60\%$ so với JSON.
  - Tốc độ đóng gói/giải mã (Serialization) nhanh gấp 5–10 lần JSON.
  - Hợp đồng giao tiếp (Contract) được định nghĩa chặt chẽ bằng tệp `.proto`.

### 2. Giao Tiếp Bất Đồng Bộ Hướng Sự Kiện (Asynchronous Event-Driven - Kafka)
- **Áp dụng cho:** Mọi thao tác ghi giao dịch làm thay đổi trạng thái nghiệp vụ.
- **Mô hình Xuất bản / Đăng ký (Publish / Subscribe):**
  - Khi một giao dịch thanh toán thành công, `Payment Service` chỉ cần phát duy nhất sự kiện `payment.success` lên Kafka topic.
  - `Order Service` nghe sự kiện $\rightarrow$ Chuyển trạng thái đơn sang `PAID`.
  - `Booking Service` nghe sự kiện $\rightarrow$ Chuyển ghế sang `SOLD` vĩnh viễn.
  - `Ticket Service` nghe sự kiện $\rightarrow$ Sinh mã vé và QR Code.
  - `Notification Service` nghe sự kiện $\rightarrow$ Gửi email hóa đơn và vé cho khách.
  - **Lợi ích:** `Payment Service` hoàn toàn không cần biết có bao nhiêu service phía sau, phản hồi cho cổng thanh toán chỉ mất $< 50\text{ ms}$.

---

## 4. Quản Lý Giao Dịch Phân Tán Bằng Mẫu Thiết Kế SAGA (Distributed Saga Pattern)

Vì mỗi Microservice sở hữu cơ sở dữ liệu riêng, hệ thống không thể sử dụng giao dịch ACID truyền thống giữa các service. Thay vào đó, áp dụng mô hình **Saga Biên Đạo (Choreography-Based Saga)** kết hợp Giao dịch Bù trừ (Compensating Transactions):

### Kịch Bản: Đặt Vé Thành Công (Happy Path Saga)
1. **Bước 1 (Booking Service):** Khóa giữ ghế thành công $\rightarrow$ Phát sự kiện `seat.held`.
2. **Bước 2 (Order Service):** Nhận `seat.held` $\rightarrow$ Tạo đơn hàng trạng thái `PENDING` $\rightarrow$ Phát sự kiện `order.created`.
3. **Bước 3 (Payment Service):** Nhận `order.created` $\rightarrow$ Tiếp nhận thanh toán của khách $\rightarrow$ Thanh toán thành công $\rightarrow$ Phát sự kiện `payment.success`.
4. **Bước 4 (Ticket Service):** Nhận `payment.success` $\rightarrow$ Phát hành vé điện tử `ORDERED` $\rightarrow$ Phát sự kiện `ticket.issued`.
5. **Bước 5 (Saga Hoàn Tất):** Toàn bộ trạng thái đạt điểm nhất quán cuối cùng.

### Kịch Bản Bù Trừ: Thanh Toán Thất Bại / Hết Hạn (Compensating Saga)
1. Nếu tại Bước 3 thanh toán thất bại hoặc quá 10 phút chưa chuyển tiền:
2. **Sự kiện kích hoạt:** `Payment Service` phát sự kiện `payment.failed`.
3. **Hành động bù trừ 1 (Order Service):** Chuyển trạng thái đơn hàng sang `CANCELLED`.
4. **Hành động bù trừ 2 (Booking Service):** Thu hồi khóa giữ ghế, chuyển trạng thái ghế trong Redis và DB trở lại `AVAILABLE`.
5. **Hành động bù trừ 3 (Notification Service):** Gửi thông báo cho khách hàng: *"Đơn hàng bị hủy do chưa hoàn tất thanh toán"*.

---

## 5. Phát Hiện Dịch Vụ & Lưới Dịch Vụ (Service Discovery & Service Mesh)

- **Service Discovery:** Sử dụng cơ chế phân giải tên miền nội bộ của Kubernetes (**CoreDNS**): Các service gọi nhau qua địa chỉ DNS cục bộ dạng `http://booking-service.ticketing.svc.cluster.local:8080`.
- **Khả Năng Chịu Lỗi Tại Client (Client-Side Resilience):** Mọi lệnh gọi HTTP/gRPC nội bộ đều được bọc bởi thư viện Circuit Breaker (như Resilience4j hoặc Go-hystrix) với cấu hình:
  - Connection Timeout: $1.000\text{ ms}$.
  - Read Timeout: $2.000\text{ ms}$.
  - Tối đa 2 lần Retry tự động kèm cơ chế lùi theo hàm mũ (Exponential Backoff with Jitter).
