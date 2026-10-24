# UC02 — Quản Lý Thông Tin Tài Khoản Cá Nhân (Personal Account Management)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC02 |
| **Tên Use Case** | Quản Lý Thông Tin Tài Khoản Cá Nhân |
| **Module** | 01 — Account Module |
| **Actor chính** | Khách hàng (Customer) |
| **Mức độ ưu tiên** | 🟡 High |
| **Precondition** | Người dùng đã đăng nhập (có JWT hợp lệ) |

---

## 2. Tóm Tắt

Use case này cho phép khách hàng xem và cập nhật thông tin hồ sơ cá nhân, đổi mật khẩu, quản lý phương thức thông báo, và xem lịch sử hoạt động tài khoản.

---

## 3. Luồng Chính & User Stories

### US-02-01: Cập Nhật Hồ Sơ Cá Nhân

> **As a** khách hàng,  
> **I want to** cập nhật tên hiển thị, ảnh đại diện và thông tin liên hệ,  
> **So that** vé điện tử và đơn hàng hiển thị đúng tên của tôi.

**Luồng chính:**

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Customer | Vào "Tài khoản của tôi" → "Chỉnh sửa hồ sơ" | Hiển thị form với dữ liệu hiện tại |
| 2 | Customer | Chỉnh sửa: Họ tên, Ngày sinh, Giới tính, Địa chỉ email phụ | Validate real-time |
| 3 | Customer | Upload ảnh đại diện mới | Hệ thống resize và lưu ảnh (JPG/PNG, tối đa 5MB) |
| 4 | Customer | Nhấn "Lưu thay đổi" | Cập nhật DB; hiển thị thông báo thành công |

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-02-01-01 | Họ tên hiển thị phải từ 2–100 ký tự, không chứa ký tự đặc biệt nguy hiểm (XSS) | Functional |
| AC-02-01-02 | Ảnh đại diện chấp nhận JPG, PNG, WebP; tối đa 5MB; tự động crop về 400x400px | Functional |
| AC-02-01-03 | Thay đổi hồ sơ được phản ánh ngay lập tức trên toàn bộ giao diện (vé, đơn hàng) | Functional |
| AC-02-01-04 | Không cho phép thay đổi số điện thoại đã xác thực OTP trực tiếp — phải qua luồng xác thực riêng | Security |

---

### US-02-02: Đổi Mật Khẩu

> **As a** khách hàng,  
> **I want to** đổi mật khẩu tài khoản,  
> **So that** tăng cường bảo mật hoặc khi nghi ngờ bị lộ mật khẩu.

**Luồng chính:**

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Customer | Vào "Bảo mật" → "Đổi mật khẩu" | Hiển thị form 3 ô: Mật khẩu hiện tại, Mật khẩu mới, Xác nhận |
| 2 | Customer | Điền đầy đủ và nhấn "Đổi mật khẩu" | Xác thực mật khẩu hiện tại bằng bcrypt compare |
| 3 | System | Xác thực thành công | Cập nhật hash mật khẩu mới; vô hiệu hóa tất cả Refresh Token cũ; gửi email thông báo |

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-02-02-01 | Phải nhập đúng mật khẩu hiện tại trước khi đổi; không bypass được | Security |
| AC-02-02-02 | Mật khẩu mới không được trùng với mật khẩu hiện tại | Functional |
| AC-02-02-03 | Sau khi đổi mật khẩu, toàn bộ Refresh Token cũ bị vô hiệu hóa (buộc đăng nhập lại trên thiết bị khác) | Security |
| AC-02-02-04 | Gửi email cảnh báo "Mật khẩu vừa được thay đổi" kèm thời gian và IP; nếu không phải bạn thực hiện → link khóa tài khoản khẩn | Security |

---

### US-02-03: Quản Lý Tùy Chọn Thông Báo

> **As a** khách hàng,  
> **I want to** tùy chỉnh loại thông báo tôi muốn nhận,  
> **So that** không bị làm phiền bởi thông báo không cần thiết.

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-02-03-01 | Khách hàng có thể bật/tắt độc lập các loại thông báo: Push, Email, SMS | Functional |
| AC-02-03-02 | Các thông báo quan trọng (xác nhận đơn hàng, hoàn tiền) không thể tắt hoàn toàn (luôn gửi qua ít nhất 1 kênh) | Functional |
| AC-02-03-03 | Tùy chọn thông báo được đồng bộ giữa Web và Mobile App trong vòng < 5 giây | Functional |

---

### US-02-04: Xem Lịch Sử Hoạt Động Tài Khoản

> **As a** khách hàng,  
> **I want to** xem lịch sử đăng nhập và thay đổi tài khoản,  
> **So that** phát hiện kịp thời nếu tài khoản bị truy cập trái phép.

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-02-04-01 | Hiển thị danh sách 10 phiên đăng nhập gần nhất: Thời gian, thiết bị, IP, địa điểm (dựa trên IP geolocation) | Functional |
| AC-02-04-02 | Có nút "Đăng xuất khỏi tất cả thiết bị khác" — vô hiệu hóa toàn bộ Refresh Token ngoài phiên hiện tại | Security |
| AC-02-04-03 | Lịch sử hoạt động được lưu tối thiểu 90 ngày | Functional |

