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
| 13 | Báo cáo | Báo cáo doanh thu/vé bán theo concert và thời gian; exports đồng bộ/bất đồng bộ theo phân quyền. Không gồm lịch gửi báo cáo định kỳ tự động trong baseline. |
| 14 | Đa nền tảng | Web (SPA), iOS và Android đều dùng chung REST API backend |
| 15 | Báo cáo và quyết toán Organizer | Lập sổ settlement, yêu cầu và phê duyệt sau concert theo hợp đồng; Finance ghi nhận transfer reference/bằng chứng. Không bao gồm chuyển khoản tự động hay xác nhận custody/escrow (D-034) |

---

## 2. Ngoài Phạm Vi (Out of Scope)

Các mục sau đây **không thuộc phạm vi** dự án trừ khi có quyết định mở rộng được ghi nhận trong Decision Log:

- Xây dựng hoặc quản lý hạ tầng vật lý của địa điểm tổ chức.
- Tích hợp tự động chuyển khoản payout Organizer hoặc vận hành tài khoản escrow/custody; baseline chỉ lưu sổ quyết toán, phê duyệt và bằng chứng chuyển tiền. Mở rộng cần chọn provider, hợp đồng và quyết định mới (D-034).
- Cấu hình lịch gửi báo cáo định kỳ qua email: chưa nằm trong baseline; không có report_schedules table hoặc tiêu chí nghiệm thu trong release hiện tại.
- Cài đặt nội bộ của nhà cung cấp thanh toán (VNPay, MoMo).
- Chuyển nhượng hoặc bán lại vé giữa các tài khoản khách hàng.
- Chương trình khách hàng thân thiết, tích điểm và ưu đãi theo nhóm.
- Voucher/mã giảm giá, chiến dịch ưu đãi và báo cáo voucher: ngoài baseline theo D-031; không triển khai API, bảng dữ liệu hoặc tiêu chí nghiệm thu voucher trong release này.
- Thiết kế chi tiết script triển khai hạ tầng và source code.
- Quét QR offline tại cổng: ngoài baseline theo D-023. Khi không thể xác thực online, scanner fail-closed, dừng xác nhận tự động và báo Gate Supervisor; không chấp nhận vé chỉ từ cache cục bộ.
- Chiến dịch marketing và hệ thống CRM.

---

## 3. Giả Định (Assumptions)

Các giả định dưới đây được coi là **đúng** trong suốt quá trình phân tích và xây dựng. Nếu bất kỳ giả định nào thay đổi, phạm vi và thiết kế cần được xem xét lại.

| ID | Giả định |
|---|---|
| A-01 | Mỗi tài khoản người dùng có tối đa **một** vị trí hàng chờ cho mỗi concert, bất kể mở bao nhiêu tab hoặc thiết bị. |
| A-02 | Một khách hàng có thể giữ tối đa **4 ghế** cho một lần đặt vé. |
| A-03 | Lệnh giữ ghế hết hạn sau **10 phút** nếu thanh toán không hoàn thành trong khoảng thời gian hợp lệ. |
| A-04 | Lượt vào khu vực chọn ghế hết hạn sau **2 phút** không hoạt động; đây là timeout hàng chờ/phiên, độc lập với seat hold 10 phút theo D-002. Ghế đang giữ không tự hết hạn sớm hơn TTL đã chốt. |
| A-05 | Vé QR là **đơn dụng** (single-use) và chỉ có thể check-in một lần duy nhất. |
| A-06 | Theo D-005/D-024: ngày lịch tại timezone địa điểm; ≥7 ngày: 100%, 3–6 ngày: 50%, 0–2 ngày: 0%. 50% làm tròn half-up đến 1 VND trên tổng giá vé đủ điều kiện. |
| A-07 | Ghế có trạng thái SOLD chỉ quay về AVAILABLE khi đơn được hoàn tiền trong khi concert còn trong giai đoạn mở bán. |
| A-08 | Khu vực chọn ghế hỗ trợ tối đa **5.000 người dùng đồng thời**. |
| A-09 | Baseline phải hỗ trợ **100.000 CCU**; 500.000 CCU là mục tiêu tương lai ngoài release theo D-022. |
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
| Hợp đồng Organizer/Platform + Legal/Finance | Chốt tỷ lệ/căn cứ phí, chi phí refund, điều kiện/timing payout, custody và phương thức chuyển theo D-034 | Không mở payout production; điều khoản chưa được xác nhận |

| VNPay API | Xử lý thanh toán và hoàn tiền qua VNPay | Không thể hoàn tất thanh toán qua VNPay |
| MoMo API | Xử lý thanh toán và hoàn tiền qua MoMo | Không thể hoàn tất thanh toán qua MoMo |
| Email / Push provider | Gửi thông báo email và push notification | Mất kênh thông báo, khách không nhận được cập nhật quan trọng |
| Identity / Auth service | Xác thực người dùng, quản lý session, OAuth2 | Không thể đăng nhập hoặc xác thực quyền |
| Organizer cung cấp dữ liệu | Thông tin địa điểm, sơ đồ ghế, giá vé đúng định dạng | Không thể tạo concert hợp lệ |
| Thiết bị quét QR tại cổng | Kết nối mạng và thiết bị phần cứng tại địa điểm | Không thể thực hiện check-in |
| Redis | Caching, session phòng chờ, giữ ghế (SET NX EX), rate limiting | Chậm hơn nhưng fallback sang DB-level lock |
| RabbitMQ | Durable queue cho notification và xử lý bất đồng bộ; publisher confirms, retry/DLQ, idempotent consumer | Notification chậm hoặc lỗi chuyển vào retry/DLQ; giao dịch nghiệp vụ vẫn được lưu bền vững |
| CDN (Cloudflare R2, AWS CloudFront) | Phân phối ảnh poster, banner, file vé | Tải chậm hơn cho người dùng xa server |
| SQL Server | Lưu trữ chính: đơn hàng, vé, concert (ACID) | Mất dữ liệu giao dịch |

---

## 6. Câu Hỏi Mở

Các quyết định baseline và cổng go-live có thẩm quyền nằm trong [Decision Log](./03_Risks_Open_Questions_And_Decision_Log.md). Mọi thay đổi sau này phải cập nhật decision log và các artifact bị ảnh hưởng.

---

## 7. Tài Liệu Liên Quan

- [`00_Project_Governance/01_Stakeholders_And_Business_Objectives.md`](./01_Stakeholders_And_Business_Objectives.md)
- [`00_Project_Governance/03_Risks_Open_Questions_And_Decision_Log.md`](./03_Risks_Open_Questions_And_Decision_Log.md)
- [`01_Business_Overview/02_Actor_Permissions.md`](../01_Business_Overview/02_Actor_Permissions.md)
- [`01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md`](<../01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md>)


