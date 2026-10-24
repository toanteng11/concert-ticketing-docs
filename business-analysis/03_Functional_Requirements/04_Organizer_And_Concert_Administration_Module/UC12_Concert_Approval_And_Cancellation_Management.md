# UC12 — Phê Duyệt & Quản Lý Concert (Admin)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC12 |
| **Tên Use Case** | Phê Duyệt & Quản Lý Concert (Admin) |
| **Module** | 04 — Organizer & Concert Administration Module |
| **Actor chính** | Admin |
| **Mức độ ưu tiên** | 🔴 Critical |
| **Precondition** | Admin đã đăng nhập với `ROLE_ADMIN` và xác thực 2FA; có concert đang ở trạng thái `PENDING_REVIEW` |
| **BR liên quan** | BR-CON-001, BR-CON-003, BR-CON-004, BR-CON-005, BR-CON-006 |

---

## 2. Tóm Tắt

Use case này mô tả quy trình Admin xem xét hồ sơ concert do Organizer gửi lên, đưa ra quyết định phê duyệt, yêu cầu chỉnh sửa, hoặc từ chối. Ngoài ra bao gồm quyền hủy concert khẩn cấp và quản lý vận hành hệ thống bán vé.

---

## 3. Luồng Chính A — Phê Duyệt Concert

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Admin | Nhận thông báo concert mới cần duyệt; vào Admin Dashboard → "Hàng chờ phê duyệt" | Danh sách concert PENDING_REVIEW, sắp xếp theo thời gian chờ lâu nhất |
| 2 | Admin | Click vào concert cần xem xét | Hiển thị đầy đủ: Thông tin concert, Sơ đồ ghế, Giá vé, Tài liệu pháp lý (GPKD, hợp đồng) |
| 3 | Admin | Xem xét hồ sơ: Kiểm tra tính hợp lệ, đối chiếu GPKD | Giao diện xem tài liệu inline, không cần tải về |
| 4 | Admin | Click "Phê duyệt" | Popup xác nhận: "Phê duyệt concert này? Hành động không thể hoàn tác ngay lập tức." |
| 5 | Admin | Nhập ghi chú phê duyệt (tùy chọn) → Xác nhận | `concert.status = PUBLISHED`; Organizer nhận thông báo |

---

## 4. Luồng Thay Thế (Alternative Flows)

### AF-01: Yêu Cầu Chỉnh Sửa (Request Revision)

| Bước | Mô tả |
|---|---|
| 1 | Admin click "Yêu cầu chỉnh sửa" |
| 2 | Form nhập lý do bắt buộc: "Thiếu GPKD", "Sơ đồ ghế chưa hoàn chỉnh", v.v. |
| 3 | `concert.status = DRAFT`; Organizer nhận email chi tiết lý do và hướng dẫn cách sửa |
| 4 | Organizer chỉnh sửa và gửi lại — concert quay lại PENDING_REVIEW |

### AF-02: Từ Chối Hồ Sơ (Reject)

| Bước | Mô tả |
|---|---|
| 1 | Admin click "Từ chối" |
| 2 | Bắt buộc nhập lý do từ chối chi tiết |
| 3 | `concert.status = CANCELLED (Rejected)`; Organizer nhận email thông báo từ chối |
| 4 | Organizer không thể gửi lại cùng hồ sơ; cần tạo concert mới |

### AF-03: Hủy Concert Khẩn Cấp (Emergency Cancellation)

| Bước | Mô tả |
|---|---|
| 1 | Admin tìm concert đang ON_SALE → Click "Hủy khẩn cấp" |
| 2 | Màn cảnh báo: Hiển thị số đơn PAID bị ảnh hưởng, tổng tiền cần hoàn; yêu cầu nhập lý do + xác nhận 2FA |
| 3 | Admin nhập lý do + xác nhận | `concert.status = CANCELLED`; toàn bộ đơn PAID → REFUND_PENDING; gửi thông báo hàng loạt đến tất cả khách hàng |
| 4 | Hệ thống tự động xử lý hoàn tiền hàng loạt (Batch Refund Saga) |

---

## 5. Luồng Ngoại Lệ (Exception Flows)

| ID | Điều kiện | Xử lý |
|---|---|---|
| EX-01 | Admin không xử lý hồ sơ trong 48h (SLA breach) | Hệ thống tự động gửi alert cảnh báo SLA vi phạm; escalate lên Supervisor Admin |
| EX-02 | Cố hủy concert sau khi đã diễn ra (COMPLETED) | `422: CONCERT_ALREADY_COMPLETED`. Không cho phép |
| EX-03 | Admin tự phê duyệt concert của chính Organizer do mình quản lý | Conflict of interest check; cần một Admin khác phê duyệt |

---

## 6. User Stories & Acceptance Criteria

### US-12-01: Phê Duyệt Concert

> **As an** Admin,  
> **I want to** xem xét và phê duyệt hồ sơ concert của Organizer,  
> **So that** chỉ các concert hợp lệ và có giấy tờ đầy đủ được hiển thị trên nền tảng.

| AC | Mô tả | Loại |
|---|---|---|
| AC-12-01-01 | Admin Dashboard hiển thị danh sách PENDING_REVIEW sắp xếp theo thời gian chờ lâu nhất; có badge cảnh báo SLA khi > 24h | Functional |
| AC-12-01-02 | Admin có thể xem inline tài liệu pháp lý PDF, sơ đồ ghế, và preview trang concert như người dùng thấy | Functional |
| AC-12-01-03 | Sau khi Admin phê duyệt: `concert.status = PUBLISHED`; Organizer nhận email với link concert trong < 5 giây | Performance |
| AC-12-01-04 | Mọi quyết định (phê duyệt/từ chối/yêu cầu sửa) được ghi vào Audit Log không thể chỉnh sửa | Security |
| AC-12-01-05 | Sau khi phê duyệt, hệ thống tự động lên lịch kích hoạt Waiting Room đúng T-15 phút | Functional |

---

### US-12-02: Hủy Concert Khẩn Cấp

> **As an** Admin,  
> **I want to** hủy concert khẩn cấp và kích hoạt hoàn tiền hàng loạt,  
> **So that** bảo vệ khách hàng khi có sự cố bất khả kháng.

| AC | Mô tả | Loại |
|---|---|---|
| AC-12-02-01 | Trước khi xác nhận hủy: Hiển thị số đơn bị ảnh hưởng và tổng tiền cần hoàn để Admin có đầy đủ thông tin | Functional |
| AC-12-02-02 | Yêu cầu Admin xác thực 2FA trước khi hủy concert có đơn PAID | Security |
| AC-12-02-03 | Sau khi hủy: Thông báo được gửi đến 100% khách hàng bị ảnh hưởng trong vòng 5 phút qua cả Email và Push | Performance |
| AC-12-02-04 | Hoàn tiền hàng loạt được xử lý tự động không cần Admin làm thủ công từng đơn | Functional |

---

## 7. Danh Sách Use Case Con & Quan Hệ Actor

| Use Case Con | Actor Chính | Actor Phụ | Ghi Chú |
|---|---|---|---|
| UC12.1 — Xem danh sách concert chờ duyệt | Admin | — | SLA tracking 48h |
| UC12.2 — Phê duyệt concert | Admin | Notification Service | Trigger → PUBLISHED |
| UC12.3 — Yêu cầu Organizer chỉnh sửa | Admin | Notification Service | Concert → DRAFT |
| UC12.4 — Từ chối hồ sơ | Admin | Notification Service | Concert → CANCELLED |
| UC12.5 — Hủy concert khẩn cấp | Admin | Refund Service, Notification Service | Yêu cầu 2FA; Batch Refund |
| UC12.6 — Chỉnh sửa thông tin concert (Admin override) | Admin | — | Trường hợp đặc biệt |
| UC12.7 — Đóng/Mở hàng chờ thủ công | Admin | Queue Service | Override tự động |

