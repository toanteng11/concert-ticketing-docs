# 06 — Ma Trận Truy Vết Yêu Cầu (Requirements Traceability Matrix - RTM)

## 1. Giới Thiệu & Mục Đích

Ma Trận Truy Vết Yêu Cầu (**Requirements Traceability Matrix — RTM**) là công cụ trung tâm liên kết chặt chẽ giữa **Quy tắc Nghiệp vụ (Business Rules - BR)**, **Yêu Cầu Chức Năng (Use Cases & User Stories)**, **Thực thể Dữ liệu (Database Entities)** và **Kịch bản Kiểm thử Nghiệm thu (Test Scenarios / Acceptance Criteria)**.

Tài liệu này được xây dựng nhằm phục vụ:
1. **System Architect & Designer:** Nắm bắt toàn bộ luồng dữ liệu, các bảng bị ảnh hưởng và biên giới dịch vụ khi vẽ Kiến trúc Microservices và ERD.
2. **Software Engineers:** Đối chiếu yêu cầu nghiệp vụ và các ràng buộc khi cài đặt logic API và trạng thái hệ thống.
3. **QA / Tester:** Thiết kế đầy đủ bộ Test Case bao phủ 100% các điều kiện biên, luồng ngoại lệ và quy tắc kinh doanh.
4. **Project Manager / Product Owner:** Kiểm soát phạm vi dự án, đảm bảo không có yêu cầu nào bị bỏ sót (Zero Requirement Gap).

---

## 2. Bảng Ma Trận Truy Vết Tổng Thể (Master Traceability Matrix)

| Mã BR | Tên Quy Tắc Nghiệp Vụ | Use Case Liên Quan | Mã User Story | Actor Thực Hiện | Bảng Dữ Liệu Bị Ảnh Hưởng | Phương Pháp Kiểm Thử & Tiêu Chí Nghiệm Thu |
|---|---|---|---|---|---|---|
| **BR-ACC-001** | Định dạng số điện thoại & mật khẩu chuẩn VN | UC01, UC02 | US-01-01, US-02-01 | Customer | `users`, `user_credentials` | Unit & Functional Test: Validate Regex SĐT VN, độ dài pass $\ge 8$ ký tự |
| **BR-ACC-002** | Cơ chế xác thực JWT, Refresh Token & Phiên | UC01, UC02, UC03 | US-01-02, US-02-02, US-03-02 | Customer, Admin | `user_sessions`, `refresh_tokens` | Security Test: Thu hồi token khi đổi pass hoặc bị khóa tài khoản |
| **BR-ACC-003** | Khóa tài khoản sau 5 lần nhập sai mật khẩu | UC01 | US-01-02 | Customer, System | `login_attempts`, `users` | Security Test: Thử đăng nhập sai 5 lần liên tiếp $\rightarrow$ Khóa 15 phút |
| **BR-CON-001** | Vòng đời trạng thái Concert 6 giai đoạn | UC11, UC12 | US-11-01, US-12-01, US-12-02 | Organizer, Admin | `concerts`, `concert_history` | State Transition Test: `DRAFT` $\rightarrow$ `PENDING` $\rightarrow$ `PUBLISHED` $\rightarrow$ `ON_SALE` $\rightarrow$ `COMPLETED` |
| **BR-CON-002** | Khóa chỉnh sửa thông tin khi Concert đang mở bán | UC11 | US-11-02 | Organizer | `concerts`, `ticket_zones` | Functional Test: Khi `ON_SALE`, chặn sửa giá vé, sơ đồ ghế và số lượng |
| **BR-CON-003** | Duyệt Concert bởi Admin trong vòng 48h (SLA) | UC12 | US-12-01 | Admin | `concerts`, `approval_logs` | Performance & SLA Test: Cảnh báo quá hạn 48h, lưu Audit Log duyệt |
| **BR-CON-004** | Quy định hủy Concert & Hoàn tiền tự động 100% | UC12, UC09 | US-12-02, US-09-02 | Admin, Organizer, System | `concerts`, `orders`, `refunds` | Integration Test: Hủy concert kích hoạt hoàn tiền tự động 100% cho toàn bộ vé |
| **BR-CON-005** | Giới hạn 2 concert hoạt động cho Organizer mới | UC03, UC11 | US-03-03, US-11-01 | Admin, Organizer | `organizer_profiles`, `concerts` | Business Logic Test: Chặn tạo concert thứ 3 nếu tài khoản chưa được xác minh |
| **BR-QUE-001** | Kích hoạt Phòng chờ ảo trước giờ mở bán 15 phút | UC04, UC05 | US-04-02 | System, Customer | `virtual_queues`, `waiting_room_sessions` | Load & Stress Test: Bắt đầu giữ hàng lúc $T - 15$ phút, phát số thứ tự ngẫu nhiên lúc $T$ |
| **BR-QUE-002** | Giới hạn 5.000 CCU đồng thời trong khu vực chọn ghế | UC05, UC06 | US-05-01, US-06-01 | System, Customer | Redis Queue, `active_selectors` | Load Test: 100K người xếp hàng, xả theo đợt 5.000 người vào vùng chọn ghế |
| **BR-QUE-003** | Thu hồi vị trí hàng chờ khi không hoạt động quá 2 phút | UC05, UC06 | US-05-02 | System, Customer | Redis Heartbeat | E2E Test: Để tab treo 120 giây không tương tác $\rightarrow$ Đẩy ra cuối hàng chờ |
| **BR-SET-001** | Tuyệt đối không bán trùng ghế (Zero Double-Booking) | UC06 | US-06-01, US-06-02 | Customer, System | `seats`, `seat_holds`, Redis Key | Concurrency Stress Test: 10.000 user cùng chọn 1 ghế $\rightarrow$ Duy nhất 1 người thành công |
| **BR-SET-002** | Giữ ghế tối đa 10 phút đếm ngược (Hold Timeout) | UC06, UC07, UC15 | US-06-01, US-07-01, US-15-02 | Customer, System | `seat_holds`, Redis TTL (600s) | Automated Test: Hết 600s chưa thanh toán $\rightarrow$ Tự động chuyển về `AVAILABLE` |
| **BR-SET-003** | Giới hạn số lượng tối đa 4 vé / người / giao dịch | UC06 | US-06-02 | Customer | `seat_holds`, `order_items` | Functional Test: Cố chọn 5 ghế $\rightarrow$ Báo lỗi vượt quá số lượng tối đa |
| **BR-ORD-001** | Tạo đơn hàng gắn liền với Khóa giữ ghế (Seat Hold Token) | UC06, UC07 | US-06-01, US-07-01 | Customer | `orders`, `order_items`, `seat_holds` | Integration Test: Chặn tạo đơn nếu Token giữ ghế không hợp lệ hoặc đã quá hạn |
| **BR-ORD-002** | Tính toán tổng tiền, chiết khấu và phí xử lý chính xác | UC07 | US-07-02 | Customer, System | `orders`, `vouchers` | Financial Unit Test: So sánh tổng tiền chi tiết, làm tròn đơn vị đồng |
| **BR-ORD-003** | Kiểm tra quyền sở hữu đơn hàng (Chống lỗ hổng IDOR) | UC08, UC09 | US-08-01, US-09-01 | Customer | `orders`, `tickets` | Security Audit: User A không thể xem đơn hàng hoặc vé của User B qua API |
| **BR-ORD-004** | Chính sách hủy đơn & Tỷ lệ hoàn tiền theo thời gian | UC09 | US-09-01, US-09-02 | Customer, Admin | `orders`, `refunds` | Business Logic Test: $\ge 7$ ngày = 100%, 3–6 ngày = 50%, $< 3$ ngày = 0% |
| **BR-PAY-001** | Bắt buộc sử dụng Idempotency Key trong giao dịch | UC07 | US-07-02 | Customer, Payment Gateway | `payment_transactions` | Network Chaos Test: Gọi lại URL thanh toán nhiều lần $\rightarrow$ Không tạo giao dịch trùng |
| **BR-PAY-002** | Tích hợp cổng thanh toán VNPay và MoMo với Webhook | UC07 | US-07-02 | Payment Gateway, System | `payment_transactions`, `payment_logs` | Webhook Integration Test: Xử lý IPN callback, xác minh chữ ký bảo mật SHA256 |
| **BR-PAY-003** | Xử lý thanh toán thành công sau khi ghế đã hết hạn giữ | UC07, UC09 | US-07-03, US-09-02 | System, Payment Gateway | `orders`, `refunds`, `seats` | Edge Case Test: Tiền trừ nhưng ghế đã bị người khác mua $\rightarrow$ Kích hoạt Auto-Refund 100% |
| **BR-PAY-004** | Thời hạn xử lý hoàn tiền tự động 3–5 ngày làm việc | UC09, UC12 | US-09-02, US-12-02 | System, Payment Gateway | `refunds`, `bank_transfers` | E2E Integration Test: Gọi Refund API đối tác, cập nhật trạng thái `REFUNDED` |
| **BR-PAY-005** | Đối soát giao dịch tự động hàng ngày lúc 02:00 sáng | UC13 | US-13-02 | Admin, System | `reconciliation_records` | Batch Job Test: So khớp dữ liệu nội bộ và file sao kê ngân hàng đối tác |
| **BR-TCK-001** | Mã QR Code động (Dynamic TOTP) thay đổi mỗi 30s | UC08, UC14 | US-08-02, US-14-01 | Customer, Check-in Staff | `tickets`, `qr_tokens` | Security Test: Ảnh chụp màn hình sau 30s bị từ chối tại cổng soát vé |
| **BR-TCK-002** | Nguyên tắc một lần quét duy nhất (Single-Scan Check-in) | UC14 | US-14-01 | Check-in Staff | `tickets`, `checkin_logs` | Real-time Test: Quét vé lần 2 tại cổng khác $\rightarrow$ Báo động đỏ `ALREADY_USED` trong $\le 300\text{ms}$ |
| **BR-TCK-003** | Vé hủy hoặc hoàn tiền không được phép vào cổng | UC14 | US-14-01 | Check-in Staff | `tickets`, `checkin_logs` | Gate Test: Vé trạng thái `CANCELLED` quét tại cổng bị từ chối lập tức |
| **BR-TCK-004** | Hỗ trợ soát vé chế độ Offline khi mất mạng Internet | UC14 | US-14-02 | Check-in Staff | Local SQLite DB, `sync_queue` | Resilience Test: Cắt mạng hoàn toàn $\rightarrow$ Quét ngoại tuyến và đồng bộ lại khi có mạng |
| **BR-REV-001** | Chỉ người tham gia thực tế (Verified Attendee) được đánh giá | UC10 | US-10-01 | Customer | `reviews`, `tickets` | Functional Test: Tài khoản chưa mua vé hoặc vé chưa qua cổng check-in bị chặn đánh giá |
| **BR-REV-002** | Đánh giá chỉ mở trong vòng 7 ngày sau khi concert kết thúc | UC10 | US-10-01 | Customer | `reviews`, `concerts` | Time-window Test: Đánh giá quá 7 ngày kể từ kết thúc sự kiện bị đóng |
| **BR-NOTI-001** | Bắt buộc gửi thông báo xác nhận giao dịch & mã vé | UC15 | US-15-01 | System | `notifications`, `notification_logs` | Performance Test: Email & Push gửi đến trong $\le 5\text{s}$ sau thanh toán thành công |
| **BR-NOTI-002** | Cơ chế chuyển kênh dự phòng (Channel Fallback) | UC15 | US-15-01 | System | `notifications`, Notification Router | Reliability Test: Push Notification lỗi $\rightarrow$ Tự động chuyển phát qua Email / SMS |
| **BR-NOTI-003** | Khử trùng lặp thông báo (Notification Deduplication) | UC15 | US-15-01 | System | Redis Deduplication Key | Chaos Test: Gửi trùng lặp message broker $\rightarrow$ Người dùng chỉ nhận 1 thông báo |

---

## 3. Ma Trận Phân Bổ Use Case Vào Các Microservice Kiến Trúc

Bảng phân bổ này giúp các kỹ sư kiến trúc phần mềm phân định ranh giới phụ trách của từng Microservice trong hệ thống:

| Tên Microservice | Use Cases Phụ Trách | Trách Nhiệm Chính | Cơ Sở Dữ Liệu Sở Hữu |
|---|---|---|---|
| **Auth & Account Service** | UC01, UC02, UC03 | Đăng ký, đăng nhập, JWT/OAuth2, phân quyền RBAC, hồ sơ người dùng | PostgreSQL (`users`, `user_credentials`, `user_sessions`) |
| **Concert & Catalog Service** | UC04, UC11, UC12 | Danh mục concert, nghệ sĩ, sơ đồ khán đài (zones, seats), duyệt sự kiện | PostgreSQL (`concerts`, `ticket_zones`, `venues`, `artists`) + Elasticsearch |
| **Waiting Room & Queue Service** | UC04, UC05 | Quản lý hàng chờ ảo, xếp hàng ngẫu nhiên, điều tiết lưu lượng truy cập | Redis Cluster (Sorted Sets, Virtual Queue Tokens) |
| **Booking & Reservation Service** | UC05, UC06 | Xem sơ đồ ghế real-time, khóa giữ ghế phân tán (Distributed Lock), timeout 10 phút | Redis (Seat Hold Locks) + PostgreSQL (`seat_holds`, `seats`) |
| **Order & Checkout Service** | UC07, UC08, UC09 | Quản lý vòng đời đơn hàng, áp dụng voucher, tính toán hóa đơn | PostgreSQL (`orders`, `order_items`, `vouchers`) |
| **Payment & Settlement Service** | UC07, UC09, UC13 | Tích hợp cổng VNPay/MoMo, xử lý webhook, hoàn tiền, đối soát tài chính | PostgreSQL (`payment_transactions`, `refunds`, `settlements`) |
| **Ticket & Check-In Service** | UC08, UC14 | Phát hành mã Dynamic QR vé điện tử, giải mã xác thực vào cổng tốc độ cao | Redis + PostgreSQL (`tickets`, `checkin_logs`) + Offline SQLite |
| **Review & Social Service** | UC10 | Đánh giá, xếp hạng sao, bình luận của khán giả thực tế (Verified Attendees) | PostgreSQL (`reviews`, `review_reports`) |
| **Notification Engine** | UC15 | Định tuyến thông báo đa kênh (Email, Push, SMS, In-App), chống spam | PostgreSQL (`notification_logs`, `templates`) + Apache Kafka |
| **Analytics & Reporting Service** | UC13 | Tổng hợp số liệu bán vé, doanh thu, tỷ lệ lấp đầy ghế theo thời gian thực | ClickHouse / PostgreSQL Read Replica |

---

## 4. Ma Trận Phủ Thử Nghiệm Nghiệm Thu (Acceptance Test Coverage)

| Nhóm Kiểm Thử | Số Lượng Use Case | Các Kịch Bản Trọng Điểm Cần Chú Ý | Loại Test Tương Ứng |
|---|---|---|---|
| **Bảo mật & Xác thực** | UC01, UC02, UC03, UC08, UC14 | SQL Injection, XSS, Brute-force mật khẩu, Đánh cắp Token, Lỗ hổng IDOR xem vé người khác, Chụp màn hình chia sẻ mã QR | Pen-test, Dynamic Application Security Testing (DAST) |
| **Độ tin cậy & Concurrency** | UC05, UC06, UC07 | Race condition khi 10.000 user cùng chọn 1 ghế VIP duy nhất; Callback thanh toán đến muộn sau khi ghế bị hủy; Quá hạn giữ ghế | Concurrency Stress Test, Chaos Engineering |
| **Hiệu năng & Tải cao** | UC04, UC05, UC06, UC14, UC15 | 100.000 người dùng đồng thời trong phòng chờ; Thời gian phản hồi quét vé tại cổng $\le 500\text{ms}$; Bắn 50.000 push notification trong 10 giây | Load Test (k6 / JMeter), Distributed Benchmarking |
| **Tích hợp Bên Thứ Ba** | UC07, UC09, UC15 | Ngắt kết nối mạng giữa chừng khi VNPay đang xử lý; MoMo trả về lỗi timeout; SMS Gateway từ chối số điện thoại | Mocking, Stubbing, Network Delay Simulation |
| **Toàn vẹn Nghiệp vụ Tài chính** | UC07, UC09, UC12, UC13 | Tiền hoàn trả tính toán đúng tỷ lệ theo ngày; Doanh thu đối soát không lệch 1 đồng; Không thanh toán thừa cho Organizer | Financial Audit Test, Database ACID Validation |
