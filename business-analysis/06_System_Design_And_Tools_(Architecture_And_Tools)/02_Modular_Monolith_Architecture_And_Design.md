# 02 — Kiến Trúc Modular Monolith & Thiết Kế Module (Modular Monolith Architecture & Design)

> **Lưu ý kiến trúc:** Hệ thống được thiết kế theo mô hình **Modular Monolith** — một ứng dụng Spring Boot duy nhất bao gồm các module nghiệp vụ được phân tách rõ ràng theo Bounded Context (DDD). Cách tiếp cận này đơn giản hóa việc triển khai, giao dịch ACID và debug so với Microservices, đồng thời vẫn duy trì ranh giới module rõ ràng để có thể nâng cấp trong tương lai.

---

## 1. Triết Lý Thiết Kế (Design Philosophy)

Hệ thống áp dụng mô hình **Modular Monolith** theo phương pháp luận **Domain-Driven Design (DDD)** với các nguyên tắc cốt lõi:

1. **Ranh Giới Module Rõ Ràng (Clear Module Boundaries):** Mỗi module sở hữu trọn vẹn một domain nghiệp vụ cụ thể (Auth, Queue, Catalog, Booking, Order, Payment, Ticket, Review, Notification, Reporting). Các module KHÔNG được gọi trực tiếp vào class của nhau — mọi tương tác phải thông qua **Interface hợp đồng (Module API Interface)** được định nghĩa công khai.

2. **Giao Tiếp Nội Bộ Qua JVM (In-Process Method Calls):** Thay vì gRPC/HTTP qua mạng, các module giao tiếp trực tiếp qua lời gọi hàm trong JVM — **độ trễ 0ms**, không overhead serialization, không cần Service Discovery.

3. **Cơ Sở Dữ Liệu Chia Sẻ Có Kiểm Soát (Controlled Shared Database):** Tất cả module dùng chung một MySQL 8.0 Instance, nhưng mỗi module chỉ được phép **WRITE** vào bảng thuộc domain mình. Module khác muốn đọc dữ liệu chéo phải gọi qua Interface API — không JOIN bảng chéo module.

4. **Giao Dịch ACID Thực Sự (True ACID Transactions):** Vì cùng trong một JVM và một DB, hệ thống có thể dùng @Transactional của Spring để bao toàn bộ luồng nghiệp vụ trong một giao dịch DB duy nhất, loại bỏ hoàn toàn sự phức tạp của Saga/2PC.

5. **Bất Đồng Bộ Nội Bộ (Internal Async via Spring Events):** Các tác vụ không yêu cầu kết quả tức thì (gửi email, push notification, export báo cáo) sử dụng ApplicationEventPublisher + @Async thay vì Apache Kafka — đơn giản hơn, không cần broker riêng biệt.

---

## 2. Cấu Trúc Package Dự Án (Project Package Structure)

Toàn bộ ứng dụng được đóng gói trong một project Spring Boot 3 duy nhất:

```
com.ticketplatform/
├── TicketPlatformApplication.java        ← Main entry point
├── shared/                               ← Thành phần dùng chung
│   ├── exception/                        ← GlobalExceptionHandler, AppException
│   ├── response/                         ← ApiResponse<T> wrapper chuẩn
│   ├── security/                         ← JwtFilter, SecurityConfig
│   ├── event/                            ← ApplicationEvent interfaces
│   └── util/                             ← DateUtils, QRCodeUtils
├── auth/                                 ← MODULE 1: Auth & Account
│   ├── api/AuthModuleApi.java
│   ├── controller/AuthController.java
│   ├── service/AuthService.java
│   ├── repository/UserRepository.java
│   ├── domain/User.java
│   └── dto/
├── queue/                                ← MODULE 2: Waiting Room & Queue
├── concert/                              ← MODULE 3: Concert & Catalog
├── booking/                              ← MODULE 4: Booking & Seat Reservation
├── order/                                ← MODULE 5: Order & Checkout
├── payment/                              ← MODULE 6: Payment & Settlement
├── ticket/                               ← MODULE 7: Ticket & Check-In
├── review/                               ← MODULE 8: Review & Social
├── notification/                         ← MODULE 9: Notification Engine
└── reporting/                            ← MODULE 10: Analytics & Reporting
```

---

## 3. Danh Mục Module & Ranh Giới Nghiệp Vụ (Module Inventory & Boundaries)

| # | Module | Package | Trách nhiệm cốt lõi | Bảng DB sở hữu (WRITE) | Interface công khai |
|---|---|---|---|---|---|
| 1 | **Auth & Account** | `com.ticketplatform.auth` | Đăng ký, đăng nhập, JWT RS256, OAuth2, RBAC | `users`, `user_credentials`, `user_sessions`, `roles` | `AuthModuleApi` |
| 2 | **Waiting Room & Queue** | `com.ticketplatform.queue` | Hàng chờ ảo (Redis Sorted Sets), cấp Queue Access Token | Redis + `queue_admissions` | `QueueModuleApi` |
| 3 | **Concert & Catalog** | `com.ticketplatform.concert` | Quản lý concert, nghệ sĩ, khán đài, zones, duyệt sự kiện | `concerts`, `venues`, `artists`, `ticket_zones` | `ConcertModuleApi` |
| 4 | **Booking & Seat Reservation** | `com.ticketplatform.booking` | Giữ ghế phân tán Redis Lock (SETNX EX 600), cập nhật trạng thái ghế | `seats`, `seat_holds` | `BookingModuleApi` |
| 5 | **Order & Checkout** | `com.ticketplatform.order` | Vòng đời đơn hàng, áp voucher, tính hóa đơn | `orders`, `order_items`, `vouchers` | `OrderModuleApi` |
| 6 | **Payment & Settlement** | `com.ticketplatform.payment` | Tích hợp VNPay/MoMo, nhận IPN webhook, hoàn tiền, đối soát | `payment_transactions`, `refunds`, `settlements` | `PaymentModuleApi` |
| 7 | **Ticket & Check-In** | `com.ticketplatform.ticket` | Phát hành Dynamic QR vé, soát vé tại cổng, chống gian lận | `tickets`, `checkin_logs` | `TicketModuleApi` |
| 8 | **Review & Social** | `com.ticketplatform.review` | Đánh giá sao, bình luận (chỉ Verified Attendees) | `reviews`, `review_reports` | `ReviewModuleApi` |
| 9 | **Notification Engine** | `com.ticketplatform.notification` | Gửi Email, Push, SMS, In-App — bất đồng bộ qua @Async | `notification_logs`, `notification_templates` | *(Internal Listener)* |
| 10 | **Analytics & Reporting** | `com.ticketplatform.reporting` | Tổng hợp doanh thu, xuất Excel/PDF báo cáo (read-only) | *(READ ONLY)* | `ReportingModuleApi` |

---

## 4. Cơ Chế Giao Tiếp Giữa Các Module

### 4.1. Đồng Bộ — Gọi hàm trực tiếp qua Interface

Module A inject ModuleBApi (không inject trực tiếp Service của module khác):

```java
// BookingService gọi Concert Module qua interface
@Service
@RequiredArgsConstructor
public class BookingService implements BookingModuleApi {
    private final ConcertModuleApi concertApi; // Gọi qua Interface
    private final RedisTemplate<String, String> redisTemplate;

    @Transactional
    public SeatHoldResponse holdSeat(Long userId, Long seatId) {
        SeatInfo info = concertApi.getSeatInfo(seatId);
        boolean locked = redisTemplate.opsForValue()
            .setIfAbsent("seat:lock:" + seatId, userId.toString(), Duration.ofSeconds(600));
        if (!locked) throw new SeatAlreadyHeldException(seatId);
        return SeatHoldResponse.from(seatRepository.save(new SeatHold(userId, seatId)));
    }
}
```

### 4.2. Bất Đồng Bộ — Spring Application Events thay Kafka

```java
// PaymentService phát event sau khi IPN thành công
@Transactional
public void handleIpnCallback(VnpayIpnRequest ipn) {
    validateSignature(ipn);
    paymentRepo.updateStatus(ipn.getTxnRef(), PaymentStatus.PAID);
    orderApi.markOrderPaid(ipn.getOrderId());      // Đồng bộ, cùng transaction
    ticketApi.issueTicket(ipn.getOrderId());        // Đồng bộ, cùng transaction
    eventPublisher.publishEvent(                    // Bất đồng bộ, ngoài transaction
        new PaymentSuccessEvent(ipn.getOrderId(), ipn.getUserId()));
}

// NotificationListener xử lý bất đồng bộ
@Async("notificationExecutor")
@EventListener
public void onPaymentSuccess(PaymentSuccessEvent event) {
    emailService.sendTicketConfirmation(event.getUserId(), event.getOrderId());
    pushService.sendPush(event.getUserId(), "Vé của bạn đã sẵn sàng!");
}
```

---

## 5. Quản Lý Giao Dịch ACID (Không Cần Saga Pattern)

Toàn bộ luồng đặt vé được bao trong **một giao dịch ACID duy nhất**:

```
[Người dùng nhấn "Đặt vé & Thanh toán"]
        ↓
[@Transactional bắt đầu — một giao dịch DB duy nhất]
  1. BookingService.holdSeat()   → SETNX Redis + INSERT seat_holds
  2. OrderService.createOrder()  → INSERT orders (PENDING) + order_items
  3. PaymentService.initPayment()→ INSERT payment_transactions (PENDING)
[@Transactional COMMIT (hoặc ROLLBACK toàn bộ nếu có lỗi)]
        ↓
[Client redirect → VNPay → thanh toán]
        ↓
[IPN Webhook → @Transactional mới]
  4. UPDATE payment_transactions → PAID
  5. OrderService.markOrderPaid() → UPDATE orders → PAID
  6. TicketService.issueTicket() → INSERT tickets + QR JWT
[@Transactional COMMIT]
        ↓
[async] PaymentSuccessEvent → Email + Push Notification
```

---

## 6. Vai Trò Redis Trong Modular Monolith

| Mục đích | Cơ chế | Chi tiết |
|---|---|---|
| **Hàng chờ ảo** | Redis Sorted Sets | `ZADD queue:{concertId} {score} {userId}` |
| **Khóa giữ ghế** | `SETNX EX 600` | Phòng double-booking, TTL 10 phút |
| **Cache concert** | Redis String/Hash | Cache `concert:{id}` 5 phút — giảm load MySQL |
| **Blacklist JWT** | Redis Set | Lưu `jti` token bị thu hồi (logout) |
| **Rate Limit** | Redis INCR + EXPIRE | Đếm request theo `IP:minute` |

---

## 7. So Sánh: Modular Monolith vs Microservices

| Tiêu chí | Microservices (cũ) | Modular Monolith (hiện tại) |
|---|---|---|
| **Triển khai** | 10 Docker images + K8s + Helm | 1 image + Redis + MySQL (3 containers) |
| **Giao dịch** | Saga Pattern, eventual consistency | @Transactional ACID, nhất quán tức thì |
| **Độ trễ nội bộ** | gRPC qua mạng: 1–5ms | Gọi hàm JVM: 0ms |
| **Debug** | Distributed tracing, nhiều log stream | 1 log stream, stack trace đầy đủ |
| **RAM** | ~20–40 GB (10 services × 2–4 GB) | ~4–8 GB (1 JVM) |
| **Message Broker** | Apache Kafka (3 brokers) | Spring @Async + ApplicationEventPublisher |
| **Onboard dev mới** | Phức tạp — nhiều repo, nhiều service | Đơn giản — 1 project Maven/Gradle |

---

## 8. Cấu Hình Spring Boot (application.yml)

```yaml
spring:
  application:
    name: ticket-platform
  datasource:
    url: jdbc:mysql://mysql-host:3306/ticketdb?useSSL=true&characterEncoding=UTF-8
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 3000
  data:
    redis:
      cluster:
        nodes:
          - redis-node-1:6379
          - redis-node-2:6379
          - redis-node-3:6379
      password: ${REDIS_PASSWORD}
  task:
    execution:
      pool:
        core-size: 10
        max-size: 50
        queue-capacity: 500
server:
  port: 8080
  tomcat:
    threads:
      max: 200
      min-spare: 20
```
