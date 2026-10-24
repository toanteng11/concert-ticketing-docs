# 03 — Thiết Kế Cơ Sở Dữ Liệu & Thực Thể (ERD & Database Schema Design)

## 1. Giới Thiệu & Nguyên Tắc Thiết Kế

Tài liệu này đặc tả toàn bộ lược đồ cơ sở dữ liệu quan hệ (**Database Schema**) của hệ thống bán vé sự kiện âm nhạc, bao gồm 19 bảng thực thể cốt lõi, kiểu dữ liệu chuẩn **MySQL 8.0+ (InnoDB Engine, utf8mb4)**, các khóa chính (PK), khóa ngoại (FK), ràng buộc toàn vẹn (Constraints) và ma trận quan hệ (Entity Relationships).

Người đọc và kỹ sư kiến trúc dùng tài liệu này làm đầu vào trực tiếp để vẽ sơ đồ **Entity Relationship Diagram (ERD) trên công cụ Astah** và sinh mã di chuyển lược đồ cơ sở dữ liệu (**Database Migrations - Flyway / Liquibase** trong dự án Java Spring Boot).

---

## 2. Ma Trận Quan Hệ Giữa Các Thực Thể (Entity Relationship Matrix)

| Thực thể nguồn (From) | Mối quan hệ | Thực thể đích (To) | Ràng buộc toàn vẹn & Khóa ngoại | Ý nghĩa nghiệp vụ |
|---|---|---|---|---|
| `users` | $1 : N$ | `user_sessions` | `ON DELETE CASCADE` | Một người dùng có nhiều phiên đăng nhập trên nhiều thiết bị |
| `users` | $1 : 1$ | `user_credentials` | `ON DELETE CASCADE` | Một người dùng có duy nhất một bản ghi xác thực mật khẩu |
| `venues` | $1 : N$ | `concerts` | `ON DELETE RESTRICT` | Một địa điểm (SVĐ) có thể tổ chức nhiều concert theo thời gian |
| `artists` | $1 : N$ | `concerts` | `ON DELETE RESTRICT` | Một nghệ sĩ có thể có nhiều show diễn |
| `users (Organizer)` | $1 : N$ | `concerts` | `ON DELETE RESTRICT` | Một Ban tổ chức quản lý nhiều concert trên nền tảng |
| `concerts` | $1 : N$ | `ticket_zones` | `ON DELETE CASCADE` | Một concert chia thành nhiều khu vực/hạng vé (VIP, CAT 1...) |
| `ticket_zones` | $1 : N$ | `seats` | `ON DELETE CASCADE` | Mỗi khu vực chứa hàng nghìn ghế ngồi vật lý |
| `seats` | $1 : N$ | `seat_holds` | `ON DELETE CASCADE` | Một ghế có thể có nhiều lịch sử giữ chỗ qua các phiên |
| `users` | $1 : N$ | `orders` | `ON DELETE RESTRICT` | Khách hàng sở hữu nhiều đơn hàng |
| `orders` | $1 : N$ | `order_items` | `ON DELETE CASCADE` | Một đơn hàng chứa từ 1 đến 4 vé (chi tiết từng ghế) |
| `seats` | $1 : 1$ | `order_items` | `ON DELETE RESTRICT` | Mỗi chiếc ghế trong đơn hàng chỉ được gán cho 1 item duy nhất |
| `orders` | $1 : N$ | `tickets` | `ON DELETE RESTRICT` | Một đơn hàng thành công phát hành số lượng vé tương ứng |
| `orders` | $1 : N$ | `payment_transactions` | `ON DELETE RESTRICT` | Một đơn hàng có thể có nhiều giao dịch thử thanh toán |
| `orders` | $1 : N$ | `refunds` | `ON DELETE RESTRICT` | Đơn hàng hủy phát sinh bản ghi hoàn tiền |
| `tickets` | $1 : N$ | `checkin_logs` | `ON DELETE RESTRICT` | Một vé lưu vết các lần quét qua cổng (thành công hoặc lỗi) |
| `tickets` | $1 : 1$ | `reviews` | `ON DELETE SET NULL` | Một vé qua cổng cho phép viết 1 bài đánh giá concert |
| `users` | $1 : N$ | `notifications` | `ON DELETE CASCADE` | Người dùng nhận nhiều thông báo trong hộp thư |

---

## 3. Đặc Tả Chi Tiết 19 Bảng Dữ Liệu Cốt Lõi (Database Tables Specification)

### Nhóm 1: Quản Trị Người Dùng & Phiên (User & Authentication)

#### 1. Bảng `users` (Thông tin người dùng)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Định danh duy nhất người dùng |
| `email` | VARCHAR(255) | No | — | **UNIQUE** | Địa chỉ thư điện tử |
| `phone_number` | VARCHAR(20) | No | — | **UNIQUE** | Số điện thoại chuẩn VN (E.164) |
| `full_name` | VARCHAR(100) | No | — | — | Họ và tên hiển thị |
| `role` | VARCHAR(30) | No | `'CUSTOMER'` | **CHECK** (`role IN ('CUSTOMER', 'ORGANIZER', 'ADMIN', 'STAFF')`) | Vai trò phân quyền |
| `status` | VARCHAR(20) | No | `'ACTIVE'` | **CHECK** (`status IN ('ACTIVE', 'LOCKED', 'DELETED')`) | Trạng thái tài khoản |
| `avatar_url` | TEXT | Yes | NULL | — | Đường dẫn ảnh đại diện trên S3 |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm đăng ký |
| `updated_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm cập nhật cuối |

#### 2. Bảng `user_credentials` (Bảo mật tài khoản)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)`, **UNIQUE** | Khóa ngoại người dùng |
| `password_hash` | VARCHAR(255) | No | — | — | Mật khẩu băm bcrypt cost 12 |
| `failed_attempts` | SMALLINT | No | 0 | — | Số lần đăng nhập sai liên tiếp |
| `locked_until` | TIMESTAMPTZ | Yes | NULL | — | Thời điểm hết hạn khóa tạm thời |
| `two_factor_secret` | VARCHAR(64) | Yes | NULL | — | Khóa bí mật 2FA TOTP (nếu bật) |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |

#### 3. Bảng `user_sessions` (Quản lý phiên đăng nhập)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Khóa ngoại người dùng |
| `refresh_token_hash`| VARCHAR(64) | No | — | **UNIQUE** | Băm SHA-256 của Refresh Token |
| `device_info` | VARCHAR(255) | Yes | NULL | — | Tên trình duyệt / Thiết bị |
| `ip_address` | VARCHAR(45) | Yes | NULL | — | Địa chỉ IP đăng nhập |
| `expires_at` | TIMESTAMPTZ | No | — | — | Thời điểm hết hạn (7 ngày) |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm khởi tạo phiên |

---

### Nhóm 2: Sự Kiện, Địa Điểm & Sơ Đồ Khán Đài (Catalog & Venues)

#### 4. Bảng `venues` (Địa điểm tổ chức)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Định danh địa điểm |
| `name` | VARCHAR(200) | No | — | — | Tên địa điểm (VD: SVĐ Mỹ Đình) |
| `address` | TEXT | No | — | — | Địa chỉ chi tiết |
| `city` | VARCHAR(100) | No | — | — | Thành phố (Hà Nội, TP.HCM...) |
| `total_capacity`| INTEGER | No | — | **CHECK** (`total_capacity > 0`) | Tổng sức chứa thiết kế |
| `layout_svg_url`| TEXT | Yes | NULL | — | File SVG sơ đồ kiến trúc gốc |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |

#### 5. Bảng `artists` (Nghệ sĩ biểu diễn)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Định danh nghệ sĩ |
| `name` | VARCHAR(150) | No | — | — | Tên nghệ sĩ / Ban nhạc |
| `bio` | TEXT | Yes | NULL | — | Giới thiệu tiểu sử nghệ sĩ |
| `image_url` | TEXT | Yes | NULL | — | Đường dẫn ảnh đại diện nghệ sĩ |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |

#### 6. Bảng `concerts` (Thông tin sự kiện ca nhạc)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Định danh concert |
| `organizer_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Ban tổ chức sở hữu |
| `venue_id` | UUID | No | — | **FK** $\rightarrow$ `venues(id)` | Địa điểm tổ chức |
| `artist_id` | UUID | No | — | **FK** $\rightarrow$ `artists(id)` | Nghệ sĩ chính |
| `title` | VARCHAR(255) | No | — | — | Tên buổi biểu diễn |
| `description` | TEXT | Yes | NULL | — | Mô tả chi tiết chương trình |
| `banner_url` | TEXT | Yes | NULL | — | Ảnh poster chính thức |
| `status` | VARCHAR(30) | No | `'DRAFT'` | **CHECK** (`status IN ('DRAFT', 'PENDING_REVIEW', 'PUBLISHED', 'ON_SALE', 'SOLD_OUT', 'COMPLETED', 'CANCELLED')`) | Trạng thái vòng đời concert |
| `sale_start_time`| TIMESTAMPTZ | No | — | — | Giờ bắt đầu mở bán vé |
| `show_start_time`| TIMESTAMPTZ | No | — | — | Giờ bắt đầu biểu diễn |
| `show_end_time` | TIMESTAMPTZ | No | — | — | Giờ kết thúc dự kiến |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |
| `updated_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm cập nhật cuối |

#### 7. Bảng `ticket_zones` (Phân khu / Hạng vé)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Định danh khu vực vé |
| `concert_id` | UUID | No | — | **FK** $\rightarrow$ `concerts(id)` | Khóa ngoại sự kiện |
| `name` | VARCHAR(100) | No | — | — | Tên hạng vé (VIP 1, CAT 1...) |
| `code` | VARCHAR(20) | No | — | — | Mã phân khu (Z_VIP_1) |
| `price` | NUMERIC(15,2)| No | — | **CHECK** (`price >= 0`) | Đơn giá niêm yết (VNĐ) |
| `total_capacity`| INTEGER | No | — | **CHECK** (`total_capacity > 0`) | Tổng số ghế/vé của Zone |
| `color_hex` | VARCHAR(7) | No | `'#3B82F6'` | — | Mã màu hiển thị trên sơ đồ |
| `is_seated` | BOOLEAN | No | `TRUE` | — | `TRUE`: Ghế ngồi; `FALSE`: Đứng (GA) |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |

#### 8. Bảng `seats` (Ghế ngồi vật lý chi tiết)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Định danh ghế |
| `concert_id` | UUID | No | — | **FK** $\rightarrow$ `concerts(id)` | Thuộc concert nào |
| `zone_id` | UUID | No | — | **FK** $\rightarrow$ `ticket_zones(id)` | Thuộc khu vực nào |
| `row_label` | VARCHAR(10) | Yes | NULL | — | Tên hàng ghế (Hàng A, B, C...) |
| `seat_number` | VARCHAR(10) | Yes | NULL | — | Số ghế (Số 01, 02...) |
| `status` | VARCHAR(20) | No | `'AVAILABLE'` | **CHECK** (`status IN ('AVAILABLE', 'HELD', 'SOLD', 'BLOCKED')`) | Trạng thái ghế |
| `coordinate_x` | INTEGER | Yes | NULL | — | Tọa độ X trên sơ đồ SVG |
| `coordinate_y` | INTEGER | Yes | NULL | — | Tọa độ Y trên sơ đồ SVG |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |
| `updated_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm cập nhật cuối |

---

### Nhóm 3: Giữ Ghế, Đơn Hàng & Thanh Toán (Booking, Orders & Payments)

#### 9. Bảng `seat_holds` (Nhật ký giữ ghế tạm thời)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Người đang giữ ghế |
| `seat_id` | UUID | No | — | **FK** $\rightarrow$ `seats(id)` | Ghế được giữ |
| `concert_id` | UUID | No | — | **FK** $\rightarrow$ `concerts(id)` | Sự kiện liên quan |
| `hold_token` | VARCHAR(64) | No | — | **UNIQUE** | Khóa bí mật giao dịch giữ ghế |
| `status` | VARCHAR(20) | No | `'ACTIVE'` | **CHECK** (`status IN ('ACTIVE', 'COMPLETED', 'EXPIRED', 'RELEASED')`) | Trạng thái giữ chỗ |
| `expires_at` | TIMESTAMPTZ | No | — | — | Thời điểm hết hạn (NOW + 10 phút) |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm bấm giữ |

#### 10. Bảng `vouchers` (Mã giảm giá & Khuyến mãi)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính |
| `concert_id` | UUID | Yes | NULL | **FK** $\rightarrow$ `concerts(id)` | NULL = Toàn sàn |
| `code` | VARCHAR(30) | No | — | **UNIQUE** | Mã nhập khuyến mãi (VD: `SUMMER2026`) |
| `discount_type` | VARCHAR(20) | No | `'PERCENT'` | **CHECK** (`discount_type IN ('PERCENT', 'FIXED_AMOUNT')`) | Loại chiết khấu |
| `discount_value`| NUMERIC(15,2)| No | — | — | Giá trị giảm (% hoặc số tiền) |
| `max_discount` | NUMERIC(15,2)| Yes | NULL | — | Số tiền giảm tối đa (với %) |
| `min_order_value`| NUMERIC(15,2)| No | 0 | — | Giá trị đơn tối thiểu để áp dụng |
| `usage_limit` | INTEGER | No | 1000 | — | Số lượt dùng tối đa toàn chiến dịch |
| `used_count` | INTEGER | No | 0 | — | Số lượt đã sử dụng thực tế |
| `start_time` | TIMESTAMPTZ | No | — | — | Thời điểm bắt đầu có hiệu lực |
| `end_time` | TIMESTAMPTZ | No | — | — | Thời điểm hết hạn mã |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |

#### 11. Bảng `orders` (Đơn hàng đặt vé)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính đơn hàng |
| `order_code` | VARCHAR(32) | No | — | **UNIQUE** | Mã hiển thị cho khách (VD: `ORD-20261024-8F9B`) |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Khách hàng mua vé |
| `concert_id` | UUID | No | — | **FK** $\rightarrow$ `concerts(id)` | Sự kiện đặt vé |
| `total_amount` | NUMERIC(15,2)| No | — | — | Tổng tiền gốc các vé |
| `discount_amount`| NUMERIC(15,2)| No | 0 | — | Số tiền chiết khấu voucher |
| `final_amount` | NUMERIC(15,2)| No | — | **CHECK** (`final_amount >= 0`) | Số tiền thực trả cuối cùng |
| `status` | VARCHAR(30) | No | `'PENDING'` | **CHECK** (`status IN ('PENDING', 'PAID', 'CANCELLED', 'OUT_TIME', 'REFUND_PENDING', 'REFUNDED')`) | Trạng thái đơn hàng |
| `payment_gateway`| VARCHAR(20) | Yes | NULL | **CHECK** (`payment_gateway IN ('VNPAY', 'MOMO')`) | Cổng thanh toán lựa chọn |
| `hold_token` | VARCHAR(64) | Yes | NULL | — | Khóa giữ ghế liên kết |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo đơn |
| `updated_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm cập nhật cuối |

#### 12. Bảng `order_items` (Chi tiết ghế trong đơn)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính |
| `order_id` | UUID | No | — | **FK** $\rightarrow$ `orders(id)` | Thuộc đơn hàng nào |
| `seat_id` | UUID | No | — | **FK** $\rightarrow$ `seats(id)`, **UNIQUE** | Ghế được mua |
| `zone_id` | UUID | No | — | **FK** $\rightarrow$ `ticket_zones(id)` | Hạng vé của ghế |
| `price` | NUMERIC(15,2)| No | — | — | Giá tiền tính tại thời điểm mua |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo |

#### 13. Bảng `tickets` (Vé điện tử phát hành)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Định danh vé |
| `ticket_code` | VARCHAR(32) | No | — | **UNIQUE** | Mã vé in trên vé điện tử |
| `order_id` | UUID | No | — | **FK** $\rightarrow$ `orders(id)` | Đơn hàng phát hành |
| `order_item_id` | UUID | No | — | **FK** $\rightarrow$ `order_items(id)` | Item cụ thể |
| `seat_id` | UUID | No | — | **FK** $\rightarrow$ `seats(id)` | Ghế tương ứng |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Chủ sở hữu vé |
| `concert_id` | UUID | No | — | **FK** $\rightarrow$ `concerts(id)` | Sự kiện tham gia |
| `qr_code_hash` | VARCHAR(128) | No | — | **UNIQUE** | Chuỗi băm đối chiếu mã QR |
| `status` | VARCHAR(20) | No | `'ORDERED'` | **CHECK** (`status IN ('ORDERED', 'CHECKED', 'CANCELLED')`) | Trạng thái vé |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm phát hành |
| `updated_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm cập nhật |

#### 14. Bảng `payment_transactions` (Lịch sử giao dịch cổng thanh toán)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính |
| `order_id` | UUID | No | — | **FK** $\rightarrow$ `orders(id)` | Đơn hàng thanh toán |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Người chuyển tiền |
| `provider` | VARCHAR(20) | No | — | **CHECK** (`provider IN ('VNPAY', 'MOMO')`) | Đơn vị cổng |
| `provider_transaction_id`| VARCHAR(100)| Yes | NULL | — | Mã giao dịch ngân hàng đối tác |
| `idempotency_key` | VARCHAR(64) | No | — | **UNIQUE** | Khóa chống trùng lặp |
| `amount` | NUMERIC(15,2)| No | — | — | Số tiền thanh toán (VNĐ) |
| `status` | VARCHAR(20) | No | `'INITIATED'` | **CHECK** (`status IN ('INITIATED', 'PROCESSING', 'SUCCESS', 'FAILED')`) | Trạng thái giao dịch |
| `payload_raw` | JSONB | Yes | NULL | — | Toàn bộ phản hồi thô từ Webhook IPN |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm bắt đầu |
| `updated_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm hoàn tất |

#### 15. Bảng `refunds` (Yêu cầu & Lịch sử hoàn tiền)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính hoàn tiền |
| `order_id` | UUID | No | — | **FK** $\rightarrow$ `orders(id)` | Đơn hàng được hoàn |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Người nhận hoàn tiền |
| `reason` | TEXT | No | — | — | Lý do hoàn tiền |
| `refund_rate` | SMALLINT | No | — | **CHECK** (`refund_rate IN (100, 50, 0)`) | Tỷ lệ hoàn theo chính sách (%) |
| `refund_amount` | NUMERIC(15,2)| No | — | — | Số tiền thực tế hoàn lại |
| `status` | VARCHAR(20) | No | `'PENDING'` | **CHECK** (`status IN ('PENDING', 'PROCESSING', 'COMPLETED', 'REJECTED')`) | Trạng thái xử lý |
| `provider_refund_id`| VARCHAR(100)| Yes | NULL | — | Mã giao dịch hoàn tiền phía đối tác |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm tạo yêu cầu |
| `updated_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm hoàn tất |

---

### Nhóm 4: Soát Vé, Đánh Giá & Vận Hành (Gate, Reviews & Auditing)

#### 16. Bảng `checkin_logs` (Nhật ký quét mã QR tại cổng)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính bản ghi |
| `ticket_id` | UUID | No | — | **FK** $\rightarrow$ `tickets(id)` | Vé được quét |
| `concert_id` | UUID | No | — | **FK** $\rightarrow$ `concerts(id)` | Sự kiện |
| `gate_id` | VARCHAR(30) | No | — | — | Cổng soát vé (GATE_A_VIP) |
| `staff_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Nhân viên thực hiện quét |
| `device_id` | VARCHAR(64) | No | — | — | Mã thiết bị cầm tay |
| `checkin_time` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm quét chính xác |
| `checkin_mode` | VARCHAR(20) | No | `'ONLINE'` | **CHECK** (`checkin_mode IN ('ONLINE', 'OFFLINE_SYNC', 'MANUAL_OVERRIDE')`) | Chế độ soát vé |
| `result` | VARCHAR(30) | No | — | **CHECK** (`result IN ('SUCCESS', 'ALREADY_USED', 'INVALID_SIGNATURE', 'CANCELLED', 'WRONG_GATE')`) | Kết quả kiểm tra |

#### 17. Bảng `reviews` (Đánh giá của khán giả thực tế)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính bài đánh giá |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Tác giả bài viết |
| `concert_id` | UUID | No | — | **FK** $\rightarrow$ `concerts(id)` | Concert được đánh giá |
| `ticket_id` | UUID | No | — | **FK** $\rightarrow$ `tickets(id)`, **UNIQUE** | Chứng chỉ vé đã tham dự |
| `rating` | SMALLINT | No | — | **CHECK** (`rating BETWEEN 1 AND 5`) | Số sao đánh giá (1–5 sao) |
| `comment` | TEXT | Yes | NULL | — | Nội dung nhận xét |
| `is_verified_attendee`| BOOLEAN | No | `TRUE` | — | Xác nhận đã qua cổng check-in |
| `status` | VARCHAR(20) | No | `'PUBLISHED'` | **CHECK** (`status IN ('PUBLISHED', 'HIDDEN', 'REPORTED')`) | Trạng thái hiển thị |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm đăng |

#### 18. Bảng `notifications` (Hộp thư thông báo người dùng)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính thông báo |
| `user_id` | UUID | No | — | **FK** $\rightarrow$ `users(id)` | Người nhận thông báo |
| `event_type` | VARCHAR(50) | No | — | — | Mã sự kiện (ORDER_SUCCESS...) |
| `channel` | VARCHAR(20) | No | — | **CHECK** (`channel IN ('IN_APP', 'EMAIL', 'PUSH', 'SMS')`) | Kênh phân phối |
| `title` | VARCHAR(255) | No | — | — | Tiêu đề thông báo |
| `content` | TEXT | No | — | — | Nội dung tóm tắt |
| `status` | VARCHAR(20) | No | `'SENT'` | **CHECK** (`status IN ('PENDING', 'SENT', 'READ', 'FAILED')`) | Trạng thái gửi tin |
| `sent_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm gửi đi |
| `read_at` | TIMESTAMPTZ | Yes | NULL | — | Thời điểm người dùng mở đọc |

#### 19. Bảng `audit_logs` (Nhật ký kiểm toán bảo mật bất biến)
| Tên cột | Kiểu dữ liệu | Nullable | Mặc định | Khóa / Ràng buộc | Diễn giải |
|---|---|---|---|---|---|
| `id` | UUID | No | `gen_random_uuid()` | **PK** | Khóa chính bản ghi kiểm toán |
| `actor_id` | UUID | No | — | — | ID của Admin/Organizer thực hiện |
| `actor_role` | VARCHAR(30) | No | — | — | Vai trò tại thời điểm thực thi |
| `action_type` | VARCHAR(50) | No | — | — | Hành động (VD: CONCERT_CANCELLED) |
| `resource_type` | VARCHAR(50) | No | — | — | Đối tượng bị tác động |
| `resource_id` | VARCHAR(100) | No | — | — | Khóa định danh của đối tượng |
| `ip_address` | VARCHAR(45) | Yes | NULL | — | Địa chỉ IP máy thực hiện |
| `user_agent` | TEXT | Yes | NULL | — | Trình duyệt / Ứng dụng |
| `payload_before`| JSONB | Yes | NULL | — | Trạng thái dữ liệu trước thay đổi |
| `payload_after` | JSONB | Yes | NULL | — | Trạng thái dữ liệu sau thay đổi |
| `created_at` | TIMESTAMPTZ | No | `NOW()` | — | Thời điểm thực hiện (Bất biến) |
