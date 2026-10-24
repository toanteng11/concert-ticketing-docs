# 05 — Tối Ưu Hóa Cơ Sở Dữ Liệu & An Toàn Bảo Mật Hệ Thống (DB Optimization & Security)

## 1. Giới Thiệu & Mục Tiêu

Một hệ thống bán vé concert quy mô lớn phải đáp ứng đồng thời hai yêu cầu tưởng chừng như đối nghịch:
1. **Tối ưu hóa hiệu năng cơ sở dữ liệu cực đại:** Đảm bảo thời gian phản hồi câu truy vấn $< 10\text{ ms}$ cho các bảng chứa hàng chục triệu bản ghi trong điều kiện tải ghi dồn dập.
2. **An toàn bảo mật cấp độ doanh nghiệp (Enterprise Security):** Bảo vệ tuyệt đối thông tin thanh toán, chống tấn công đánh cắp vé, ngăn chặn tài khoản ảo (Bot Farming) và tuân thủ các tiêu chuẩn bảo mật dữ liệu.

Tài liệu này đặc tả chi tiết kiến trúc tối ưu hóa cơ sở dữ liệu MySQL 8.0 và toàn bộ các lớp phòng thủ an ninh thông tin của hệ thống.

---

## 2. Chiến Lược Tối Ưu Hóa Cơ Sở Dữ Liệu (MySQL 8.0 Optimization)

### 1. Thiết Kế Chỉ Mục Thông Minh (Targeted & Partial Indexing)
Tránh tạo chỉ mục tràn lan gây chậm thao tác ghi (`INSERT`/`UPDATE`), hệ thống tập trung vào các chỉ mục có tính chọn lọc cao:

```sql
-- 1. Partial Index cực nhanh để tìm ghế trống (loại trừ các ghế đã bán)
CREATE INDEX idx_seats_available ON seats (concert_id, zone_id) 
WHERE status = 'AVAILABLE';

-- 2. Composite Index tối ưu hóa truy vấn lịch sử đơn hàng của người dùng
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);

-- 3. Unique Index chống tạo đơn trùng lặp cho cùng một lượt giữ ghế
CREATE UNIQUE INDEX uq_order_hold_token ON orders (hold_token) 
WHERE status != 'CANCELLED';

-- 4. B-tree Index phục vụ quét vé siêu tốc tại cổng
CREATE INDEX idx_tickets_qr_hash ON tickets (qr_code_hash) 
INCLUDE (id, concert_id, status, zone_id);
```

### 2. Phân Vùng Dữ Liệu Khối Lớn (Declarative Table Partitioning)
Đối với các bảng có khối lượng bản ghi tăng trưởng nhanh chóng theo thời gian, áp dụng phân vùng theo phạm vi thời gian (**Range Partitioning**):
- **Bảng `audit_logs` & `checkin_logs`:** Phân vùng theo tháng (`PARTITION BY RANGE (created_at)`).
- **Lợi ích:** Khi kiểm toán một sự kiện diễn ra vào tháng 10/2026, câu truy vấn chỉ quét duy nhất phân vùng `checkin_logs_2026_10` (Partition Pruning), giảm $95\%$ số trang đĩa phải đọc. Đồng thời cho phép dọn dẹp dữ liệu cũ sau 5 năm chỉ bằng một lệnh `DROP TABLE partition_name` tức thì mà không gây khóa bảng.

### 3. Tinh Chỉnh Cơ Chế Dọn Rác Tự Động (Aggressive Autovacuum Tuning)
Các bảng có tần suất cập nhật trạng thái cực cao như `seats` (chuyển đổi liên tục giữa `AVAILABLE` $\leftrightarrow$ `HELD` $\rightarrow$ `SOLD`) tạo ra một lượng lớn dòng chết (Dead Tuples - MVCC Bloat):

```ini
# Tinh chỉnh riêng cho bảng seats và seat_holds
ALTER TABLE seats SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- Kích hoạt vacuum khi 5% số dòng bị thay đổi
    autovacuum_vacuum_cost_limit = 1000,    -- Cấp thêm tài nguyên I/O cho tiến trình dọn dẹp
    autovacuum_vacuum_cost_delay = 2        -- Giảm thời gian trễ giữa các lần dọn
);
```

### 4. Quản Lý Kết Nối Bằng HikariCP / ProxySQL (Transaction Pooling)
- Thay vì cấp mỗi Microservice một kết nối riêng làm cạn kiệt RAM Database:
- Triển khai cụm HikariCP / ProxySQL ở chế độ `pool_mode = transaction`.
- Các kết nối client chỉ giữ kết nối vật lý tới MySQL 8.0 trong khoảng thời gian thực thi một giao dịch (`BEGIN` $\rightarrow$ `COMMIT`). Ngay sau khi hoàn tất, kết nối được thu hồi và gán cho request khác, cho phép $150$ kết nối vật lý phục vụ hơn $10.000$ client đồng thời.

---

## 3. Kiến Trúc Xác Thực & Phân Quyền (Authentication & RBAC)

### 1. Cơ Chế Xác Thực Kép (Dual-Token Pattern)
- **Access Token:** Định dạng JWT chuẩn (RS256 - ký bằng khóa riêng Private Key). Thời hạn sống ngắn (**15 phút**). Mang các thông tin phân quyền tối thiểu (`user_id`, `role`, `email`).
- **Refresh Token:** Chuỗi ngẫu nhiên có độ dài 256 bits, được mã hóa băm SHA-256 lưu trong cơ sở dữ liệu `user_sessions`. Thời hạn sống dài (**7 ngày**).
- **Bảo vệ Cookie trên Web:** Refresh Token được lưu trữ trong Cookie với đầy đủ các thuộc tính bảo vệ nghiêm ngặt:
  ```http
  Set-Cookie: refresh_token=abc123xyz...; Path=/api/v1/auth; HttpOnly; Secure; SameSite=Strict; Max-Age=604800
  ```
  - `HttpOnly`: Mã JavaScript trong trình duyệt hoàn toàn không thể đọc được Cookie này $\rightarrow$ Miễn nhiễm với tấn công XSS đánh cắp token.
  - `Secure`: Chỉ truyền tải qua kênh mã hóa HTTPS.
  - `SameSite=Strict`: Ngăn chặn triệt để tấn công Cross-Site Request Forgery (CSRF).

### 2. Ma Trận Phân Quyền Theo Vai Trò (Role-Based Access Control - RBAC)
Hệ thống thiết lập 4 nhóm vai trò (Roles) với ranh giới quyền hạn phân định rõ:

| Nhóm quyền hạn | Customer (Khách hàng) | Organizer (Ban tổ chức) | Admin (Quản trị sàn) | Check-in Staff (Soát vé) |
|---|---|---|---|---|
| **Xem concert & Chọn ghế** | ✅ Được phép | ✅ Được phép | ✅ Được phép | ❌ Không |
| **Giữ ghế & Thanh toán** | ✅ Được phép | ❌ Không | ❌ Không | ❌ Không |
| **Tạo & Cập nhật Concert** | ❌ Không | ✅ Chỉ concert của mình | ✅ Toàn quyền override | ❌ Không |
| **Phê duyệt & Hủy Concert** | ❌ Không | ❌ Không | ✅ Toàn quyền | ❌ Không |
| **Xem Báo cáo Doanh thu** | ❌ Không | ✅ Chỉ concert của mình | ✅ Toàn bộ sàn | ❌ Không |
| **Quét mã QR Check-in** | ❌ Không | ❌ Không | ✅ Giám sát | ✅ Chỉ cổng được gán |
| **Khóa / Mở khóa Tài khoản** | ❌ Không | ❌ Không | ✅ Toàn quyền | ❌ Không |

---

## 4. Các Biện Pháp Bảo Mật Ứng Dụng (Application Security Controls)

### 1. Phòng Chống Tấn Công Tiêm Lệnh (SQL Injection & XSS Defense)
- **Zero SQL Injection:** Nghiêm cấm $100\%$ việc nối chuỗi câu lệnh SQL trong mã nguồn. Toàn bộ các truy vấn bắt buộc phải dùng **Parameterized Queries (Prepared Statements)** qua ORM/Query Builder (như Prisma, TypeORM, GORM, Hibernate).
- **Zero Cross-Site Scripting (XSS):** Toàn bộ dữ liệu nhập từ người dùng (tên hiển thị, nội dung bình luận, đánh giá) được làm sạch (Sanitize) và mã hóa ký tự đặc biệt (HTML Entity Encoding) trước khi hiển thị ra giao diện.

### 2. Phòng Chống Lỗ Hổng Truy Cập Trực Tiếp Đối Tượng (IDOR Defense)
- Tuyệt đối không cho phép truy vấn đơn hàng hoặc vé chỉ dựa vào `id` truyền từ URL (ví dụ: `GET /orders/123`).
- **Mọi truy vấn dữ liệu nhạy cảm bắt buộc phải kiểm tra kèm định danh tài khoản đã xác thực từ JWT:**
  ```sql
  -- Chống IDOR: Chỉ trả về đơn hàng nếu nó thuộc về chính user đang đăng nhập
  SELECT * FROM orders WHERE id = :order_id AND user_id = :authenticated_user_id;
  ```

### 3. An Toàn Dữ Liệu Thẻ Thanh Toán (PCI-DSS Compliance)
- **Nguyên tắc không lưu trữ dữ liệu nhạy cảm:** Hệ thống **HOÀN TOÀN KHÔNG** lưu trữ số thẻ tín dụng đầy đủ (PAN), ngày hết hạn, hoặc mã bảo mật CVV/CVC trên máy chủ nội bộ.
- Mọi thao tác nhập thẻ đều được chuyển hướng (Redirect) hoặc nhúng qua Iframe bảo mật trực tiếp của Cổng thanh toán được cấp chứng chỉ **PCI-DSS Level 1** (VNPay, MoMo, OnePay). Hệ thống nội bộ chỉ lưu trữ mã giao dịch tham chiếu (Tokenized Transaction Reference).

### 4. Tường Lửa Ứng Dụng Web & Chống Tấn Công DDoS (WAF & Rate Limiting)
- Triển khai **Cloudflare Web Application Firewall (WAF)** ở tuyến đầu để lọc các gói tin độc hại:
  - Tự động chặn các mẫu tấn công đã biết trong danh mục OWASP Top 10.
  - Ngăn chặn các công cụ quét lỗ hổng tự động (Nmap, SQLmap, Nikto).
  - Áp dụng kiểm tra thử thách hình ảnh (Cloudflare Turnstile CAPTCHA) khi phát hiện hành vi truy cập bất thường giống bot trước khi cho phép vào phòng chờ.

