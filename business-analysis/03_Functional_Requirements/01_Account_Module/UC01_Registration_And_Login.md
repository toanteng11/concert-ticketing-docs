# UC01 — Đăng Ký & Đăng Nhập Tài Khoản (Registration & Login)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC01 |
| **Tên Use Case** | Đăng Ký & Đăng Nhập Tài Khoản |
| **Module** | 01 — Account Module |
| **Actor chính** | Khách vãng lai (Guest), Khách hàng (Customer) |
| **Actor phụ** | Google OAuth2 Provider, Apple ID Provider, SMS Gateway (OTP) |
| **Mức độ ưu tiên** | 🔴 Critical — Bắt buộc có trước mọi tính năng khác |
| **BR liên quan** | BR-QUEUE-004 (1 tài khoản / 1 số thứ tự), BR-PAY-001 |
| **State Machine liên quan** | Không trực tiếp |

---

## 2. Tóm Tắt (Brief Description)

Use case này mô tả cách người dùng mới đăng ký tài khoản trên hệ thống (qua email/SĐT hoặc tài khoản Google/Apple) và cách người dùng hiện tại đăng nhập để truy cập các tính năng yêu cầu xác thực như mua vé, xem lịch sử đơn hàng.

---

## 3. Điều Kiện Tiên Quyết (Preconditions)

- Người dùng chưa có phiên đăng nhập hợp lệ (chưa có JWT Access Token).
- Với luồng đăng ký: Số điện thoại / email chưa được đăng ký trong hệ thống.
- Với đăng nhập Google/Apple: Người dùng có tài khoản Google/Apple còn hoạt động.

---

## 4. Điều Kiện Kết Thúc (Postconditions)

**Thành công:**
- Hệ thống tạo JWT Access Token (TTL 30 phút) và Refresh Token (TTL 30 ngày).
- Token được lưu an toàn phía client (HttpOnly Cookie hoặc Secure Storage trên mobile).
- Người dùng được chuyển hướng đến trang trước đó hoặc trang chủ.

**Thất bại:**
- Không có session nào được tạo.
- Người dùng nhận thông báo lỗi rõ ràng, không lộ thông tin nhạy cảm.

---

## 5. Luồng Chính (Main Flow) — Đăng Ký Qua Email/Số Điện Thoại

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Guest | Nhấn "Đăng ký" trên trang chủ hoặc khi cố truy cập tính năng yêu cầu đăng nhập | Hiển thị form đăng ký |
| 2 | Guest | Nhập: Họ tên, Số điện thoại (hoặc Email), Mật khẩu, Xác nhận mật khẩu | Validate real-time: định dạng SĐT VN (+84 hoặc 0xxx), độ mạnh mật khẩu |
| 3 | Guest | Nhấn "Gửi mã OTP" | Hệ thống gửi mã OTP 6 chữ số qua SMS (TTL 5 phút); hiển thị đồng hồ đếm ngược |
| 4 | Guest | Nhập mã OTP nhận được | Hệ thống xác thực OTP: đúng → tiếp tục; sai → hiện thông báo lỗi (tối đa 5 lần thử) |
| 5 | Guest | Nhấn "Hoàn tất đăng ký" | Hệ thống tạo tài khoản; gửi email chào mừng; cấp JWT; chuyển hướng về trang trước |

---

## 6. Luồng Thay Thế (Alternative Flows)

### AF-01: Đăng Ký / Đăng Nhập Qua Google OAuth2

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Guest | Nhấn "Tiếp tục với Google" | Chuyển hướng đến Google OAuth2 Consent Screen |
| 2 | Guest | Chọn tài khoản Google, cấp quyền | Google trả về Authorization Code |
| 3 | System | Exchange code → Access Token → Lấy profile (name, email, google_id) | — |
| 4 | System | Kiểm tra email trong DB | Nếu đã tồn tại: đăng nhập; Nếu chưa: tạo tài khoản mới với email Google |
| 5 | System | Cấp JWT + Refresh Token | Chuyển hướng về trang đích |

### AF-02: Đăng Nhập Qua Email/SĐT + Mật Khẩu

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Customer | Nhấn "Đăng nhập", nhập email/SĐT và mật khẩu | — |
| 2 | System | Tìm tài khoản, so sánh hash mật khẩu (bcrypt) | Khớp → Cấp token; Không khớp → Lỗi "Sai mật khẩu" |
| 3 | System | Kiểm tra tài khoản có bị khóa không (`account.status = LOCKED`) | Bị khóa → Hiện thông báo lý do và cách mở khóa |

### AF-03: Quên Mật Khẩu

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Guest | Nhấn "Quên mật khẩu", nhập email/SĐT | Hệ thống gửi link reset mật khẩu (TTL 15 phút) qua email, hoặc OTP qua SMS |
| 2 | Guest | Click link / Nhập OTP | Hiển thị form nhập mật khẩu mới |
| 3 | Guest | Nhập mật khẩu mới và xác nhận | Cập nhật hash mật khẩu; Vô hiệu hóa tất cả Refresh Token cũ; Đăng nhập tự động |

---

## 7. Luồng Ngoại Lệ (Exception Flows)

| ID | Điều kiện | Xử lý |
|---|---|---|
| EX-01 | SĐT/Email đã được đăng ký | Hiện thông báo "Số điện thoại đã được sử dụng. Bạn có muốn đăng nhập không?" |
| EX-02 | OTP nhập sai quá 5 lần | Khóa yêu cầu OTP trong 15 phút; hiện đồng hồ đếm ngược |
| EX-03 | OTP hết hạn (> 5 phút) | Hiện nút "Gửi lại OTP" sau khi đồng hồ đếm ngược về 0 |
| EX-04 | Mật khẩu nhập sai quá 10 lần liên tiếp | Tài khoản bị khóa tạm thời 1 giờ; gửi email thông báo |
| EX-05 | Google OAuth thất bại / bị từ chối | Quay về trang đăng nhập với thông báo "Đăng nhập Google thất bại" |
| EX-06 | Tài khoản bị Admin khóa vĩnh viễn | Hiện thông báo "Tài khoản bị vô hiệu hóa. Liên hệ hỗ trợ để biết thêm." |

---

## 8. User Stories & Acceptance Criteria

### US-01-01: Đăng Ký Tài Khoản Mới

> **As a** khách vãng lai,  
> **I want to** đăng ký tài khoản bằng số điện thoại và mật khẩu,  
> **So that** tôi có thể mua vé concert trên nền tảng.

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-01-01-01 | Hệ thống xác thực định dạng SĐT Việt Nam hợp lệ (bắt đầu bằng 0 hoặc +84, 10 chữ số) ngay khi người dùng rời khỏi ô nhập | Functional |
| AC-01-01-02 | Mật khẩu phải có tối thiểu 8 ký tự, bao gồm ít nhất 1 chữ hoa, 1 chữ số. Hiển thị thanh chỉ báo độ mạnh | Functional |
| AC-01-01-03 | Hệ thống gửi OTP SMS trong vòng dưới 30 giây | Performance |
| AC-01-01-04 | Mã OTP có hiệu lực trong đúng 5 phút; hết hạn tự động vô hiệu | Functional |
| AC-01-01-05 | Sau khi đăng ký thành công, người dùng nhận email chào mừng có chứa tên và hướng dẫn sử dụng cơ bản | Functional |
| AC-01-01-06 | Mật khẩu được lưu dưới dạng bcrypt hash (cost factor ≥ 12), không bao giờ lưu plaintext | Security |
| AC-01-01-07 | Cùng 1 SĐT không thể đăng ký 2 tài khoản khác nhau | Functional |
| AC-01-01-08 | Thời gian tạo tài khoản sau khi OTP thành công < 500ms | Performance |

---

### US-01-02: Đăng Nhập Tài Khoản

> **As a** khách hàng đã đăng ký,  
> **I want to** đăng nhập bằng SĐT/email và mật khẩu,  
> **So that** tôi có thể truy cập tài khoản và mua vé.

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-01-02-01 | Đăng nhập thành công cấp JWT Access Token (TTL 30 phút) và Refresh Token (TTL 30 ngày) | Functional |
| AC-01-02-02 | JWT được lưu trong HttpOnly Cookie (web) hoặc SecureStorage (mobile); không lộ trong localStorage | Security |
| AC-01-02-03 | Sau 10 lần đăng nhập sai mật khẩu liên tiếp, tài khoản bị khóa 1 giờ và gửi email cảnh báo | Security |
| AC-01-02-04 | Hệ thống tự động làm mới Access Token khi hết hạn bằng Refresh Token (Silent Refresh) không làm gián đoạn trải nghiệm | Functional |
| AC-01-02-05 | Thời gian phản hồi đăng nhập < 1 giây ở điều kiện bình thường | Performance |
| AC-01-02-06 | Đăng nhập Google hiển thị thông tin tài khoản Google đã được liên kết, không yêu cầu nhập mật khẩu lần 2 | Functional |

---

### US-01-03: Đăng Xuất

> **As a** khách hàng đang đăng nhập,  
> **I want to** đăng xuất khỏi tài khoản,  
> **So that** bảo vệ tài khoản trên thiết bị dùng chung.

**Acceptance Criteria:**

| AC | Mô tả | Loại |
|---|---|---|
| AC-01-03-01 | Đăng xuất xóa JWT và Refresh Token phía client; vô hiệu hóa Refresh Token trên server (blacklist) | Security |
| AC-01-03-02 | Sau khi đăng xuất, truy cập lại trang yêu cầu auth → chuyển hướng về trang đăng nhập | Functional |
| AC-01-03-03 | Đăng xuất trên 1 thiết bị không ảnh hưởng phiên trên thiết bị khác (theo design Refresh Token per device) | Functional |

---

## 9. Danh Sách Use Case Con & Quan Hệ Actor (Dùng Làm Input Vẽ Use Case Diagram)

| Use Case Con | Actor Chính | Actor Phụ (External System) | Ghi Chú |
|---|---|---|---|
| UC01.1 — Đăng ký tài khoản mới | Guest | SMS Gateway (gửi OTP) | Bao gồm validate SĐT và xác thực OTP |
| UC01.2 — Đăng nhập Email/SĐT + Mật khẩu | Guest | — | Include: kiểm tra trạng thái tài khoản |
| UC01.3 — Đăng nhập qua Google OAuth2 | Guest | Google OAuth2 Provider | Extend UC01.2 |
| UC01.4 — Đăng nhập qua Apple ID | Guest | Apple ID Provider | Extend UC01.2 |
| UC01.5 — Quên mật khẩu / Đặt lại mật khẩu | Guest | SMS Gateway / Email Service | Include UC01.1 (OTP verify) |
| UC01.6 — Đăng xuất | Customer | — | Vô hiệu hóa Refresh Token |

---

## 10. Ghi Chú Kỹ Thuật Bổ Sung

| Hạng mục | Chi tiết |
|---|---|
| **Rate Limiting** | Tối đa 5 request OTP/SĐT/giờ; 100 request đăng nhập/IP/phút |
| **Audit Log** | Ghi nhận mọi sự kiện: đăng nhập thành công/thất bại, đổi mật khẩu, khóa tài khoản với IP + timestamp |
| **CAPTCHA** | reCAPTCHA v3 invisible khi phát hiện hành vi bất thường; CAPTCHA v2 visible sau 5 lần thất bại |
| **Xác thực 2 yếu tố (2FA)** | Tùy chọn TOTP (Google Authenticator) cho tài khoản Admin và Organizer (bắt buộc), Customer (tùy chọn) |

