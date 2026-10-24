# UC11 — Tạo & Quản Lý Concert (Organizer)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC11 |
| **Tên Use Case** | Tạo & Quản Lý Concert (Organizer) |
| **Module** | 04 — Organizer & Concert Administration Module |
| **Actor chính** | Ban tổ chức (Organizer) |
| **Mức độ ưu tiên** | 🔴 Critical |
| **Precondition** | Organizer đã đăng nhập với `ROLE_ORGANIZER`; tài khoản đã xác minh pháp lý KYC doanh nghiệp |
| **BR liên quan** | BR-CON-001 đến BR-CON-008; BR-SEAT-006 |

---

## 2. Tóm Tắt

Use case này mô tả toàn bộ quy trình Organizer tạo một concert mới: từ nhập thông tin sự kiện, thiết kế sơ đồ ghế, định giá vé, đến gửi hồ sơ cho Admin phê duyệt. Sau khi concert được phê duyệt, Organizer có thể theo dõi tiến độ bán vé và quản lý vận hành.

---

## 3. Giai Đoạn 1: Tạo Thông Tin Concert

### 3.1 Luồng Chính — Nhập Thông Tin Concert

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Organizer | Vào Organizer Portal → Click "Tạo concert mới" | Hiển thị form đa bước (Multi-step wizard) |
| 2 | Organizer | Bước 1 — Thông tin cơ bản: Tên concert, Mô tả, Thể loại, Danh sách nghệ sĩ | Validate: Tên ≥ 3 ký tự; Mô tả ≥ 50 ký tự |
| 3 | Organizer | Bước 2 — Địa điểm & thời gian: Tên địa điểm, Địa chỉ, Thành phố, Ngày & giờ diễn, Ngày & giờ mở cửa, Ngày & giờ bắt đầu mở bán | Validate: Ngày diễn > hôm nay + 7 ngày; Giờ mở bán < Giờ diễn |
| 4 | Organizer | Bước 3 — Hình ảnh: Upload banner concert, ảnh nghệ sĩ | Hệ thống tự resize, convert WebP, lưu CDN |
| 5 | Organizer | Bước 4 — Hồ sơ pháp lý: Upload GPKD, Hợp đồng biểu diễn | File PDF/JPG, max 10MB mỗi file |
| 6 | Organizer | Nhấn "Lưu bản nháp" | `concert.status = DRAFT`; có thể chỉnh sửa bất kỳ lúc nào |

---

## 4. Giai Đoạn 2: Thiết Lập Sơ Đồ Ghế

### 4.1 Luồng Chính — Thiết Kế Sơ Đồ Khán Phòng

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Organizer | Vào tab "Sơ đồ ghế" → Chọn template khán phòng | Danh sách template: Nhà hát (numbered), Sân vận động, Ngoài trời (standing), Tùy chỉnh |
| 2 | Organizer | Chỉnh sửa sơ đồ: Thêm/bớt hàng, đổi tên hàng/ghế | Canvas editor thời gian thực |
| 3 | Organizer | Tạo và đặt tên phân khu (Zone): VIP_A, Standard_B, Standing | Gán màu sắc phân biệt cho từng Zone |
| 4 | Organizer | Nhập giá vé cho từng Zone | Validate: Giá > 0 VNĐ; bắt buộc nhập giá cho 100% Zone trước khi gửi duyệt |
| 5 | Organizer | Khóa ghế kỹ thuật (BLOCKED): Chọn ghế → Click "Khóa" | Ghế chuyển màu xám đậm, không hiển thị cho khách hàng |
| 6 | Organizer | Nhấn "Xác nhận sơ đồ" | Hệ thống tạo tất cả bản ghi `seats` trong DB với status `AVAILABLE` |

### 4.2 Thông Tin Phân Khu Cần Nhập

| Trường | Mô tả | Bắt buộc |
|---|---|---|
| Tên Zone | VIP_A, Standard_B, Standing_Floor... | ✅ |
| Loại Zone | Có ghế số (numbered) / Đứng tự do (standing) | ✅ |
| Số lượng ghế | Tự động tính từ sơ đồ | Auto |
| Giá vé (VNĐ) | Giá niêm yết cho Zone này | ✅ |
| Hạn mua tối đa/đơn | Mặc định 4; Organizer có thể đặt 1-6 | Tùy chỉnh |

---

## 5. Giai Đoạn 3: Gửi Hồ Sơ Phê Duyệt

### 5.1 Luồng Chính

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Organizer | Xem preview concert hoàn chỉnh → Click "Gửi phê duyệt" | Hệ thống validate checklist đầy đủ |
| 2 | System | Kiểm tra checklist | Tất cả Zone đã có giá; Thông tin pháp lý đã upload; Thời gian hợp lệ |
| 3 | Organizer | Xác nhận gửi | `concert.status = PENDING_REVIEW`; Admin nhận thông báo |
| 4 | System | Phản hồi | "Hồ sơ đã được gửi. Admin sẽ phản hồi trong 48 giờ." |

---

## 6. Giai Đoạn 4: Quản Lý Sau Khi Concert Được Duyệt & Mở Bán

| Tính năng | Mô tả | Điều kiện |
|---|---|---|
| **Chỉnh sửa thông tin phụ** | Cập nhật mô tả, hình ảnh, danh sách nghệ sĩ khách mời | Bất kỳ trạng thái nào trước COMPLETED |
| **Xem Dashboard bán vé** | Tỷ lệ lấp đầy, doanh thu tạm tính, số vé bán theo Zone | Concert đang ON_SALE |
| **Quản lý nhân viên soát vé** | Thêm/sửa/xóa tài khoản Staff; phân công cổng check-in | Sau khi PUBLISHED |
| **Đóng bán sớm** | Chuyển concert từ ON_SALE sang CLOSED | Khi concert ON_SALE |
| **Yêu cầu hủy concert** | Gửi yêu cầu hủy cần Admin xác nhận | Trước giờ diễn ≥ 72h |
| **Xem báo cáo check-in** | Số vé đã quét, tiến độ theo cổng và thời gian | Trong ngày diễn |
| **Yêu cầu quyết toán** | Gửi yêu cầu rút doanh thu sau khi concert COMPLETED | Sau khi COMPLETED |

---

## 7. Luồng Ngoại Lệ (Exception Flows)

| ID | Điều kiện | Xử lý |
|---|---|---|
| EX-01 | Organizer mới chưa xác minh KYC đầy đủ | Cho phép tạo DRAFT; không cho gửi phê duyệt; hiển thị "Hoàn tất xác minh doanh nghiệp để gửi concert" |
| EX-02 | Organizer mới đang có ≥ 2 concert ACTIVE | `403: ORGANIZER_QUOTA_EXCEEDED`. "Tài khoản mới giới hạn 2 concert đang hoạt động đồng thời" (BR-CON-007) |
| EX-03 | Ngày mở bán <= hôm nay | Validate fail: "Ngày mở bán phải cách ít nhất 24 giờ từ hiện tại" |
| EX-04 | Có Zone chưa được định giá khi gửi duyệt | "Vui lòng nhập giá vé cho tất cả khu vực: [Danh sách Zone chưa có giá]" |
| EX-05 | Cố sửa sơ đồ ghế sau khi concert đã ON_SALE | `403: VENUE_MAP_FROZEN`. "Không thể chỉnh sửa sơ đồ ghế khi concert đang mở bán" (BR-CON-002) |

---

## 8. User Stories & Acceptance Criteria

### US-11-01: Tạo Concert Mới

> **As an** Organizer,  
> **I want to** tạo hồ sơ concert với đầy đủ thông tin và sơ đồ ghế,  
> **So that** gửi lên nền tảng để mở bán vé cho khán giả.

| AC | Mô tả | Loại |
|---|---|---|
| AC-11-01-01 | Form tạo concert theo dạng multi-step wizard với progress indicator; có thể quay lại bước trước mà không mất dữ liệu | UX |
| AC-11-01-02 | Mọi thay đổi được auto-save dạng DRAFT mỗi 30 giây để tránh mất dữ liệu khi mất mạng | Functional |
| AC-11-01-03 | Công cụ tạo sơ đồ ghế hỗ trợ: Chọn template sẵn có hoặc tùy chỉnh hoàn toàn; phóng to/thu nhỏ; undo/redo | Functional |
| AC-11-01-04 | Hệ thống tự động tính và hiển thị tổng số ghế sau khi thiết lập sơ đồ | Functional |
| AC-11-01-05 | Không thể gửi phê duyệt nếu chưa nhập giá vé cho 100% Zone | Functional |

---

### US-11-02: Theo Dõi Dashboard Bán Vé Real-time

> **As an** Organizer,  
> **I want to** xem dashboard bán vé cập nhật theo thời gian thực,  
> **So that** theo dõi tiến độ và ra quyết định vận hành kịp thời.

| AC | Mô tả | Loại |
|---|---|---|
| AC-11-02-01 | Dashboard hiển thị: % ghế đã bán, doanh thu tạm tính, tốc độ bán (vé/phút), phân tích theo Zone | Functional |
| AC-11-02-02 | Dữ liệu dashboard tự cập nhật mỗi 30 giây mà không cần F5 | Functional |
| AC-11-02-03 | Organizer chỉ thấy dữ liệu concert của mình; không thể xem concert của Organizer khác | Security |

---

## 9. Danh Sách Use Case Con & Quan Hệ Actor

| Use Case Con | Actor Chính | Actor Phụ | Ghi Chú |
|---|---|---|---|
| UC11.1 — Nhập thông tin concert cơ bản | Organizer | — | Multi-step form; auto-save DRAFT |
| UC11.2 — Thiết kế sơ đồ ghế & định giá | Organizer | — | Canvas editor; Zone management |
| UC11.3 — Khóa ghế kỹ thuật (BLOCKED) | Organizer | — | Chỉ trước khi ON_SALE |
| UC11.4 — Gửi hồ sơ phê duyệt | Organizer | Notification Service | Trigger Admin review |
| UC11.5 — Quản lý tài khoản Staff | Organizer | — | Tạo/phân công Gate Staff |
| UC11.6 — Xem Dashboard bán vé | Organizer | — | Real-time; Organizer-scoped |
| UC11.7 — Xem báo cáo check-in hiện trường | Organizer | — | Ngày diễn ra sự kiện |
| UC11.8 — Yêu cầu quyết toán doanh thu | Organizer | Finance Service | Sau khi COMPLETED |

