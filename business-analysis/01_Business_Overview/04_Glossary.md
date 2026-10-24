# Từ Điển Thuật Ngữ (Glossary)

## Mục đích

Tài liệu này định nghĩa thống nhất tất cả thuật ngữ, trạng thái, viết tắt và khái niệm kỹ thuật được sử dụng trong toàn bộ tài liệu BA của hệ thống đặt vé concert. Mọi thành viên dự án — BA, Developer, QA, Designer — phải dùng chung từ điển này để tránh hiểu nhầm.

---

## 1. Trạng Thái Đối Tượng (Entity Status Codes)

### 1.1 Trạng thái Concert (`concert.status`)

| Mã trạng thái | Tên tiếng Việt | Mô tả |
|---|---|---|
| `PENDING_REVIEW` | Chờ duyệt | Concert vừa được tạo bởi Organizer, đang chờ Admin phê duyệt trong 48 giờ |
| `PUBLISHED` | Đã công bố | Admin đã duyệt; concert hiển thị công khai nhưng chưa đến giờ mở bán |
| `ON_SALE` | Đang mở bán | Đã đến giờ mở bán; phòng chờ và khu chọn ghế hoạt động |
| `SOLD_OUT` | Hết vé | Toàn bộ ghế đã được bán; hàng chờ đóng lại |
| `CLOSED` | Đóng bán | Ban tổ chức hoặc hệ thống đóng bán trước khi hết vé |
| `COMPLETED` | Đã diễn ra | Concert đã diễn ra xong; khách hàng có thể đánh giá |
| `CANCELLED` | Đã hủy | Concert bị hủy bởi Admin (bất kỳ lúc trước COMPLETED) hoặc Organizer (trong điều kiện cho phép) |

### 1.2 Trạng thái Ghế (`seat.status`)

| Mã trạng thái | Tên tiếng Việt | Mô tả |
|---|---|---|
| `AVAILABLE` | Trống | Ghế chưa được chọn, sẵn sàng cho khách hàng |
| `HELD` | Đang giữ | Có khách đang giữ ghế; khóa tối đa 10 phút. Chỉ một khách hàng tại một thời điểm |
| `SOLD` | Đã bán | Thanh toán thành công; ghế thuộc về khách hàng — có vé tương ứng |
| `BLOCKED` | Khóa | Ghế bị khóa bởi hệ thống hoặc ban tổ chức (ghế VIP dành riêng, ghế hỏng...) |

### 1.3 Trạng thái Đơn hàng (`order.status`)

| Mã trạng thái | Tên tiếng Việt | Mô tả |
|---|---|---|
| `PENDING` | Chờ thanh toán | Đơn đã được tạo trong khi seat hold còn hiệu lực; chờ khách thanh toán |
| `PAID` | Đã thanh toán | Thanh toán thành công; vé đã được phát hành (ticket.status = ORDERED) |
| `REFUND_PENDING` | Yêu cầu hoàn tiền | Khách đã gửi yêu cầu hủy đơn PAID; đang xử lý hoàn tiền với payment provider |
| `REFUNDED` | Đã hoàn tiền | Tiền đã hoàn về tài khoản khách theo đúng chính sách |
| `OUT_TIME` | Hết hạn | Đơn PENDING nhưng quá 10 phút không hoàn tất thanh toán; ghế tự động trả về AVAILABLE |
| `CANCELLED` | Đã hủy | Đơn bị hủy trực tiếp (khách hủy đơn PENDING hoặc concert bị hủy) |

### 1.4 Trạng thái Vé (`ticket.status`)

| Mã trạng thái | Tên tiếng Việt | Mô tả |
|---|---|---|
| `ORDERED` | Đã phát hành | Vé được tạo sau khi thanh toán thành công; QR sẵn sàng để check-in |
| `CHECKED` | Đã check-in | Vé đã được quét thành công tại cổng vào; không thể check-in lại |
| `CANCELLED` | Đã hủy | Vé bị hủy do đơn hàng được hoàn tiền hoặc concert bị hủy |

### 1.5 Trạng thái Giao dịch thanh toán (`payment_transaction.status`)

| Mã trạng thái | Tên tiếng Việt | Mô tả |
|---|---|---|
| `INITIATED` | Khởi tạo | Yêu cầu thanh toán vừa được gửi đến payment provider |
| `PROCESSING` | Đang xử lý | Payment provider đang xử lý giao dịch |
| `SUCCESS` | Thành công | Giao dịch hoàn tất, tiền đã được trừ |
| `FAILED` | Thất bại | Giao dịch thất bại (không đủ tiền, lỗi mạng, hết hạn phiên) |
| `REFUND_PROCESSING` | Đang hoàn tiền | Yêu cầu hoàn tiền đã gửi đến provider, chờ xác nhận |
| `REFUNDED` | Đã hoàn tiền | Tiền đã hoàn về tài khoản khách hàng |

---

## 2. Khái Niệm Nghiệp Vụ

| Thuật ngữ (EN) | Thuật ngữ (VI) | Định nghĩa chi tiết |
|---|---|---|
| **Seat Hold** | Giữ ghế | Hành động tạm thời khóa một ghế (AVAILABLE → HELD) cho khách trong 10 phút để hoàn tất thanh toán. Nếu quá thời gian → ghế tự về AVAILABLE. Tối đa 4 ghế mỗi khách. |
| **Waiting Room** | Phòng chờ | Khu vực ảo nơi khách hàng tập trung trước giờ mở bán 15 phút. Không ai có thể vào khu chọn ghế trước giờ này. |
| **Virtual Queue** | Hàng đợi ảo | Sau khi phòng chờ đóng (đúng giờ mở bán), toàn bộ người trong phòng chờ được xếp số thứ tự ngẫu nhiên. Người đến sau giờ mở bán xếp cuối hàng theo thứ tự đến. |
| **Queue Position** | Số thứ tự hàng chờ | Vị trí (số nguyên dương) của một tài khoản trong hàng đợi ảo. **Duy nhất per tài khoản per concert** — mở nhiều tab hay thiết bị vẫn chỉ có một số. |
| **Seat Map** | Sơ đồ ghế | Bản đồ tương tác của địa điểm tổ chức, hiển thị từng ghế với màu sắc phản ánh trạng thái real-time: xanh=AVAILABLE, vàng=HELD, đỏ=SOLD, xám=BLOCKED. |
| **Concert Lifecycle** | Vòng đời concert | Chuỗi chuyển trạng thái của concert từ PENDING_REVIEW đến COMPLETED hoặc CANCELLED. |
| **Refund Policy** | Chính sách hoàn tiền | Quy tắc: ≥7 ngày trước concert → hoàn 100%; 3–6 ngày → hoàn 50%; <3 ngày → không hoàn. Áp dụng cho cả Khách hủy đơn lẫn Admin hủy concert (Admin hủy thì luôn 100%). |
| **Late Payment** | Thanh toán trễ | Payment provider gửi callback SUCCESS sau khi seat hold đã hết hạn (>10 phút). Cần quy trình xử lý riêng: kiểm tra ghế còn available không, nếu không → hoàn tiền tự động. |
| **Concert Approval** | Duyệt concert | Quy trình Admin xem xét và phê duyệt concert trong ≤48 giờ. Giờ mở bán phải cách ngày duyệt ≥1 ngày. |
| **Cancellation Request** | Yêu cầu hủy | Khi Organizer muốn hủy concert đã PUBLISHED nhưng chưa ON_SALE, phải gửi yêu cầu hủy. Admin chấp thuận thì concert mới bị hủy. |
| **QR Ticket** | Vé QR | Vé điện tử chứa mã QR duy nhất, một-đối-một với ghế. Mỗi mã QR chỉ có thể quét thành công **một lần duy nhất**. |
| **Gate Check-in** | Check-in cổng vào | Nhân viên quét QR của khách → hệ thống validate → cập nhật ticket ORDERED→CHECKED → phản hồi ≤500ms. |
| **Seat Inventory** | Tồn kho ghế | Số lượng ghế AVAILABLE tại một thời điểm. Phải đảm bảo strong consistency: không bao giờ bán vượt quá số ghế thực tế. |

---

## 3. Khái Niệm Kỹ Thuật

| Thuật ngữ (EN) | Thuật ngữ (VI) | Định nghĩa |
|---|---|---|
| **Idempotency Key** | Khóa bất biến | UUID do client tạo, gửi kèm header mỗi request thanh toán. Server lưu kết quả theo key; cùng key gọi lại → trả kết quả cũ, không xử lý lại → không trừ tiền hai lần. |
| **Race Condition** | Xung đột đồng thời | Hai người dùng cùng thao tác trên cùng tài nguyên (cùng ghế), dẫn đến kết quả không nhất quán nếu không có cơ chế bảo vệ. |
| **Optimistic Lock** | Khóa lạc quan | Thêm cột `version` vào bảng DB. Khi update, kiểm tra version không đổi; nếu đổi → conflict → retry. Phù hợp tải ghi thấp. |
| **Pessimistic Lock** | Khóa bi quan | `SELECT ... FOR UPDATE` — khóa dòng DB ngay khi đọc, ngăn transaction khác. Dùng cho seat hold khi Redis fail. |
| **Redis SET NX EX** | Lệnh Redis giữ ghế | `SET seat:{concertId}:{seatId} {userId} NX EX 600` — chỉ set nếu key chưa tồn tại (NX), tồn tại 600 giây (EX). Tạo distributed lock cho seat hold. |
| **Dead Letter Queue** | Hàng đợi thư chết | Hàng đợi nhận message thất bại sau N retry. Dùng để giữ lại notification lỗi để debug và retry thủ công. |
| **Webhook Callback** | Callback webhook | HTTP POST mà payment provider gửi đến endpoint `/payment/callback` của hệ thống khi giao dịch hoàn tất. Phải validate chữ ký và xử lý idempotent. |
| **Signed URL** | URL có chữ ký | URL CDN chứa token chữ ký + thời hạn hết hạn. Dùng để serve file vé/QR image — chỉ người có link mới xem được, và chỉ trong thời hạn. |
| **JWT** | JSON Web Token | Token stateless sau đăng nhập. Payload: `{user_id, role, iat, exp}`. Server verify bằng secret key. Gửi qua header `Authorization: Bearer <token>`. |
| **RBAC** | Kiểm soát theo vai trò | Hệ thống phân quyền: mỗi role (CUSTOMER, ORGANIZER, ADMIN, STAFF) có tập permission cố định. API gateway enforce RBAC trước khi route request. |
| **CAP Theorem** | Định lý CAP | Hệ thống phân tán chỉ đảm bảo được 2/3: Consistency, Availability, Partition Tolerance. Hệ thống này chọn **CP** — ưu tiên tính nhất quán dữ liệu. |
| **Horizontal Scaling** | Scale ngang | Tăng số instance/server để xử lý tải; mỗi instance stateless để dễ scale. Ngược với vertical scaling (nâng cấp phần cứng). |
| **Rate Limiting** | Giới hạn tốc độ | Giới hạn số request/IP hoặc /user trong khoảng thời gian. Áp dụng chặt tại `/login`, `/payment`, `/seat-hold`. |
| **Message Queue** | Hàng đợi tin nhắn | Kafka hoặc RabbitMQ — dùng để tách luồng thanh toán khỏi luồng gửi thông báo. Đảm bảo at-least-once delivery với retry. |

---

## 4. Chỉ Số Đo Lường

| Viết tắt | Tên đầy đủ | Định nghĩa |
|---|---|---|
| **QPS** | Queries Per Second | Số request hệ thống xử lý mỗi giây |
| **CCU** | Concurrent Users | Số người dùng hoạt động đồng thời |
| **P99** | 99th Percentile Latency | 99% request xử lý nhanh hơn hoặc bằng giá trị này |
| **SLA** | Service Level Agreement | Cam kết uptime và latency với người dùng |
| **RTO** | Recovery Time Objective | Thời gian tối đa hệ thống có thể ngừng hoạt động |
| **RPO** | Recovery Point Objective | Lượng dữ liệu tối đa có thể mất (theo thời gian) |
| **TPS** | Transactions Per Second | Số giao dịch DB/payment hoàn tất mỗi giây |
| **CDN Hit Rate** | Tỷ lệ cache CDN | % request được phục vụ từ CDN thay vì origin server |

---

## 5. Các Actor

| Mã | Tên | Mô tả ngắn |
|---|---|---|
| `CUSTOMER` | Khách hàng | Người dùng cuối mua vé, quản lý đơn, đánh giá |
| `ORGANIZER` | Ban tổ chức | Tạo và quản lý concert |
| `ADMIN` | Quản trị viên | Duyệt concert, quản lý hệ thống |
| `STAFF` | Nhân viên quét vé | Quét QR tại cổng vào |
| `SYSTEM` | Hệ thống | Scheduler, background job, message consumer |
| `PAYMENT_PROVIDER` | Nhà cung cấp TT | VNPay, MoMo — xử lý giao dịch tài chính |

---

## 6. Viết Tắt Trong Tài Liệu

| Viết tắt | Nghĩa đầy đủ |
|---|---|
| BA | Business Analysis / Business Analyst |
| BR-xxx | Business Rule (Quy tắc nghiệp vụ) |
| UC-xx | Use Case |
| SM-xx | State Machine |
| BF-xx | Business Flow |
| NFR | Non-Functional Requirement |
| RTM | Requirements Traceability Matrix |
| ERD | Entity Relationship Diagram |
| DLQ | Dead Letter Queue |
| CCU | Concurrent Users |
| TT | Thanh Toán |

---

## 7. Tài Liệu Liên Quan

- [`01_Business_Context_And_System_Scope.md`](./01_Business_Context_And_System_Scope.md)
- [`02_Actor_Permissions.md`](./02_Actor_Permissions.md)
- [`03_Business_Rules_Catalog_(Business_Rules).md`](./03_Business_Rules_Catalog_(Business_Rules).md)
- [`02_State_Machines_And_Workflows/`](../02_State_Machines_And_Workflows/)

