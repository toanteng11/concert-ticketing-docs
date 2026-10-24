# 01 — Kiến Trúc Tổng Thể Client-Server & API Gateway (High-Level Architecture)

## 1. Giới Thiệu & Mục Đích Tài Liệu

Tài liệu này đặc tả kiến trúc tổng thể cấp cao (**High-Level System Architecture**) của nền tảng bán vé sự kiện âm nhạc quy mô lớn. Tài liệu cung cấp toàn bộ các thành phần, các lớp kiến trúc (Architectural Layers), biên giới mạng, cơ chế tiếp nhận lưu lượng và đường đi của dữ liệu qua từng tầng để làm cơ sở cho kỹ sư hệ thống vẽ sơ đồ kiến trúc tổng thể (**High-Level Architecture Diagram**) và sơ đồ luồng dữ liệu (**Data Flow Diagram**).

---

## 2. Phân Tầng Kiến Trúc Hệ Thống (Layered Architecture Breakdown)

Hệ thống được tổ chức thành 7 tầng kiến trúc vật lý và logic phân tách rõ ràng, đảm bảo nguyên tắc trách nhiệm duy nhất (Single Responsibility) và khả năng mở rộng độc lập:

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
- **Công nghệ cốt lõi:** Cụm Kong Gateway hoặc Envoy Proxy chạy trên Kubernetes.
- **Chức năng tập trung:**
  - Giải mã SSL (TLS Termination) tập trung.
  - Xác thực chữ ký số JWT và kiểm tra cờ thu hồi phiên (Token Revocation Check).
  - Kiểm tra thẻ thông hành phòng chờ ảo (Virtual Queue Access Token).
  - Giới hạn tần suất gọi API (Rate Limiting & IP Throttling).
  - Chèn mã theo dõi phân tán (`X-Correlation-ID` / `traceparent`) vào mọi request.
  - Định tuyến thông minh (Dynamic Routing) tới các dịch vụ Backend theo đường dẫn URI.

### Tầng 4: Tầng Dịch Vụ Nghiệp Vụ (Microservices Business Layer)
- Cụm các dịch vụ phi trạng thái (Stateless Services) đóng gói trong Docker Container và điều phối bởi Kubernetes (EKS / GKE):
  - *Auth & User Service*
  - *Waiting Room & Queue Service*
  - *Concert & Catalog Service*
  - *Booking & Seat Reservation Service*
  - *Order & Checkout Service*
  - *Payment & Settlement Service*
  - *Ticket & Check-In Service*
  - *Notification Engine*
  - *Review & Social Service*

### Tầng 5: Tầng Bộ Đệm Nóng & Phân Tán (In-Memory Caching & Locking Layer)
- **Redis Cluster:** Phân cụm 6 nodes (3 Master + 3 Replica) chia mảnh (Sharding) tự động.
- Chịu trách nhiệm: Hàng chờ ảo (Sorted Sets), Khóa giữ ghế phân tán (`SETNX EX`), Bộ đếm Rate Limit, Cache danh mục sự kiện và Blacklist Token.

### Tầng 6: Tầng Luồng Sự Kiện Phân Tán (Event Streaming & Message Broker Layer)
- **Apache Kafka Cluster:** Chế độ KRaft gồm 3 Brokers phân tán, Replication Factor = 3.
- Chịu trách nhiệm: Phân phối sự kiện bất đồng bộ giữa các Microservices (`order.created`, `payment.success`, `ticket.issued`, `concert.cancelled`), tách rời hoàn toàn thời gian phản hồi API với các tác vụ xử lý nặng.

### Tầng 7: Tầng Lưu Trữ Bền Vững (Persistence & Storage Layer)
- **Cơ sở dữ liệu Giao dịch (MySQL 8.0 HA Cluster):** 1 Primary + 2 Read Replicas (InnoDB Engine) kết hợp HikariCP / ProxySQL Connection Pooling.
- **Kho Lưu Trữ Đối Tượng (AWS S3 / MinIO):** Lưu ảnh poster, sơ đồ ghế gốc, hóa đơn PDF, tệp xuất báo cáo lớn và bản sao lưu nhị phân Binlog.
- **Công Cụ Tìm Kiếm & Nhật Ký (OpenSearch / Elasticsearch):** Phục vụ tìm kiếm concert toàn văn (Full-Text Search) và lưu trữ nhật ký kiểm toán (Audit Logs).

---

## 3. Đường Đi Của Yêu Cầu & Vòng Đời Dữ Liệu (Request Lifecycle & Data Flow)

### Kịch Bản Trọng Điểm: Người Dùng Mua Vé Concert Trong Giờ Cao Điểm

| Giai đoạn | Điểm chạm (Touchpoint) | Hành động xử lý | Dữ liệu trao đổi |
|---|---|---|---|
| **1. Truy cập ban đầu** | Client $\rightarrow$ Cloudflare Edge | Tải mã nguồn ứng dụng Web/App; kiểm tra bot qua Turnstile CAPTCHA | Nhận HTML, CSS, JS Bundle từ Edge Cache ($< 20\text{ ms}$) |
| **2. Xếp hàng chờ ảo** | Client $\rightarrow$ API Gateway $\rightarrow$ Waiting Room Service | Kiểm tra thời gian mở bán; cấp số thứ tự ngẫu nhiên trong Redis | Client nhận số thứ tự và bắt đầu polling trạng thái |
| **3. Cấp quyền vào chọn ghế** | Waiting Room $\rightarrow$ Client | Đến lượt người dùng; cấp thẻ `queue_token` có chữ ký điện tử HMAC | Client nhận `queue_token` (hiệu lực 15 phút) |
| **4. Xem sơ đồ ghế** | Client $\rightarrow$ Booking Service $\rightarrow$ Redis/CDN | Tải cấu trúc khán đài và trạng thái ghế trống/đang giữ/đã bán | Nhận JSON diff trạng thái ghế |
| **5. Giữ ghế (Hold Seat)** | Client $\rightarrow$ API Gateway $\rightarrow$ Booking Service $\rightarrow$ Redis | Thực thi Lua Script `SETNX EX 600` khóa ghế; ghi bản ghi `seat_holds` vào PostgreSQL | Giữ ghế thành công; phát WebSocket đổi màu ghế sang Vàng |
| **6. Đặt hàng & Thanh toán** | Client $\rightarrow$ Order Service $\rightarrow$ Payment Service $\rightarrow$ VNPay | Kiểm tra khóa giữ ghế còn hạn; tạo đơn `PENDING`; chuyển hướng sang cổng thanh toán | Nhận URL thanh toán VNPay kèm mã giao dịch |
| **7. Xác nhận giao dịch** | VNPay $\rightarrow$ API Gateway $\rightarrow$ Payment Webhook | Cổng thanh toán gọi IPN callback; xác minh chữ ký bảo mật SHA256 | Cập nhật đơn `PAID`, ghế `SOLD`, bắn sự kiện Kafka `order.paid` |
| **8. Phát hành vé & Báo cáo** | Kafka $\rightarrow$ Ticket Worker & Notification Worker | Sinh mã Dynamic QR vé điện tử; gửi Email xác nhận kèm PDF; gửi Push Notification | Khách hàng nhận email vé; vé hiển thị trên ứng dụng di động |

---

## 4. Đặc Tả Chi Tiết Quy Trình Xử Lý Tại API Gateway (Pipeline Execution)

Mọi yêu cầu HTTP/RESTful từ phía máy khách khi đi qua API Gateway đều phải trải qua 6 bước xử lý tuần tự (Gateway Middleware Pipeline):

1. **Correlation ID Filter:** Kiểm tra tiêu đề `X-Request-ID`. Nếu chưa có, Gateway tự động sinh mã UUID v4 mới và đính kèm vào ngữ cảnh (Context) để theo vết phân tán xuyên suốt các Microservices.
2. **CORS & Security Headers Filter:** Thêm các tiêu đề bảo mật: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security: max-age=31536000`.
3. **Rate Limiting Filter:** Tra cứu bộ đếm IP/User trong Redis. Nếu vượt quá ngưỡng (ví dụ $> 120\text{ req/phút}$) $\rightarrow$ Trả về mã lỗi `429 Too Many Requests` ngay tại Gateway mà không gọi vào dịch vụ Backend.
4. **Authentication & Queue Token Filter:**
   - Với các API công khai (`/api/v1/concerts`): Cho phép đi qua trực tiếp.
   - Với các API bảo vệ (`/api/v1/users/me`, `/orders`): Giải mã và kiểm tra hạn dùng của Access Token JWT.
   - Với API giữ ghế (`/api/v1/seats/hold`): Kiểm tra thêm tiêu đề `X-Queue-Token` đảm bảo khách đã qua phòng chờ hợp lệ.
5. **Circuit Breaker & Routing Filter:** Định tuyến yêu cầu tới đúng cụm Pod của Microservice tương ứng dựa trên Service Discovery (CoreDNS). Nếu service đích đang bị lỗi liên tục, Circuit Breaker trả về phản hồi suy giảm duyên dáng.
6. **Access Logging & Metrics Filter:** Ghi nhận mã trạng thái phản hồi, thời gian xử lý (latency), kích thước gói tin và xuất ra Prometheus Metrics / FluentBit.
