# Stakeholders và Mục Tiêu Kinh Doanh

## Mục đích

Tài liệu này xác định tất cả các bên liên quan (stakeholders) của hệ thống đặt vé concert trực tuyến, mô tả vai trò, trách nhiệm, mức độ ảnh hưởng của từng bên, đồng thời định nghĩa các mục tiêu kinh doanh dẫn dắt toàn bộ quá trình phân tích và xây dựng sản phẩm.

---

## 1. Stakeholders

### 1.1 Ma trận Stakeholder

| Stakeholder | Loại | Vai trò chính | Mức ảnh hưởng | Mức quan tâm |
|---|---|---|---|---|
| Khách hàng (Customer) | Bên ngoài | Khám phá concert, mua vé, nhận vé điện tử, yêu cầu hoàn tiền, check-in | Cao | Rất cao |
| Ban tổ chức (Organizer) | Bên ngoài | Tạo concert, cấu hình địa điểm & ghế, quản lý bán vé, xem báo cáo | Cao | Rất cao |
| Quản trị viên (Administrator) | Nội bộ | Duyệt concert, quản lý tài khoản, chấp thuận yêu cầu hủy, hủy concert | Rất cao | Cao |
| Nhân viên quét vé (Check-in Staff) | Bên ngoài | Quét mã QR, xác thực vé tại cổng vào concert | Thấp | Trung bình |
| Nhà cung cấp thanh toán (Payment Provider) | Bên ngoài | Xử lý thanh toán và hoàn tiền qua VNPay, MoMo | Cao | Thấp |
| Đội vận hành nền tảng (Platform Ops) | Nội bộ | Giám sát khả dụng, năng lực, sự cố và sức khỏe vận hành hệ thống | Rất cao | Cao |

---

### 1.2 Mô tả chi tiết từng Stakeholder

#### Khách hàng (Customer)
- **Đối tượng:** Người dùng cuối sử dụng nền tảng trên Web, iOS và Android — quy mô khoảng **2 triệu người dùng**.
- **Nhu cầu cốt lõi:**
  - Tìm kiếm, lọc và xem thông tin chi tiết concert (nghệ sĩ, địa điểm, thời gian, giá vé).
  - Tham gia phòng chờ trước giờ mở bán 15 phút; nhận số thứ tự hàng chờ ngẫu nhiên khi giờ mở bán đến.
  - Xem sơ đồ ghế tương tác, chọn và giữ tối đa 4 ghế trong 10 phút.
  - Thanh toán qua VNPay hoặc MoMo, nhận vé điện tử kèm mã QR (trạng thái ORDERED).
  - Xem lịch sử đơn hàng, lọc theo trạng thái và ngày.
  - Hủy đơn và nhận hoàn tiền theo chính sách (100% / 50% / 0% tùy thời điểm).
  - Đánh giá chất lượng concert **sau khi concert đã hoàn thành** (trạng thái COMPLETED).
- **Kỳ vọng chất lượng:** Trải nghiệm mượt mà, không bị bán trùng ghế, thông báo kịp thời về mọi thay đổi trạng thái đơn hàng và vé.

#### Ban tổ chức (Organizer)
- **Đối tượng:** Các công ty hoặc cá nhân tổ chức sự kiện âm nhạc sử dụng nền tảng để phát hành vé bán công khai.
- **Nhu cầu cốt lõi:**
  - Tạo concert với đầy đủ thông tin: nghệ sĩ, địa điểm, thời gian, sơ đồ ghế, giá vé và giờ mở bán.
  - Gửi yêu cầu duyệt đến quản trị viên và theo dõi trạng thái phê duyệt.
  - Xem báo cáo doanh thu và số vé đã bán theo thời gian thực.
  - Gửi yêu cầu hủy concert trong các trường hợp được phép.
- **Ràng buộc nghiệp vụ:**
  - **Không được sửa** sơ đồ ghế hoặc giá vé sau khi concert đã chuyển sang trạng thái **ON_SALE**.
  - Giờ mở bán phải cách thời điểm được duyệt **ít nhất 1 ngày**; hệ thống tự động cảnh báo nếu vi phạm.
  - Chỉ được tự hủy concert khi chưa được duyệt. Nếu đã duyệt nhưng chưa bán vé, phải gửi yêu cầu hủy để admin xét duyệt. Không thể hủy sau khi đã bắt đầu bán vé.

#### Quản trị viên (Administrator)
- **Đối tượng:** Nhân sự nội bộ của nền tảng với quyền hạn cao nhất trên toàn hệ thống.
- **Nhu cầu cốt lõi:**
  - Duyệt hoặc từ chối yêu cầu tạo concert **trong vòng 48 giờ**.
  - Quản lý tài khoản người dùng (khóa, mở khóa, phân quyền).
  - Chấp thuận hoặc từ chối yêu cầu hủy concert từ ban tổ chức.
  - Hủy concert ở bất kỳ giai đoạn nào trước khi diễn ra và kích hoạt hoàn tiền 100% tự động cho toàn bộ khách hàng.
- **Ràng buộc nghiệp vụ:**
  - Phải xử lý yêu cầu duyệt concert trong tối đa **48 giờ**.
  - Không thể hủy concert sau khi concert đã diễn ra (trạng thái COMPLETED).

#### Nhân viên quét vé (Check-in Staff)
- **Đối tượng:** Nhân viên tại cổng vào concert, sử dụng thiết bị di động hoặc máy quét QR chuyên dụng.
- **Nhu cầu cốt lõi:**
  - Quét mã QR của khách hàng nhanh chóng và nhận phản hồi tức thì: hợp lệ / đã dùng rồi / không hợp lệ.
  - Hệ thống tự động cập nhật trạng thái vé từ `ORDERED` → `CHECKED` sau mỗi lần quét thành công.
  - Ngăn chặn việc dùng lại vé đã check-in (một mã QR chỉ hợp lệ **một lần duy nhất**).
- **Kỳ vọng:** Hệ thống phản hồi quét QR **≤ 500 ms** và phải hoạt động ổn định ngay cả khi kết nối mạng tại địa điểm không ổn định.

#### Nhà cung cấp thanh toán (Payment Provider)
- **Đối tượng:** VNPay, MoMo và các cổng thanh toán bên thứ ba được tích hợp vào nền tảng.
- **Nhu cầu cốt lõi:**
  - Nhận yêu cầu thanh toán / hoàn tiền hợp lệ từ hệ thống.
  - Gửi webhook callback xác nhận trạng thái giao dịch (thành công / thất bại / đang xử lý).
- **Lưu ý kỹ thuật quan trọng:** Hệ thống phải xử lý được: callback đến trễ sau khi ghế đã hết hạn giữ, callback bị gửi trùng lặp, và đảm bảo **idempotency** (không trừ tiền hai lần) ở mọi điểm tích hợp với payment provider.

#### Đội vận hành nền tảng (Platform Ops)
- **Đối tượng:** Kỹ sư hệ thống, DevOps và SRE nội bộ.
- **Nhu cầu cốt lõi:**
  - Giám sát toàn bộ hệ thống qua dashboard (load, latency, error rate, queue depth).
  - Thiết lập alerting khi lưu lượng tăng đột biến (đặc biệt thời điểm mở bán vé với ~100.000 CCU).
  - Đảm bảo SLA, xử lý incident và scale hệ thống kịp thời.
  - Giám sát Redis, Kafka, DB replicas và trạng thái các dịch vụ phụ trợ.

---

## 2. Mục Tiêu Kinh Doanh

| ID | Mục tiêu | Ưu tiên | Chỉ số đo lường (KPI) |
|---|---|---|---|
| BO-01 | Cung cấp trải nghiệm mua vé concert trực tuyến đáng tin cậy trên Web, iOS và Android | Rất cao | Tỷ lệ hoàn thành đặt vé thành công ≥ 95% trong điều kiện tải bình thường |
| BO-02 | Hỗ trợ mở bán vé với tải cao (~100.000 CCU) mà không xảy ra bán trùng ghế | Rất cao | Số sự kiện bán trùng ghế = 0 trong mọi điều kiện tải |
| BO-03 | Quản lý phòng chờ và phân bổ ghế một cách công bằng, minh bạch | Cao | Số khiếu nại về thứ tự hàng chờ ≤ 0,1% tổng phiên mua vé |
| BO-04 | Đảm bảo thanh toán, hoàn tiền và phát hành vé an toàn, có thể truy vết và idempotent | Rất cao | Số lần trừ tiền trùng lặp = 0; 100% giao dịch hoàn tiền có bản ghi audit đầy đủ |
| BO-05 | Trao cho ban tổ chức và quản trị viên công cụ quản lý vòng đời concert đầy đủ | Cao | 100% yêu cầu duyệt concert được xử lý trong vòng 48 giờ |
| BO-06 | Cho phép xác thực vé QR một lần, nhanh chóng tại cổng vào | Cao | Thời gian phản hồi quét QR ≤ 500 ms; tỷ lệ check-in trùng lặp = 0 |
| BO-07 | Gửi thông báo kịp thời và cung cấp báo cáo kinh doanh cho đúng bên liên quan | Trung bình | Tỷ lệ gửi thông báo thành công ≥ 99% trong vòng 30 giây sau sự kiện kích hoạt |

---

## 3. Chỉ Số Thành Công (Success Indicators)

- **[SI-01]** Không có ghế nào bị bán hoặc giữ thành công bởi nhiều hơn một khách hàng trong cùng một thời điểm.
- **[SI-02]** Một yêu cầu thanh toán bị retry (do mạng gián đoạn hoặc khách nhấn nhiều lần) không dẫn đến việc trừ tiền khách hàng lần thứ hai.
- **[SI-03]** Mã QR vé hợp lệ chỉ có thể được check-in đúng một lần; mọi lần quét tiếp theo đều bị từ chối và ghi log.
- **[SI-04]** Khách hàng nhận được thông báo rõ ràng về mọi thay đổi trạng thái quan trọng: thanh toán thành công, hoàn tiền, hủy đơn, hết vé.
- **[SI-05]** Ban tổ chức và quản trị viên có thể hoàn thành tất cả hành động quản lý và phê duyệt bắt buộc mà không gặp lỗi hệ thống.
- **[SI-06]** Hệ thống duy trì tính nhất quán dữ liệu ghế ngồi (seat inventory consistency) ngay cả khi Redis gặp sự cố — fallback sang DB-level locking.
- **[SI-07]** Tất cả giao dịch hoàn tiền từ việc admin hủy concert được xử lý hoàn toàn tự động, không cần can thiệp thủ công.

---

## 4. Tài Liệu Liên Quan

- [`01_Business_Overview/01_Business_Context_And_System_Scope.md`](../01_Business_Overview/01_Business_Context_And_System_Scope.md)
- [`01_Business_Overview/02_Actor_Permissions.md`](../01_Business_Overview/02_Actor_Permissions.md)
- [`01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md`](../01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md)
- [`02_State_Machines_And_Workflows/`](../02_State_Machines_And_Workflows/)
- [`00_Project_Governance/02_Scope_Assumptions_Constraints_And_Dependencies.md`](./02_Scope_Assumptions_Constraints_And_Dependencies.md)
