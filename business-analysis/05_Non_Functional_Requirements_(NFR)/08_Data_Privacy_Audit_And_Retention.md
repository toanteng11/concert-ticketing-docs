# 08 — Quyền Riêng Tư Dữ Liệu, Nhật Ký Kiểm Toán & Lưu Trữ (Data Privacy, Audit & Retention)

## 1. Giới Thiệu & Khung Pháp Lý Áp Dụng

Hệ thống bán vé sự kiện âm nhạc trực tuyến thu thập và xử lý khối lượng lớn thông tin định danh cá nhân, dữ liệu giao dịch tài chính và lịch sử di chuyển của khán giả. Do đó, nền tảng bắt buộc phải tuân thủ nghiêm ngặt các quy định pháp luật hiện hành:
1. **Nghị định số 13/2023/NĐ-CP của Chính phủ Việt Nam** về Bảo vệ dữ liệu cá nhân (PDPD).
2. **Luật An ninh mạng số 24/2018/QH14** và **Luật Giao dịch điện tử số 20/2023/QH15**.
3. **Luật Kế toán Việt Nam** về thời hạn lưu trữ chứng từ kế toán, giao dịch điện tử.
4. Định hướng tương thích với các tiêu chuẩn quốc tế như **GDPR (Châu Âu)** để sẵn sàng phục vụ khách du lịch quốc tế mua vé concert tại Việt Nam.

Tài liệu này quy định chi tiết việc phân loại dữ liệu, cơ chế đồng ý, ẩn danh hóa thông tin, thiết kế hệ thống nhật ký kiểm toán không thể chỉnh sửa (Audit Trail) và vòng đời lưu trữ/tiêu hủy dữ liệu.

---

## 2. Phân Loại Dữ Liệu Cá Nhân (Data Classification Matrix)

Hệ thống phân định dữ liệu thành 4 cấp độ bảo mật nhằm áp dụng các biện pháp kiểm soát kỹ thuật tương ứng:

| Cấp độ | Nhóm dữ liệu | Ví dụ cụ thể | Yêu cầu mã hóa & Bảo vệ | Quyền truy cập |
|---|---|---|---|---|
| **Cấp 1: Công khai (Public)** | Thông tin sự kiện | Tên concert, nghệ sĩ, thời gian biểu diễn, địa điểm, giá vé niêm yết | Không mã hóa; Cache tại CDN | Toàn bộ Internet |
| **Cấp 2: Nội bộ (Internal)** | Thống kê vận hành | Số vé đã bán, tỷ lệ lấp đầy rạp, cấu hình hệ thống, số lượng cổng soát vé | Mã hóa khi truyền tải (TLS 1.3) | Nhân viên & Hệ thống nội bộ |
| **Cấp 3: Dữ liệu cá nhân cơ bản (Confidential)** | Thông tin khách hàng | Họ và tên, số điện thoại, địa chỉ email, ngày sinh, lịch sử đơn hàng | Mã hóa đường truyền & Lưu trữ (AES-256); Mặt nạ hóa khi hiển thị | Chính chủ sở hữu, CSKH (có giới hạn) |
| **Cấp 4: Dữ liệu cá nhân nhạy cảm & Tài chính (Restricted)** | Định danh & Giao dịch | Số Căn cước công dân (CCCD), Mật khẩu (Hash), Token thanh toán, Tọa độ GPS soát vé | Mã hóa mức cột (Column-level Encryption); Băm bcrypt cost 12 | Nghiêm cấm truy cập thô; Chỉ dùng qua hàm bảo mật |

---

## 3. Quyền Của Chủ Thể Dữ Liệu & Cơ Chế Thực Thi (Data Subject Rights)

Tuân thủ Nghị định 13/2023/NĐ-CP, hệ thống cung cấp đầy đủ các cổng giao tiếp để người dùng thực thi các quyền hợp pháp của mình:

### 1. Quyền Đồng Ý & Rút Lại Sự Đồng Ý (Consent Management)
- Khi đăng ký tài khoản hoặc mua vé, người dùng bắt buộc phải tích chọn hộp kiểm đồng ý (Opt-in Explicit Consent): *"Tôi đã đọc và đồng ý với Điều khoản Dịch vụ và Chính sách Bảo vệ Dữ liệu Cá nhân"*.
- Tuyệt đối không sử dụng ô tích chọn mặc định sẵn (No Pre-ticked Boxes).
- Người dùng có quyền vào trang cá nhân để bật/tắt sự đồng ý đối với việc sử dụng dữ liệu cho mục đích tiếp thị và phân tích thói quen nghe nhạc bất kỳ lúc nào.

### 2. Quyền Yêu Cầu Trích Xuất Dữ Liệu (Right to Access - DSAR)
- Khách hàng có quyền gửi yêu cầu trích xuất dữ liệu cá nhân qua nút "Tải về dữ liệu của tôi" trên ứng dụng.
- Hệ thống tự động đóng gói toàn bộ thông tin hồ sơ, lịch sử đơn hàng, danh sách vé và bài đánh giá thành một tệp nén `.zip` chứa file `profile.json` có mã hóa mật khẩu, gửi về email đăng ký trong vòng không quá **72 giờ**.

### 3. Quyền Xóa Dữ Liệu Cá Nhân ("Quyền Được Quên" vs Nghĩa Vụ Pháp Lý Lưu Trữ Thuế)
Đây là điểm xung đột pháp lý điển hình: Người dùng yêu cầu xóa tài khoản, nhưng Luật Thuế và Luật Kế toán yêu cầu phải lưu giữ chứng từ giao dịch thanh toán trong tối thiểu 5 năm.
- **Hệ thống áp dụng giải pháp Ẩn Danh Hóa Mềm (Data Anonymization / Pseudonymization):**
  1. Tài khoản bị vô hiệu hóa đăng nhập vĩnh viễn (`users.status = DELETED`).
  2. Toàn bộ thông tin định danh cá nhân trực tiếp được tẩy xóa hoặc ghi đè bằng chuỗi băm ngẫu nhiên:
     ```sql
     UPDATE users SET 
         full_name = 'Anonymous User',
         phone_number = 'DELETED_' || uuid_generate_v4(),
         email = 'deleted_' || id || '@anonymized.local',
         avatar_url = NULL,
         citizen_id = NULL
     WHERE id = :user_id;
     ```
  3. Bản ghi đơn hàng (`orders`) và thanh toán (`payment_transactions`) vẫn được giữ nguyên vẹn về mặt số tiền, ngày giờ và mã tham chiếu ngân hàng để phục vụ kiểm toán tài chính, nhưng không còn bất kỳ mối liên hệ nào với danh tính con người thực tế ngoài đời.

---

## 4. Mặt Nạ Hóa Dữ Liệu Khi Hiển Thị & Ghi Log (Data Masking Standards)

Nhằm ngăn chặn việc rò rỉ dữ liệu qua màn hình nhân viên hỗ trợ hoặc qua tệp nhật ký ứng dụng (Log Leaks):

| Trường thông tin | Dữ liệu gốc thực tế | Định dạng hiển thị mặt nạ (Masked Output) | Quy tắc áp dụng |
|---|---|---|---|
| **Số điện thoại** | `0912345678` | `091****678` | Giữ 3 số đầu và 3 số cuối; che 4 số giữa bằng `*` |
| **Địa chỉ Email** | `nguyenvana@gmail.com` | `n*****a@gmail.com` | Giữ ký tự đầu và cuối của tên hộp thư |
| **Số Căn cước công dân** | `001201012345` | `********2345` | Che toàn bộ, chỉ hiển thị 4 số cuối |
| **Tên chủ thẻ ngân hàng** | `NGUYEN VAN A` | `NGU*** *** A` | Che một phần họ và tên đệm |
| **Mật khẩu & OTP** | `MyPassword@2026` | `[REDACTED]` | **TUYỆT ĐỐI KHÔNG GHI LOG DƯỚI MỌI HÌNH THỨC** |
| **Mã Token JWT** | `eyJhbGciOi...` | `[TOKEN_MASKED]` | Cắt ngắn hoặc lọc bỏ hoàn toàn khỏi log |

---

## 5. Kiến Trúc Nhật Ký Kiểm Toán Bất Biến (Immutable Audit Trail Policy)

Mọi thao tác thay đổi dữ liệu hoặc truy cập thông tin nhạy cảm của các vai trò Quản trị viên (Admin), Ban tổ chức (Organizer) và Nhân viên soát vé (Staff) đều bắt buộc phải được ghi nhận vào bảng **Nhật Ký Kiểm Toán (Audit Trail)**.

### 1. Nguyên Tắc Bất Biến (Write-Once-Read-Many - WORM)
- Bảng `audit_logs` chỉ hỗ trợ duy nhất quyền `INSERT`. Nghiêm cấm cấp quyền `UPDATE` hoặc `DELETE` cho bất kỳ người dùng, ứng dụng hay Admin nào.
- Định kỳ hằng đêm, dữ liệu Audit Log được xuất ra file và ký số điện tử (Digital Signature HMAC-SHA256), sau đó đẩy lên kho lưu trữ **AWS S3 Object Lock (Compliance Mode)** để đảm bảo không ai có thể can thiệp hay chỉnh sửa lịch sử kiểm toán.

### 2. Cấu Trúc Bản Ghi Nhật Ký Kiểm Toán (Audit Log Schema)

```json
{
  "audit_id": "aud_7a9f-3b2c-4e1d",
  "actor": {
    "user_id": "adm_888",
    "role": "ROLE_ADMIN",
    "ip_address": "113.161.45.12",
    "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."
  },
  "action": {
    "action_type": "CONCERT_EMERGENCY_CANCEL",
    "resource_type": "CONCERT",
    "resource_id": "con_blackpink_2026",
    "status": "SUCCESS"
  },
  "changes": {
    "payload_before": { "status": "ON_SALE", "tickets_sold": 45000 },
    "payload_after": { "status": "CANCELLED", "reason": "Bão khẩn cấp đổ bộ" }
  },
  "metadata": {
    "auth_method": "2FA_TOTP_VERIFIED",
    "reason": "Chỉ thị an toàn từ UBND Thành phố",
    "timestamp": "2026-10-24T14:30:00.125Z"
  }
}
```

---

## 6. Vòng Đời Lưu Trữ & Kế Hoạch Tiêu Hủy Dữ Liệu (Data Retention & Purging Schedule)

| Nhóm dữ liệu | Thời gian lưu trữ nóng (Hot Storage) | Thời gian lưu trữ ấm/lạnh (Warm/Cold Archive) | Thời điểm tiêu hủy vĩnh viễn (Permanent Purge) | Căn cứ pháp lý |
|---|---|---|---|---|
| **Dữ liệu Đơn hàng, Hóa đơn & Đối soát** | 1 năm trong MySQL 8.0 | 4 năm tiếp theo trong AWS S3 Glacier | Sau đúng **5 năm** kể từ ngày quyết toán thuế | Luật Kế toán & Luật Thuế VN |
| **Nhật ký Quét Vé Check-In tại Cổng** | 30 ngày trong MySQL 8.0 | 11 tháng tiếp theo trong S3 | Sau **12 tháng** kể từ khi concert kết thúc | Phục vụ đối soát & giải quyết khiếu nại |
| **Nhật ký Kiểm toán (Audit Logs)** | 1 năm trong MySQL 8.0 | 4 năm trong S3 Object Lock | Sau **5 năm** | Tiêu chuẩn kiểm toán bảo mật |
| **Nhật ký Truy cập Mạng (Access Logs / Nginx)** | 30 ngày trong OpenSearch | 60 ngày tiếp theo dạng nén S3 | Sau **90 ngày** | Nghị định quản lý Internet & An ninh mạng |
| **Phiên làm việc & Token Hàng chờ (Redis)** | 10 phút đến 24 giờ | Không lưu trữ | Tự động hết hạn qua cơ chế Redis TTL | Tối ưu tài nguyên In-Memory |

