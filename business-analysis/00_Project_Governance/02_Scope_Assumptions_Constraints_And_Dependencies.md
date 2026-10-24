# Phạm Vi, Giả Định, Ràng Buộc và Phụ Thuộc

## Mục đích

Tài liệu này giới hạn rõ ràng những gì nằm trong và ngoài phạm vi xây dựng của hệ thống đặt vé concert trực tuyến, đồng thời liệt kê các giả định nền tảng, ràng buộc cứng và các phụ thuộc bên ngoài mà đội phát triển phải tính đến.

---

## 1. Trong Phạm Vi (In Scope)

Các chức năng và luồng nghiệp vụ sau đây **nằm trong phạm vi** phân tích và xây dựng của dự án:

| # | Nhóm chức năng | Mô tả tóm tắt |
|---|---|---|
| 1 | Xác thực & Tài khoản | Đăng ký, đăng nhập (email/mật khẩu, OAuth), quản lý thông tin cá nhân, đổi mật khẩu |
| 2 | Khám phá Concert | Xem danh sách, tìm kiếm theo từ khóa, lọc theo ngày / địa điểm / nghệ sĩ, xem chi tiết concert |
| 3 | Phòng chờ & Hàng đợi | Vào phòng chờ trước 15 phút, xếp hàng ngẫu nhiên khi giờ mở bán đến, quản lý số thứ tự duy nhất mỗi tài khoản |
| 4 | Sơ đồ ghế & Giữ ghế | Hiển thị sơ đồ ghế tương tác, giữ tối đa 4 ghế trong 10 phút, giải phóng ghế khi hết thời gian hoặc hủy chọn |
| 5 | Đặt hàng & Thanh toán | Tạo đơn hàng, thanh toán qua VNPay/MoMo, xử lý idempotency key, xử lý callback trễ/trùng từ payment provider |
| 6 | Vé điện tử & Check-in | Phát hành vé kèm mã QR khi thanh toán thành công, quét QR một lần tại cổng vào, cập nhật trạng thái vé CHECKED |
| 7 | Hủy đơn & Hoàn tiền | Hủy đơn PENDING/PAID, tính toán hoàn tiền theo chính sách thời gian, xử lý hoàn tiền qua payment provider, thông báo kết quả |
| 8 | Quản lý Concert (Organizer) | Tạo concert, gửi yêu cầu duyệt, xem trạng thái, gửi yêu cầu hủy, xem báo cáo doanh thu và vé bán |
| 9 | Quản lý Concert (Admin) | Duyệt/từ chối yêu cầu tạo concert, duyệt/từ chối yêu cầu hủy của organizer, hủy concert và kích hoạt hoàn tiền tự động |
| 10 | Quản lý Tài khoản (Admin) | Xem danh sách tài khoản, khóa/mở khóa, thay đổi phân quyền |
| 11 | Thông báo | Thông báo trong ứng dụng và email cho các sự kiện: thanh toán, hoàn tiền, hủy concert, hết vé, kết quả check-in |
| 12 | Đánh giá Concert | Khách hàng đánh giá concert sau khi concert đã hoàn thành (trạng thái COMPLETED) |
| 13 | Báo cáo | Báo cáo doanh thu và số vé bán theo concert, theo thời gian cho ban tổ chức và admin |
| 14 | Đa nền tảng | Web (SPA), iOS và Android đều dùng chung REST API backend |

---

## 2. Ngoài Phạm Vi (Out of Scope)

Các mục sau đây **không thuộc phạm vi** dự án trừ khi có quyết định mở rộng được ghi nhận trong Decision Log:

- Xây dựng hoặc quản lý hạ tầng vật lý của địa điểm tổ chức.
- Cài đặt nội bộ của nhà cung cấp thanh toán (VNPay, MoMo).
- Chuyển nhượng hoặc bán lại vé giữa các tài khoản khách hàng.
- Chương trình khách hàng thân thiết, tích điểm, mã giảm giá hoặc ưu đãi theo nhóm.
- Thiết kế chi tiết script triển khai hạ tầng và source code.
- Hỗ trợ quét QR offline (không có mạng) tại cổng vào.
- Chiến dịch marketing và hệ thống CRM.

---

## 3. Giả Định (Assumptions)

Các giả định dưới đây được coi là **đúng** trong suốt quá trình phân tích và xây dựng. Nếu bất kỳ giả định nào thay đổi, phạm vi và thiết kế cần được xem xét lại.

| ID | Giả định |
|---|---|
| A-01 | Mỗi tài khoản người dùng có tối đa **một** vị trí hàng chờ cho mỗi concert, bất kể mở bao nhiêu tab hoặc thiết bị. |
| A-02 | Một khách hàng có thể giữ tối đa **4 ghế** cho một lần đặt vé. |
| A-03 | Lệnh giữ ghế hết hạn sau **10 phút** nếu thanh toán không hoàn thành trong khoảng thời gian hợp lệ. |
| A-04 | Vị trí hàng chờ bị mất nếu khách hàng không hoạt động (inactive) quá **2 phút** trong phòng chờ. |
| A-05 | Vé QR là **đơn dụng** (single-use) và chỉ có thể check-in một lần duy nhất. |
| A-06 | Chính sách hoàn tiền được áp dụng cố định: **100%** nếu còn ≥ 7 ngày trước concert; **50%** nếu còn 3–6 ngày; **0%** nếu còn < 3 ngày. |
| A-07 | Ghế có trạng thái SOLD chỉ quay về AVAILABLE khi đơn được hoàn tiền trong khi concert còn trong giai đoạn mở bán. |
| A-08 | Khu vực chọn ghế hỗ trợ tối đa **5.000 người dùng đồng thời**. |
| A-09 | Lưu lượng cao điểm khi mở bán vé có thể đạt đến **100.000 concurrent users (CCU)**. |
| A-10 | Giờ mở bán phải cách thời điểm concert được duyệt **ít nhất 1 ngày**. |
| A-11 | Admin phải xử lý yêu cầu duyệt concert **trong vòng 48 giờ**. |

---

## 4. Ràng Buộc (Constraints)

Các ràng buộc cứng mà hệ thống **phải tuân thủ**:

| ID | Loại | Ràng buộc |
|---|---|---|
| C-01 | Kỹ thuật | Hệ thống phải hỗ trợ đồng thời Web, iOS và Android qua REST API. |
| C-02 | Quy mô | Nền tảng phục vụ khoảng **2 triệu người dùng** đăng ký. |
| C-03 | Tải cao điểm | Khu vực chọn ghế tối đa **5.000 CCU**; toàn hệ thống tối đa **~100.000 CCU** khi mở bán. |
| C-04 | Tính nhất quán | Không được phép xảy ra tình trạng bán trùng ghế dưới bất kỳ điều kiện race condition nào. |
| C-05 | Idempotency | Không giao dịch nào được trừ tiền khách hàng hai lần, dù client retry hay network interruption. |
| C-06 | Thời gian duyệt | Quản trị viên phải xử lý yêu cầu duyệt concert trong tối đa **48 giờ**. |
| C-07 | Thời gian mở bán | Giờ mở bán phải cách ngày được duyệt **ít nhất 1 ngày**. |
| C-08 | Bảo mật | Phải sử dụng JWT/OAuth2, HTTPS/TLS, cookie HttpOnly + Secure + SameSite; không lưu số thẻ thô. |
| C-09 | Vé QR | Mã QR vé chỉ được dùng **một lần**; mọi lần quét trùng đều bị từ chối và ghi log. |

---

## 5. Phụ Thuộc Bên Ngoài (Dependencies)

| Phụ thuộc | Mô tả | Rủi ro nếu không có |
|---|---|---|
| VNPay API | Xử lý thanh toán và hoàn tiền qua VNPay | Không thể hoàn tất thanh toán qua VNPay |
| MoMo API | Xử lý thanh toán và hoàn tiền qua MoMo | Không thể hoàn tất thanh toán qua MoMo |
| Email / Push provider | Gửi thông báo email và push notification | Mất kênh thông báo, khách không nhận được cập nhật quan trọng |
| Identity / Auth service | Xác thực người dùng, quản lý session, OAuth2 | Không thể đăng nhập hoặc xác thực quyền |
| Organizer cung cấp dữ liệu | Thông tin địa điểm, sơ đồ ghế, giá vé đúng định dạng | Không thể tạo concert hợp lệ |
| Thiết bị quét QR tại cổng | Kết nối mạng và thiết bị phần cứng tại địa điểm | Không thể thực hiện check-in |
| Redis | Caching, session phòng chờ, giữ ghế (SET NX EX), rate limiting | Chậm hơn nhưng fallback sang DB-level lock |
| Kafka / RabbitMQ | Message queue cho notification và xử lý bất đồng bộ | Thông báo bị trễ hoặc mất; tải đồng bộ tăng |
| CDN (Cloudflare R2, AWS CloudFront) | Phân phối ảnh poster, banner, file vé | Tải chậm hơn cho người dùng xa server |
| MySQL 8.0 | Lưu trữ chính: đơn hàng, vé, concert (ACID) | Mất dữ liệu giao dịch |

---

## 6. Các Điểm Cần Xác Nhận Thêm

| # | Vấn đề | Trạng thái |
|---|---|---|
| 1 | Định nghĩa chính xác của "hoạt động" trong phòng chờ để tính thời gian 2 phút timeout | Chưa xác định |
| 2 | Xử lý khi payment callback thành công đến sau khi ghế đã hết hạn giữ | Chưa xác định |
| 3 | Chính sách hoàn tiền khi admin hủy concert có áp dụng cho vé đã check-in không? | Chưa xác định |
| 4 | Khách hàng có thể tạo nhiều đơn hàng cho cùng một concert không? | Chưa xác định |
| 5 | Cách làm tròn số tiền hoàn trả và xử lý khi hoàn tiền một phần thất bại | Chưa xác định |
| 6 | Hệ thống có yêu cầu quét QR offline (không cần mạng) tại cổng vào không? | Chưa xác định |

---

## 7. Tài Liệu Liên Quan

- [`00_Project_Governance/01_Stakeholders_And_Business_Objectives.md`](./01_Stakeholders_And_Business_Objectives.md)
- [`00_Project_Governance/03_Risks_Open_Questions_And_Decision_Log.md`](./03_Risks_Open_Questions_And_Decision_Log.md)
- [`01_Business_Overview/02_Actor_Permissions.md`](../01_Business_Overview/02_Actor_Permissions.md)
- [`01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md`](../01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md)


