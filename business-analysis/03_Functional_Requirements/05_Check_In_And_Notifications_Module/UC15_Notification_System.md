# UC15 — Hệ Thống Thông Báo Đa Kênh (Notification System)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC15 |
| **Tên Use Case** | Hệ Thống Thông Báo Đa Kênh (Notification System) |
| **Module** | 05 — Check-In & Notifications Module |
| **Actor chính** | System (Tự động kích hoạt theo sự kiện), Admin, Organizer |
| **Actor phụ** | Khách hàng (Customer - Người nhận), Email Gateway (SES/SendGrid), Push Service (FCM/APNs), SMS Gateway (Twilio/Brandname SMS) |
| **Mức độ ưu tiên** | 🔴 High |
| **Precondition** | Người nhận có tài khoản hoặc thông tin liên lạc hợp lệ (email, số điện thoại, FCM token); sự kiện nghiệp vụ được phát sinh từ các service nội bộ |
| **Postcondition** | Thông điệp được phân phối qua các kênh đã cấu hình; trạng thái gửi (Sent/Delivered/Read/Failed) được lưu vết đầy đủ trong Audit Log |
| **BR liên quan** | BR-NOTI-001 (Thông báo giao dịch bắt buộc), BR-NOTI-002 (Kênh dự phòng khi lỗi gửi tin), BR-NOTI-003 (Chống gửi tin lặp - Deduplication) |

---

## 2. Tóm Tắt

Use case này đặc tả kiến trúc và quy trình xử lý của Hệ thống Thông báo Đa kênh (Omni-channel Notification Engine). Hệ thống tiếp nhận các sự kiện nghiệp vụ từ toàn bộ nền tảng (xác thực OTP, đặt hàng thành công, cảnh báo sắp hết giờ giữ ghế, thông báo hủy/đổi lịch concert, thông báo nhắc nhở trước giờ diễn) và tự động định tuyến thông điệp qua các kênh: **In-App Notification**, **Mobile/Web Push**, **Email Giao dịch**, và **SMS Brandname**.

---

## 3. Luồng Chính A — Xử Lý Gửi Thông Báo Giao Dịch Tự Động (Transactional Notification)

| Bước | Actor / Thành phần | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Microservice nghiệp vụ | Phát sinh sự kiện (Event) lên Message Broker (Spring ApplicationEvent: `notification.events`) kèm payload (Mã sự kiện: `ORDER_PAID_SUCCESS`, ID người nhận, Dữ liệu đơn hàng, Deduplication Key) | Message Broker tiếp nhận và lưu sự kiện vào phân vùng |
| 2 | Notification Consumer | Đọc sự kiện từ topic, kiểm tra trùng lặp thông báo dựa trên `deduplication_key` trong Redis Cache (TTL = 24 giờ) | Nếu đã tồn tại key → Bỏ qua (chống spam trùng). Nếu chưa → Tiếp tục xử lý |
| 3 | System | Tra cứu cấu hình người nhận (User Preferences): Kiểm tra quyền riêng tư, danh sách kênh đăng ký, ngôn ngữ ưa thích (vi/en) | Xác định danh sách kênh cần gửi: Email + In-App + Push Notification |
| 4 | System | Nạp mẫu thông điệp (Notification Template), biên dịch dữ liệu động (Tên khách, Mã vé, Tên concert, Số ghế, Tổng tiền, Mã QR vé đính kèm) | Tạo nội dung hoàn chỉnh cho từng kênh |
| 5 | Dispatcher | Chuyển tiếp tác vụ gửi tới các Adapter kênh tương ứng (Email Worker, Push Worker, In-App DB Worker) | Thực hiện gọi API đối tác (AWS SES / Firebase Cloud Messaging) song song |
| 6 | System | Ghi nhận trạng thái gửi vào cơ sở dữ liệu `notification_logs`: Trạng thái `SENT`, mã phản hồi đối tác, timestamp | Hoàn tất trong vòng $\le 2\text{ giây}$ kể từ lúc phát sinh sự kiện |

---

## 4. Luồng Chính B — Admin / Organizer Phát Sóng Thông Báo Khẩn Hàng Loạt (Broadcast Emergency)

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Admin hoặc Organizer | Truy cập màn hình "Gửi thông báo sự kiện" trên Portal Quản trị | Hiển thị form tạo chiến dịch thông báo |
| 2 | Actor | Chọn đối tượng nhận: *"Tất cả khán giả đã mua vé concert X"* hoặc *"Toàn bộ người dùng nền tảng"* | Hệ thống đếm nhanh số lượng người nhận dự kiến (VD: 45.000 khách hàng) |
| 3 | Actor | Chọn mức độ ưu tiên: `CRITICAL` (Bắt buộc gửi qua Push + Email + SMS khẩn); Nhập tiêu đề và nội dung (VD: *"Thông báo hoãn concert do bão"*); Nhập mật khẩu xác thực 2FA | Hệ thống kiểm tra quyền hạn và validate nội dung |
| 4 | Actor | Nhấn "Phát sóng ngay lập tức" | Hệ thống tạo chiến dịch Broadcast, chia nhỏ danh sách người nhận thành các batch (1.000 người/batch) đẩy vào hàng đợi ưu tiên cao |
| 5 | Batch Workers | Hàng chục worker xử lý song song gửi thông báo đồng loạt | Hiển thị thanh tiến độ gửi theo thời gian thực trên màn hình Admin (Đã gửi: X%, Thành công: Y, Thất bại: Z) |

---

## 5. Luồng Thay Thế (Alternative Flows)

### AF-01: Kênh Dự Phòng Tự Động (Channel Fallback Cascading)
| Bước | Mô tả |
|---|---|
| 1 | Hệ thống kích hoạt gửi cảnh báo thanh toán hoặc mã vé qua Push Notification tới điện thoại người dùng. |
| 2 | Nhà mạng hoặc Google FCM trả về mã lỗi thiết bị không online hoặc Token không hợp lệ (`DEVICE_UNREGISTERED`). |
| 3 | Hệ thống bắt sự kiện thất bại và tự động kích hoạt **Kênh Dự Phòng Cấp 1 (Fallback to Email)**. |
| 4 | Trường hợp đây là tin khẩn cấp/OTP mà Email cũng bị trễ hoặc khách hàng chưa mở sau 3 phút, hệ thống kích hoạt **Kênh Dự Phòng Cấp 2 (SMS Brandname)** để đảm bảo khách nhận được thông tin. |

### AF-02: Người Dùng Tùy Biến Cài Đặt Thông Báo (User Notification Preferences)
| Bước | Mô tả |
|---|---|
| 1 | Khách hàng vào mục "Cài đặt thông báo" trên ứng dụng di động. |
| 2 | Bật/Tắt các danh mục: Khuyến mãi vé mới, Nhắc nhở nghệ sĩ yêu thích, Bản tin tuần. |
| 3 | Khách hàng **không thể tắt** nhóm thông báo "Giao dịch & An toàn tài khoản" (Bắt buộc theo BR-NOTI-001). |
| 4 | Cấu hình mới có hiệu lực ngay lập tức cho các lần gửi thông báo tiếp theo. |

---

## 6. Luồng Ngoại Lệ (Exception Flows)

| ID | Điều kiện ngoại lệ | Cơ chế xử lý | Hành vi hệ thống |
|---|---|---|---|
| EX-01 | **Email bị dội (Hard Bounce / Mailbox Full)** | Nhà cung cấp Email (SES/SendGrid) trả về mã lỗi Bounce. Hệ thống ghi nhận trạng thái `BOUNCED` vào hồ sơ người dùng | Tạm khóa gửi email quảng cáo tới địa chỉ này; hiển thị cảnh báo đỏ trên trang cá nhân của khách hàng: *"Vui lòng cập nhật lại địa chỉ email hợp lệ"* |
| EX-02 | **Nhà mạng SMS từ chối do số thuê bao khóa 2 chiều hoặc chặn tin quảng cáo** | SMS Gateway trả về lỗi không thể chuyển phát | Lưu log lỗi `SMS_UNDELIVERED`; chuyển sang gửi qua Email và Notification In-App |
| EX-03 | **Nghẽn hàng đợi thông báo trong giờ mở bán cao điểm (Queue Congestion)** | Lưu lượng tạo đơn hàng tăng vọt tạo ra hàng trăm nghìn thông báo cùng lúc | Hệ thống kích hoạt cơ chế phân tách hàng đợi (Priority Queue Sharding): Hàng đợi `Critical` (OTP, Mã vé, Báo hết hạn ghế) được ưu tiên xử lý trước; hàng đợi `Marketing` bị hoãn lại chạy sau giờ cao điểm |
| EX-04 | **Gửi trùng lặp do mạng chập chờn (Duplicate Dispatch)** | Consumer nhận lại sự kiện trùng lặp từ Kafka do rebalance hoặc retry | Kiểm tra khóa Deduplication `noti:dedup:{event_id}:{user_id}` trong Redis. Nếu đã tồn tại thì hủy tác vụ và trả về kết quả thành công giả lập (Ack message) |

---

## 7. Ma Trận Kênh Thông Báo & Mức Độ Ưu Tiên (Notification Matrix)

| Loại sự kiện | Tên sự kiện | Mức ưu tiên | Kênh gửi mặc định | Kênh dự phòng | Người dùng có thể tắt? |
|---|---|---|---|---|---|
| **AUTH_OTP** | Mã OTP đăng ký / Đổi mật khẩu | 🔴 Critical | SMS, Email | — | ❌ Không |
| **SEAT_EXPIRY_WARN** | Cảnh báo sắp hết 10 phút giữ ghế (còn 2 phút) | 🔴 Critical | In-App Pop-up, Push | — | ❌ Không |
| **ORDER_SUCCESS** | Xác nhận đặt vé & Thanh toán thành công | 🔴 Critical | In-App, Email, Push | SMS | ❌ Không |
| **TICKET_ISSUED** | Phát hành mã QR vé điện tử | 🔴 Critical | In-App, Email | — | ❌ Không |
| **CONCERT_CANCEL** | Hủy hoặc thay đổi lịch biểu diễn khẩn cấp | 🔴 Critical | SMS, Email, Push | In-App Banner | ❌ Không |
| **REFUND_SUCCESS** | Hoàn tiền đơn hàng thành công | 🟡 High | Email, In-App, Push | — | ❌ Không |
| **SHOW_REMINDER** | Nhắc nhở trước giờ diễn ($T - 24$h, $T - 3$h) | 🟡 High | Push, Email, In-App | — | ✅ Có |
| **MARKETING_SALE** | Thông báo mở bán concert mới / Mã giảm giá | 🟢 Normal | Push, In-App | Email | ✅ Có |

---

## 8. User Stories & Acceptance Criteria

### US-15-01: Nhận Thông Báo Vé & Giao Dịch Tức Thì
> **As a** Khách hàng,  
> **I want to** nhận được thư xác nhận thanh toán kèm mã vé điện tử ngay khi chuyển tiền thành công,  
> **So that** tôi an tâm đơn hàng đã được bảo đảm và có đầy đủ thông tin chuẩn bị đi xem ca nhạc.

| Mã AC | Tiêu chí nghiệm thu (Acceptance Criteria) | Phân loại |
|---|---|---|
| AC-15-01-01 | Email xác nhận đặt vé và thông báo đẩy (Push Notification) phải được gửi đến thiết bị người dùng trong vòng không quá 5 giây kể từ khi cổng thanh toán ghi nhận thành công | Performance |
| AC-15-01-02 | Nội dung email hiển thị đầy đủ: Mã đơn hàng, Tên sự kiện, Thời gian biểu diễn, Địa điểm, Sơ đồ khu vực, Hạng vé, Số ghế, Tổng tiền và hướng dẫn vào cổng | Functional |
| AC-15-01-03 | Email có tệp đính kèm tóm tắt đơn hàng định dạng PDF chuẩn A4 có thể in trực tiếp | Usability |
| AC-15-01-04 | Đảm bảo tính duy nhất: Mỗi đơn hàng thành công chỉ gửi duy nhất một thông báo xác nhận, không bao giờ gửi trùng lặp email do retry mạng | Reliability |

---

### US-15-02: Cảnh Báo Sắp Hết Hạn Giữ Ghế (Seat Hold Countdown Alert)
> **As a** Khách hàng đang trong luồng thanh toán,  
> **I want to** nhận được cảnh báo khi thời gian giữ ghế 10 phút sắp hết (còn 2 phút),  
> **So that** tôi nhanh chóng hoàn tất thanh toán trước khi ghế bị giải phóng cho người khác.

| Mã AC | Tiêu chí nghiệm thu (Acceptance Criteria) | Phân loại |
|---|---|---|
| AC-15-02-01 | Khi đồng hồ đếm ngược còn đúng 120 giây, hệ thống kích hoạt thông báo âm thanh và pop-up cảnh báo màu cam nổi bật trên màn hình thanh toán | Usability |
| AC-15-02-02 | Nếu người dùng đang thu nhỏ tab hoặc chuyển app, gửi thông báo Web Push / Mobile Push nhắc nhở ngay lập tức | Functional |
| AC-15-02-03 | Khi hết 10 phút mà chưa thanh toán xong: Pop-up chuyển đỏ thông báo *"Ghế của bạn đã hết hạn giữ và được giải phóng"* kèm nút điều hướng quay về màn hình chọn ghế | Functional |

---

### US-15-03: Phát Sóng Thông Báo Khẩn Cấp Từ Ban Quản Trị
> **As an** Admin,  
> **I want to** phát đi thông báo khẩn cấp tới hàng chục nghìn khán giả của concert khi có thay đổi thời tiết hoặc địa điểm,  
> **So that** toàn bộ người tham gia nắm được thông tin kịp thời và tránh các rủi ro an ninh.

| Mã AC | Tiêu chí nghiệm thu (Acceptance Criteria) | Phân loại |
|---|---|---|
| AC-15-03-01 | Hệ thống hỗ trợ gửi broadcast tới tối thiểu 100.000 người dùng với tốc độ phân phối không dưới 5.000 thông điệp mỗi giây | Performance |
| AC-15-03-02 | Có dashboard theo dõi trực quan số lượng thông điệp: Đang gửi, Đã nhận, Thất bại và tỷ lệ mở xem (Open Rate) | Functional |
| AC-15-03-03 | Yêu cầu nhập mã xác thực OTP/2FA của Admin trước khi kích hoạt bất kỳ chiến dịch phát sóng diện rộng nào | Security |

---

## 9. Danh Sách Use Case Con & Phân Quyền Kỹ Thuật

| Mã Use Case Con | Tên Use Case Con | Actor Chính | Actor Phụ | Mục Đích Nghiệp Vụ |
|---|---|---|---|---|
| **UC15.1** | Gửi Mã OTP Xác Thực Giao Dịch / Đăng Ký | System | SMS Gateway, Email API | Bảo mật tài khoản & xác thực số điện thoại |
| **UC15.2** | Gửi Thông Báo Vé Điện Tử & Đơn Hàng Thành Công | System | Push Service, Email Worker | Xác nhận giao dịch mua vé |
| **UC15.3** | Cảnh Báo Sắp Hết Hạn Giữ Ghế (Real-Time Toast) | System | WebSocket, SSE, Push | Thúc đẩy hoàn tất thanh toán |
| **UC15.4** | Phát Sóng Thông Báo Khẩn Cấp Sự Kiện (Broadcast) | Admin, Organizer | Bulk Dispatcher Worker | Thông báo dời lịch, hủy show, an ninh |
| **UC15.5** | Xem & Đánh Dấu Đã Đọc Hộp Thư Thông Báo In-App | Khách hàng | Notification DB | Quản lý thông báo cá nhân trên App/Web |
| **UC15.6** | Cấu Hình Tùy Chọn Nhận Thông Báo (Notification Prefs) | Khách hàng | User Profile Service | Quản lý kênh nhận tin và quyền riêng tư |

