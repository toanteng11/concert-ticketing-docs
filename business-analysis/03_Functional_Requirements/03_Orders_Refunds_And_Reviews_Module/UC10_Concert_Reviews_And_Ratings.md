# UC10 — Đánh Giá & Nhận Xét Concert (Concert Reviews & Ratings)

## 1. Thông Tin Chung

| Trường | Nội dung |
|---|---|
| **Use Case ID** | UC10 |
| **Tên Use Case** | Đánh Giá & Nhận Xét Concert |
| **Module** | 03 — Orders, Refunds & Reviews Module |
| **Actor chính** | Khách hàng (Customer) |
| **Actor phụ** | Admin (kiểm duyệt vi phạm) |
| **Mức độ ưu tiên** | 🟢 Medium |
| **Precondition** | Customer có vé `CHECKED` cho concert đó; concert đang ở trạng thái `COMPLETED`; cổng đánh giá đang mở (chưa quá 30 ngày sau concert) |
| **BR liên quan** | BR-REV-001 đến BR-REV-004 |

---

## 2. Tóm Tắt

Use case này cho phép khách hàng đã thực sự tham dự concert (check-in hợp lệ) gửi đánh giá sao và nhận xét văn bản. Hệ thống kiểm duyệt tự động từ ngữ vi phạm trước khi hiển thị công khai.

---

## 3. Điều Kiện Tiên Quyết (Preconditions)

- Customer đã đăng nhập.
- Customer có ít nhất 1 vé ở trạng thái `CHECKED` cho concert cụ thể.
- `concert.status = COMPLETED`.
- Cổng đánh giá còn mở: `current_time < concert.completed_at + 30 days`.
- Customer chưa có đánh giá nào cho concert này.

---

## 4. Điều Kiện Kết Thúc (Postconditions)

**Thành công:** Đánh giá được lưu và hiển thị công khai trên trang chi tiết concert; rating trung bình của concert được cập nhật.

**Thất bại:** Đánh giá bị từ chối (nội dung vi phạm), thông báo lý do và yêu cầu chỉnh sửa.

---

## 5. Luồng Chính (Main Flow)

| Bước | Actor | Hành động | Phản hồi hệ thống |
|---|---|---|---|
| 1 | Customer | Vào trang chi tiết concert COMPLETED → Tab "Đánh giá" | Kiểm tra điều kiện đủ quyền đánh giá (vé CHECKED?) |
| 2 | System | Xác nhận đủ quyền | Hiển thị form đánh giá: Rating sao (1-5), Tiêu đề (tùy chọn), Nội dung nhận xét |
| 3 | Customer | Chọn số sao (bắt buộc), viết nhận xét (tùy chọn, tối đa 1000 ký tự) | Real-time đếm số ký tự còn lại |
| 4 | Customer | Nhấn "Gửi đánh giá" | Hệ thống chạy bộ lọc kiểm duyệt tự động |
| 5 | System | Nội dung vượt qua kiểm duyệt | Lưu đánh giá; tính lại rating trung bình concert; hiển thị công khai |
| 6 | System | Phản hồi cho customer | "Đánh giá của bạn đã được đăng thành công. Cảm ơn!" |

---

## 6. Luồng Thay Thế (Alternative Flows)

### AF-01: Chỉnh Sửa Đánh Giá Đã Đăng

| Bước | Mô tả |
|---|---|
| 1 | Customer vào đánh giá đã gửi → Click "Chỉnh sửa" (trong 30 ngày kể từ ngày đăng) |
| 2 | Form hiển thị nội dung đánh giá cũ để chỉnh sửa |
| 3 | Customer sửa và gửi lại → Qua bộ lọc kiểm duyệt một lần nữa |
| 4 | Lưu lại với thời gian chỉnh sửa; rating concert cập nhật theo sao mới |

### AF-02: Xóa Đánh Giá

| Bước | Mô tả |
|---|---|
| 1 | Customer click "Xóa đánh giá" (trong 30 ngày) |
| 2 | Popup xác nhận: "Bạn có chắc muốn xóa? Thao tác không thể hoàn tác." |
| 3 | Xóa đánh giá; cập nhật lại rating trung bình concert |

---

## 7. Luồng Ngoại Lệ (Exception Flows)

| ID | Điều kiện | Xử lý |
|---|---|---|
| EX-01 | Không có vé `CHECKED` cho concert này | Ẩn form đánh giá; hiển thị "Chỉ khách hàng đã tham dự mới có thể đánh giá" |
| EX-02 | Concert chưa ở trạng thái `COMPLETED` | Ẩn tab đánh giá; hoặc hiển thị "Cổng đánh giá sẽ mở sau khi concert kết thúc" |
| EX-03 | Quá 30 ngày sau concert | Ẩn form; hiển thị "Thời hạn đánh giá đã kết thúc" |
| EX-04 | Đánh giá chứa từ ngữ vi phạm | Hệ thống từ chối; highlight phần vi phạm; "Vui lòng điều chỉnh nội dung: [mô tả vi phạm]" |
| EX-05 | Customer cố gửi đánh giá thứ 2 cho cùng concert | API trả `409: REVIEW_ALREADY_EXISTS`; hiển thị đánh giá cũ với nút chỉnh sửa |

---

## 8. User Stories & Acceptance Criteria

### US-10-01: Gửi Đánh Giá Concert Đã Tham Dự

> **As a** khách hàng đã tham dự concert,  
> **I want to** đánh giá concert bằng sao và nhận xét văn bản,  
> **So that** chia sẻ trải nghiệm và giúp người khác đưa ra quyết định.

| AC | Mô tả | Loại |
|---|---|---|
| AC-10-01-01 | Chỉ customer có vé `CHECKED` cho concert mới thấy form đánh giá; customer không tham dự không thể đánh giá dưới mọi hình thức | Functional (Critical) |
| AC-10-01-02 | Rating sao từ 1 đến 5 là bắt buộc; nhận xét văn bản là tùy chọn | Functional |
| AC-10-01-03 | Đánh giá hiển thị công khai kèm tên người đánh giá (có thể dùng tên hiển thị đã chọn trong hồ sơ) và avatar | Functional |
| AC-10-01-04 | Rating trung bình của concert được tính lại ngay sau khi đánh giá được đăng | Functional |
| AC-10-01-05 | Nội dung nhận xét qua bộ lọc tự động trong < 500ms; nếu vi phạm không được đăng, phản hồi cụ thể | Performance |

---

### US-10-02: Xem Đánh Giá Concert

> **As a** người dùng bất kỳ (kể cả chưa đăng nhập),  
> **I want to** đọc đánh giá của khán giả về một concert,  
> **So that** đánh giá chất lượng sự kiện trước khi quyết định mua vé.

| AC | Mô tả | Loại |
|---|---|---|
| AC-10-02-01 | Đánh giá hiển thị công khai trên trang concert không yêu cầu đăng nhập | Functional |
| AC-10-02-02 | Hiển thị rating trung bình (dạng sao), phân bố sao (1★-5★), tổng số đánh giá | Functional |
| AC-10-02-03 | Danh sách đánh giá có thể sắp xếp theo: Mới nhất, Hữu ích nhất, Điểm cao nhất, Điểm thấp nhất | Functional |
| AC-10-02-04 | Đánh giá vi phạm bị ẩn bởi Admin không còn hiển thị; không lộ lý do ẩn cho người dùng thường | Functional |

---

## 9. Danh Sách Use Case Con & Quan Hệ Actor

| Use Case Con | Actor Chính | Actor Phụ | Ghi Chú |
|---|---|---|---|
| UC10.1 — Gửi đánh giá mới | Customer | Content Moderation Service | Kiểm tra điều kiện tham dự |
| UC10.2 — Chỉnh sửa đánh giá | Customer | Content Moderation | Trong 30 ngày kể từ ngày đăng |
| UC10.3 — Xóa đánh giá | Customer | — | Trong 30 ngày |
| UC10.4 — Xem đánh giá concert | Guest, Customer | — | Không cần đăng nhập |
| UC10.5 — Kiểm duyệt đánh giá vi phạm | Admin | — | Ẩn đánh giá spam/độc hại |

