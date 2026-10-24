# Rủi Ro, Câu Hỏi Mở và Nhật Ký Quyết Định

## Mục đích

Tài liệu này tập hợp các rủi ro nghiệp vụ và kỹ thuật đã được nhận diện, các câu hỏi còn chưa có câu trả lời rõ ràng, và nhật ký ghi lại những quyết định thiết kế quan trọng đã được thống nhất. Mục tiêu là đảm bảo mọi thành viên trong dự án đều biết những gì đã chốt, những gì còn đang chờ xác nhận, và ai chịu trách nhiệm cho mỗi quyết định.

---

## 1. Rủi Ro Đã Nhận Diện

| ID | Rủi ro | Mức độ tác động | Xác suất | Biện pháp giảm thiểu / Phản hồi |
|---|---|---|---|---|
| R-001 | Lưu lượng tăng đột biến khi mở bán vé (~100.000 CCU đồng thời) | Rất cao | Cao | Áp dụng phòng chờ (waiting room), rate limiting, load balancer và scale ngang (horizontal scaling) tự động |
| R-002 | Hai khách hàng cùng lúc chọn và giữ thành công cùng một ghế (race condition) | Rất cao | Trung bình | Hai lớp bảo vệ: Redis `SET NX EX` (lớp 1) + DB unique constraint / `SELECT FOR UPDATE` (lớp 2) |
| R-003 | Payment callback bị retry hoặc đến trễ sau khi ghế đã hết hạn giữ | Cao | Trung bình | Sử dụng Idempotency Key; xây dựng luồng xử lý late-payment riêng biệt với reconciliation |
| R-004 | Giao dịch thanh toán thất bại khi người dùng bấm nút nhiều lần hoặc mạng gián đoạn | Cao | Cao | Client gửi kèm idempotency key; server lưu kết quả theo key và trả lại kết quả cũ nếu gọi lại |
| R-005 | Giao hàng thông báo bị thất bại (email, push notification) | Trung bình | Trung bình | Message Queue (Kafka/RabbitMQ) với cơ chế retry tự động và dead-letter queue; hiển thị trạng thái trong ứng dụng |
| R-006 | Mã QR được sử dụng tại nhiều cổng vào cùng một lúc (duplicate check-in) | Rất cao | Thấp | Chuyển trạng thái vé sang `CHECKED` bằng atomic operation; từ chối mọi lần quét tiếp theo với thông báo "đã check-in" |
| R-007 | Redis bị lỗi hoặc khởi động lại (cache miss toàn bộ) | Cao | Thấp | Fallback sang DB-level lock; rate limit dùng bộ nhớ local cache tạm thời; vẫn đúng nhưng chậm hơn |
| R-008 | Database chính bị ngừng hoạt động trong giờ cao điểm | Rất cao | Rất thấp | Đọc từ replica; ghi bị từ chối rõ ràng với thông báo; đơn hàng đang xử lý nằm trong queue chờ DB hồi phục (ưu tiên CP trong CAP) |
| R-009 | Ban tổ chức không gửi thông tin sơ đồ ghế đúng định dạng | Trung bình | Trung bình | Validation schema nghiêm ngặt khi tạo concert; hiển thị lỗi rõ ràng cho organizer |
| R-010 | Admin không xử lý yêu cầu duyệt concert trong 48 giờ | Trung bình | Thấp | Alerting tự động nhắc nhở admin; hệ thống ghi nhận SLA violation |

---

## 2. Câu Hỏi Mở (Open Questions)

Các câu hỏi dưới đây **chưa có câu trả lời xác nhận** từ Product Owner hoặc các bên liên quan. Cần được giải quyết trước khi thiết kế chi tiết hoặc triển khai các luồng liên quan.

| ID | Câu hỏi | Ảnh hưởng đến | Người cần trả lời | Trạng thái |
|---|---|---|---|---|
| Q-001 | Định nghĩa chính xác của "hoạt động" trong phòng chờ để tính timeout 2 phút là gì? (heartbeat API call, thao tác UI, hay chỉ kết nối WebSocket?) | Luồng phòng chờ, cơ chế timeout | Product Owner | Chưa giải quyết |
| Q-002 | Nếu payment callback thành công đến SAU khi ghế đã hết hạn giữ (10 phút), hệ thống xử lý thế nào? Hoàn tiền tự động hay tạo đơn hàng mới? | Luồng thanh toán trễ, hoàn tiền | Product Owner / Payment team | Chưa giải quyết |
| Q-003 | Khi admin hủy concert, chính sách hoàn tiền 100% có áp dụng cho cả vé đã check-in (trạng thái CHECKED) không? | Luồng hủy concert admin, hoàn tiền | Product Owner / Legal | Chưa giải quyết |
| Q-004 | Một khách hàng có được phép tạo nhiều đơn hàng riêng biệt cho cùng một concert không? | Giới hạn đặt vé, kiểm soát tồn kho | Product Owner | Chưa giải quyết |
| Q-005 | Làm tròn số tiền hoàn trả (50%) theo quy tắc nào? Và nếu hoàn tiền một phần thất bại thì retry hay báo lỗi thủ công? | Luồng hoàn tiền, kế toán | Payment team / Finance | Chưa giải quyết |
| Q-006 | Hệ thống quét QR tại cổng vào có cần hỗ trợ chế độ offline (không có mạng) không? | Kiến trúc check-in, thiết bị phần cứng | Product Owner / Ops | Chưa giải quyết |
| Q-007 | Sau khi hết vé, khách hàng trong hàng chờ chưa đến lượt nhận thông báo qua kênh nào? (Push, Email, hay chỉ trong ứng dụng?) | Luồng thông báo, UX | Product Owner | Chưa giải quyết |
| Q-008 | Có cho phép Organizer sửa thông tin concert (không phải sơ đồ/giá) sau khi đã PUBLISHED nhưng chưa ON_SALE không? | Vòng đời concert | Product Owner | Chưa giải quyết |

---

## 3. Nhật Ký Quyết Định (Decision Log)

Các quyết định dưới đây đã được **thống nhất chính thức** và phải được tuân thủ trong suốt quá trình thiết kế và triển khai.

| ID | Ngày | Quyết định | Lý do | Chủ sở hữu |
|---|---|---|---|---|
| D-001 | — | Mỗi khách hàng được giữ tối đa **4 ghế** cho một lần đặt vé | Ngăn chặn độc chiếm tồn kho và đảm bảo công bằng cho các khách hàng khác | Product Owner |
| D-002 | — | Thời gian giữ ghế là **10 phút** kể từ khi khách hàng chọn ghế | Đủ thời gian hoàn tất thanh toán trong khi giải phóng tồn kho nếu khách bỏ dở | Product Owner |
| D-003 | — | Vé QR là **đơn dụng** (single-use), chỉ check-in được đúng một lần | Ngăn chặn tái sử dụng vé đã check-in, bảo đảm tính toàn vẹn của sự kiện | Product Owner |
| D-004 | — | Mọi yêu cầu thanh toán phải là **idempotent** thông qua Idempotency Key | Ngăn trừ tiền hai lần khi client retry hoặc mạng gián đoạn | Product Owner / Payment team |
| D-005 | — | Chính sách hoàn tiền: **100%** nếu ≥ 7 ngày; **50%** nếu 3–6 ngày; **0%** nếu < 3 ngày trước concert | Cân bằng giữa quyền lợi khách hàng và chi phí vận hành của ban tổ chức | Product Owner |
| D-006 | — | Hệ thống ưu tiên **CP (Consistency + Partition Tolerance)** trong mô hình CAP | Tính nhất quán dữ liệu ghế và giao dịch quan trọng hơn tính khả dụng tức thì khi có lỗi phân vùng | Kiến trúc sư hệ thống |
| D-007 | — | Dùng **hai lớp bảo vệ** chống bán trùng ghế: Redis `SET NX EX` (lớp 1) + DB constraint / pessimistic lock (lớp 2) | Redis có thể mất lock khi bị lỗi; DB là lớp bảo vệ cuối cùng đảm bảo tính nhất quán tuyệt đối | Kiến trúc sư hệ thống |
| D-008 | — | Thông báo được xử lý **bất đồng bộ** qua Message Queue (Kafka/RabbitMQ) với retry và dead-letter queue | Không làm chậm luồng thanh toán chính; đảm bảo ít nhất một lần giao hàng (at-least-once delivery) | Kiến trúc sư hệ thống |
| D-009 | — | Giờ mở bán phải cách ngày được duyệt **ít nhất 1 ngày** | Đảm bảo Admin có đủ thời gian duyệt và hệ thống chuẩn bị hạ tầng cho đợt bán | Product Owner |
| D-010 | — | Admin phải xử lý yêu cầu duyệt concert **trong vòng 48 giờ** | Đảm bảo ban tổ chức không bị kẹt chờ duyệt quá lâu, ảnh hưởng đến kế hoạch bán vé | Product Owner |

---

## 4. Hướng Dẫn Cập Nhật Tài Liệu

- **Quyết định mới** phải được thêm vào bảng Decision Log với ID duy nhất theo định dạng `D-###` và ghi rõ ngày, người quyết định.
- **Câu hỏi đã được giải quyết** phải được chuyển từ bảng Open Questions sang Decision Log.
- **Rủi ro mới** được thêm vào bảng Rủi ro với ID `R-###`, phải kèm biện pháp giảm thiểu cụ thể.
- Mọi quyết định thay đổi phạm vi phải được cập nhật đồng thời trong [`02_Scope_Assumptions_Constraints_And_Dependencies.md`](./02_Scope_Assumptions_Constraints_And_Dependencies.md).

---

## 5. Tài Liệu Liên Quan

- [`00_Project_Governance/01_Stakeholders_And_Business_Objectives.md`](./01_Stakeholders_And_Business_Objectives.md)
- [`00_Project_Governance/02_Scope_Assumptions_Constraints_And_Dependencies.md`](./02_Scope_Assumptions_Constraints_And_Dependencies.md)
- [`01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md`](../01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md)
- [`02_State_Machines_And_Workflows/`](../02_State_Machines_And_Workflows/)

