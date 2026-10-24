# UC03 — Quản Lý Tài Khoản Người Dùng (Admin Account Management)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC03 |
| **Tên Use Case** | Quản Lý Tài Khoản Người Dùng (Admin) |
| **Module** | 01 — Account Module |
| **Actor chính** | Admin |
| **Mức độ ưu tiên** | 🟡 High |
| **Precondition** | Admin đã đăng nhập với quyền `ROLE_ADMIN` và xác thực 2FA |
| **BR liên quan** | BR-CON-007 (giới hạn Organizer mới) |

---

## 2. Tóm Tắt

Use case này cho phép Admin quản lý toàn bộ tài khoản người dùng trong hệ thống: xem, tìm kiếm, khóa/mở khóa, phân quyền và điều tra hành vi gian lận.

---

## 3. User Stories & Acceptance Criteria

### US-03-01: Tìm Kiếm & Xem Thông Tin Tài Khoản

> **As an** Admin,  
> **I want to** tìm kiếm và xem thông tin chi tiết bất kỳ tài khoản người dùng,  
> **So that** hỗ trợ khách hàng và điều tra sự cố.

**Luồng chính:**

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Admin | Vào "Quản lý người dùng" trên Admin Dashboard | Hiển thị danh sách tài khoản (phân trang 50/trang) |
| 2 | Admin | Tìm theo: SĐT, email, tên, user_id | Hiển thị kết quả lọc real-time |
| 3 | Admin | Click vào tài khoản cụ thể | Hiển thị: Thông tin cá nhân, Trạng thái, Lịch sử đơn hàng, Lịch sử đăng nhập, Cờ bảo mật |

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-03-01-01 | Tìm kiếm trả về kết quả trong < 500ms cho database < 2 triệu user | Performance |
| AC-03-01-02 | Admin không được xem mật khẩu (không bao giờ lộ hash) hoặc số thẻ ngân hàng đầy đủ | Security |
| AC-03-01-03 | Mọi thao tác xem thông tin người dùng được ghi vào Audit Log với admin_id + timestamp | Security |

---

### US-03-02: Khóa & Mở Khóa Tài Khoản

> **As an** Admin,  
> **I want to** khóa tài khoản có hành vi gian lận (dùng bot, đầu cơ vé),  
> **So that** bảo vệ sự công bằng của hệ thống xếp hàng.

**Luồng chính:**

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Admin | Mở tài khoản → Click "Khóa tài khoản" | Hiển thị dialog xác nhận yêu cầu nhập lý do |
| 2 | Admin | Chọn loại khóa: Tạm thời (X giờ) hoặc Vĩnh viễn; nhập lý do | — |
| 3 | Admin | Xác nhận | Tài khoản bị khóa; vô hiệu hóa tất cả JWT/Refresh Token; gửi email thông báo đến user |

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-03-02-01 | Tài khoản bị khóa không thể đăng nhập; mọi JWT hiện tại bị vô hiệu hóa ngay lập tức | Security |
| AC-03-02-02 | Phải ghi rõ lý do khóa, Admin thực hiện và thời gian; không thể khóa mà không có lý do | Functional |
| AC-03-02-03 | Gửi email thông báo cho người dùng với lý do khóa và hướng dẫn khiếu nại | Functional |
| AC-03-02-04 | Admin không thể tự khóa tài khoản của chính mình | Security |

---

### US-03-03: Phân Quyền Tài Khoản Organizer

> **As an** Admin,  
> **I want to** nâng cấp tài khoản Customer lên vai trò Organizer sau khi xác minh pháp lý,  
> **So that** Ban tổ chức có thể bắt đầu tạo concert trên nền tảng.

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-03-03-01 | Chỉ Admin mới có quyền gán role `ROLE_ORGANIZER` — không có API endpoint công khai nào cho phép tự nâng quyền | Security |
| AC-03-03-02 | Hệ thống lưu lịch sử phân quyền: ai cấp, khi nào, tài khoản nào được cấp | Functional |
| AC-03-03-03 | Organizer mới (chưa xác minh doanh nghiệp đầy đủ) bị giới hạn tối đa 2 concert đang hoạt động (BR-CON-007) | Functional |
| AC-03-03-04 | Admin có thể thu hồi quyền Organizer bất kỳ lúc nào; Organizer bị thu hồi quyền không còn truy cập được Organizer Portal | Functional |
