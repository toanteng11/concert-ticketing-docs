# 01 — Kiến Trúc Tổng Thể Client-Server & API Gateway (High-Level Architecture)

## 1. Giới Thiệu & Mục Đích Tài Liệu

Tài liệu này đặc tả kiến trúc tổng thể cấp cao (**High-Level System Architecture**) của nền tảng bán vé sự kiện âm nhạc quy mô lớn theo mô hình **Modular Monolith**. Tài liệu cung cấp toàn bộ các thành phần, các lớp kiến trúc (Architectural Layers), biên giới mạng, cơ chế tiếp nhận lưu lượng và đường đi của dữ liệu qua từng tầng để làm cơ sở cho kỹ sư hệ thống vẽ sơ đồ kiến trúc tổng thể (**High-Level Architecture Diagram**) và sơ đồ luồng dữ liệu (**Data Flow Diagram**).

> **Ghi chú kiến trúc:** Hệ thống backend là một ứng dụng **Spring Boot Modular Monolith** duy nhất — không phải tập hợp các Microservices riêng lẻ. Toàn bộ 10 module nghiệp vụ (Auth, Queue, Concert, Booking, Order, Payment, Ticket, Review, Notification, Reporting) được đóng gói trong một Docker container duy nhất và giao tiếp nội bộ qua lời gọi hàm trong JVM.

---

## 2. Phân Tầng Kiến Trúc Hệ Thống (Layered Architecture Breakdown)

Hệ thống được tổ chức thành 6 tầng kiến trúc vật lý và logic phân tách rõ ràng:

### Tầng 1: Tầng Ứng Dụng Khách (Client & Presentation Layer)
- **Web Portal (Khách hàng & Ban tổ chức):** Ứng dụng Single Page Application (SPA) phát triển bằng React / Next.js hoặc Vue.js, tối ưu hóa SEO và tải trang tức thì.
- **Mobile Native Application (iOS & Android):** Ứng dụng di động phát triển bằng Flutter hoặc React Native, tích hợp Dynamic QR Token Generator và Push Notification Native (APNs / FCM).
- **Staff Check-in Scanner App:** Ứng dụng chuyên dụng cho nhân viên soát vé tại cổng với camera quét tốc độ cao và cơ chế lưu trữ đệm SQLite cục bộ khi mất mạng.
- **Admin Management Portal:** Giao diện quản trị hệ thống tập trung dành cho Quản trị viên sàn.

### Tầng 2: Tầng Mạng Biên & Phòng Thủ Tuyến Đầu (Edge CDN & WAF Layer)
- **DNS Anycast Toàn cầu:** Phân giải tên miền độ trễ thấp và chuyển hướng lưu lượng thông minh.
- **Cloudflare Enterprise WAF & DDoS Shield:** Hấp thụ các cuộc tấn công từ chối dịch vụ (DDoS Lớp 3/4/7) lên tới hàng Terabit/giây; kiểm tra chứng chỉ SSL/TLS 1.3 tại biên mạng.
- **Edge CDN Caching:** Lưu trữ và phân phối tài nguyên tĩnh (JS, CSS, hình ảnh, sơ đồ ghế SVG) gần người dùng nhất tại các PoP (Hà Nội, TP.HCM, Đà Nẵng).

### Tầng 3: Tầng Cổng Giao Tiếp Ứng Dụng (API Gateway & Ingress Layer)
- **Công nghệ cốt lõi:** Kong Gateway hoặc Nginx Reverse Proxy (trên Docker Compose / Kubernetes).
- **Chức năng tập trung:**
  - Giải mã SSL (TLS Termination) tập trung.
  - Xác thực chữ ký số JWT và kiểm tra cờ thu hồi phiên (Token Revocation Check via Redis Blacklist).
  - Kiểm tra thẻ thông hành phòng chờ ảo (Virtual Queue Access Token).
  - Giới hạn tần suất gọi API (Rate Limiting & IP Throttling — dựa trên Redis Counter).
  - Chèn mã theo dõi (`X-Correlation-ID`) vào mọi request.
  - Định tuyến toàn bộ traffic tới **một endpoint duy nhất** của ứng dụng Modular Monolith.

### Tầng 4: Tầng Ứng Dụng Monolith (Modular Monolith Application Layer)
- **Một ứng dụng Spring Boot 3 duy nhất** chứa 10 module nghiệp vụ:
  - `auth` — Xác thực & Tài khoản
  - `queue` — Phòng Chờ Ảo
  - `concert` — Danh Mục Sự Kiện
  - `booking` — Đặt Ghế & Giữ Ghế
  - `order` — Quản Lý Đơn Hàng
  - `payment` — Thanh Toán & Đối Soát
  - `ticket` — Phát Hành Vé & Soát Vé
  - `review` — Đánh Giá & Bình Luận
  - `notification` — Gửi Thông Báo Đa Kênh (Async)
  - `reporting` — Báo Cáo & Phân Tích
- Tất cả module chạy trong **một JVM duy nhất**, giao tiếp qua lời gọi hàm trực tiếp.
- Triển khai dưới dạng **1 Docker container** (port 8080).

### Tầng 5: Tầng Bộ Đệm Nóng & Khóa Phân Tán (In-Memory Caching & Locking Layer)
- **Redis Cluster (3 nodes):** Chịu trách nhiệm cho các tác vụ đòi hỏi tốc độ cực cao:
  - Hàng chờ ảo (Sorted Sets).
  - Khóa giữ ghế phân tán (`SETNX EX 600`).
  - Bộ đếm Rate Limit (INCR + EXPIRE).
  - Cache danh mục concert và Blacklist JWT Token.

### Tầng 6: Tầng Lưu Trữ Bền Vững (Persistence & Storage Layer)
- **Cơ sở dữ liệu Giao dịch (MySQL 8.0 HA):** 1 Primary + 2 Read Replicas (InnoDB Engine) kết hợp HikariCP Connection Pooling (pool-size: 50). Toàn bộ 10 module dùng chung một database instance — mỗi module sở hữu bộ bảng riêng.
- **Kho Lưu Trữ Đối Tượng (AWS S3 / MinIO):** Lưu ảnh poster, sơ đồ ghế gốc, hóa đơn PDF, tệp xuất báo cáo lớn.
- **Công Cụ Tìm Kiếm (OpenSearch):** Phục vụ tìm kiếm concert toàn văn (Full-Text Search) và lưu trữ nhật ký kiểm toán (Audit Logs).

---

## 3. Đường Đi Của Yêu Cầu & Vòng Đời Dữ Liệu (Request Lifecycle & Data Flow)

### Kịch Bản Trọng Điểm: Người Dùng Mua Vé Concert Trong Giờ Cao Điểm

| Giai đoạn | Điểm chạm (Touchpoint) | Hành động xử lý | Dữ liệu trao đổi |
|---|---|---|---|
| **1. Truy cập ban đầu** | Client → Cloudflare Edge | Tải mã nguồn ứng dụng Web/App; kiểm tra bot qua Turnstile CAPTCHA | Nhận HTML, CSS, JS Bundle từ Edge Cache (< 20 ms) |
| **2. Xếp hàng chờ ảo** | Client → API Gateway → `queue` Module | Kiểm tra thời gian mở bán; cấp số thứ tự ngẫu nhiên trong Redis Sorted Set | Client nhận số thứ tự và bắt đầu polling trạng thái |
| **3. Cấp quyền vào chọn ghế** | `queue` Module → Client | Đến lượt người dùng; cấp thẻ `queue_token` có chữ ký điện tử HMAC | Client nhận `queue_token` (hiệu lực 15 phút) |
| **4. Xem sơ đồ ghế** | Client → `booking` Module → Redis/MySQL | Tải cấu trúc khán đài và trạng thái ghế trống/đang giữ/đã bán | Nhận JSON trạng thái ghế (từ Redis cache hoặc MySQL) |
| **5. Giữ ghế (Hold Seat)** | Client → API Gateway → `booking` Module → Redis | Thực thi `SETNX EX 600` khóa ghế; ghi bản ghi `seat_holds` vào MySQL (cùng @Transactional) | Giữ ghế thành công; phát WebSocket đổi màu ghế sang Vàng |
| **6. Đặt hàng & Thanh toán** | Client → `order` Module → `payment` Module → VNPay | Kiểm tra khóa giữ ghế còn hạn; tạo đơn PENDING; khởi tạo giao dịch; chuyển hướng sang cổng thanh toán | Nhận URL thanh toán VNPay kèm mã giao dịch |
| **7. Xác nhận giao dịch** | VNPay → API Gateway → `payment` Module (IPN) | Cổng thanh toán gọi IPN callback; xác minh chữ ký SHA256; cập nhật Payment→Order→Ticket trong 1 @Transactional | Cập nhật đơn PAID, ghế SOLD, phát hành vé QR |
| **8. Thông báo & Báo cáo** | `payment` Module → PaymentSuccessEvent → `notification` Module (@Async) | Sinh mã Dynamic QR vé điện tử; gửi Email xác nhận kèm PDF; gửi Push Notification (bất đồng bộ) | Khách hàng nhận email vé; vé hiển thị trên ứng dụng di động |

---

## 4. Đặc Tả Chi Tiết Quy Trình Xử Lý Tại API Gateway (Pipeline Execution)

Mọi yêu cầu HTTP/RESTful từ phía máy khách khi đi qua API Gateway đều phải trải qua 6 bước xử lý tuần tự:

1. **Correlation ID Filter:** Kiểm tra tiêu đề `X-Request-ID`. Nếu chưa có, Gateway tự động sinh mã UUID v4 mới và đính kèm vào ngữ cảnh để theo vết xuyên suốt các module trong ứng dụng Monolith.
2. **CORS & Security Headers Filter:** Thêm các tiêu đề bảo mật: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security: max-age=31536000`.
3. **Rate Limiting Filter:** Tra cứu bộ đếm IP/User trong Redis. Nếu vượt quá ngưỡng (> 120 req/phút) → Trả về mã lỗi `429 Too Many Requests` ngay tại Gateway mà không gọi vào ứng dụng Backend.
4. **Authentication & Queue Token Filter:**
   - Với các API công khai (`/api/v1/concerts`): Cho phép đi qua trực tiếp.
   - Với các API bảo vệ (`/api/v1/users/me`, `/orders`): Giải mã và kiểm tra hạn dùng của Access Token JWT (kiểm tra Redis Blacklist).
   - Với API giữ ghế (`/api/v1/seats/hold`): Kiểm tra thêm tiêu đề `X-Queue-Token` đảm bảo khách đã qua phòng chờ hợp lệ.
5. **Routing Filter:** Định tuyến toàn bộ request tới **một instance duy nhất** của ứng dụng Spring Boot Monolith (port 8080). Trong trường hợp Scale-Out (nhiều instance), Load Balancer phân phối theo Round-Robin/Least-Connection.
6. **Access Logging & Metrics Filter:** Ghi nhận mã trạng thái phản hồi, thời gian xử lý (latency), kích thước gói tin và xuất ra Prometheus Metrics / FluentBit.

---

## 5. Sơ Đồ Kiến Trúc Tổng Thể (Để Vẽ Trong Astah)

```
[Clients: Web SPA / Mobile App / Staff Check-in App / Admin Portal]
                        |
              [Cloudflare WAF + CDN]
                        |
              [Kong API Gateway / Nginx]
           (Rate Limit | JWT Auth | Queue Token | Correlation ID)
                        |
    ┌───────────────────────────────────────────────────────┐
    │          SPRING BOOT MODULAR MONOLITH (Port 8080)     │
    │  ┌──────┐ ┌───────┐ ┌─────────┐ ┌─────────┐          │
    │  │ auth │ │ queue │ │ concert │ │ booking │          │
    │  └──────┘ └───────┘ └─────────┘ └─────────┘          │
    │  ┌───────┐ ┌─────────┐ ┌────────┐ ┌────────┐         │
    │  │ order │ │ payment │ │ ticket │ │ review │         │
    │  └───────┘ └─────────┘ └────────┘ └────────┘         │
    │  ┌──────────────┐ ┌───────────┐                       │
    │  │ notification │ │ reporting │  (10 Modules)         │
    │  └──────────────┘ └───────────┘                       │
    └───────────────────────────────────────────────────────┘
              |                         |
    [Redis Cluster]              [MySQL 8.0 HA Cluster]
    (Queue, Locks,               (1 Primary + 2 Replicas)
     Cache, Rate Limit)
              |
    [MinIO/S3 Object Storage]  [OpenSearch (Full-Text + Audit)]
```
