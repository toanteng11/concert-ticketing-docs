# 05 — Tối Ưu Hóa Cơ Sở Dữ Liệu & An Toàn Bảo Mật Hệ Thống (DB Optimization & Security)

## 1. Giới Thiệu & Mục Tiêu

Một hệ thống bán vé concert quy mô lớn phải đáp ứng đồng thời hai yêu cầu tưởng chừng như đối nghịch:
1. **Tối ưu hóa hiệu năng cơ sở dữ liệu cực đại:** Đảm bảo thời gian phản hồi câu truy vấn < 10ms cho các bảng chứa hàng chục triệu bản ghi trong điều kiện tải ghi dồn dập.
2. **An toàn bảo mật cấp độ doanh nghiệp (Enterprise Security):** Bảo vệ tuyệt đối thông tin thanh toán, chống tấn công đánh cắp vé, ngăn chặn tài khoản ảo (Bot Farming) và tuân thủ các tiêu chuẩn bảo mật dữ liệu.

Tài liệu này đặc tả chi tiết kiến trúc tối ưu hóa cơ sở dữ liệu MySQL 8.0 và toàn bộ các lớp phòng thủ an ninh thông tin của hệ thống.

---

## 2. QUYẾT ĐỊNH KIẾN TRÚC: CÓ NÊN DÙNG DATABASE SHARDING KHÔNG?

> **Kết luận chính thức: KHÔNG sử dụng Horizontal Sharding trong giai đoạn hiện tại.**

### 2.1. Sharding Là Gì & Khi Nào Cần Dùng?

**Database Sharding (Horizontal Sharding)** là kỹ thuật chia nhỏ một bảng dữ liệu khổng lồ thành nhiều phân mảnh (Shard), mỗi Shard lưu trên một máy chủ MySQL riêng biệt, theo một khóa phân mảnh (Shard Key — ví dụ `user_id % 4`, `concert_id % 8`).

**Sharding chỉ thực sự cần thiết khi:**
- Bảng dữ liệu đã vượt quá **1 tỷ bản ghi** và Index B-tree không còn fit vào RAM.
- Tốc độ ghi vượt quá giới hạn vật lý của **một máy chủ SSD NVMe** (thường > 500.000 writes/giây).
- Dung lượng lưu trữ vượt quá **10 TB** trên một máy chủ đơn.

### 2.2. Phân Tích Quy Mô Thực Tế Của Hệ Thống

| Bảng | Ước tính bản ghi (5 năm vận hành) | Kích thước ước tính | Cần Sharding? |
|---|---|---|---|
| `users` | ~2 triệu người dùng | ~2 GB | ❌ Không |
| `concerts` | ~5.000 sự kiện | ~50 MB | ❌ Không |
| `seats` | ~10 triệu ghế (500 concert × 20K ghế) | ~3 GB | ❌ Không |
| `orders` | ~20 triệu đơn hàng | ~15 GB | ❌ Không |
| `tickets` | ~15 triệu vé điện tử | ~10 GB | ❌ Không |
| `payment_transactions` | ~25 triệu giao dịch | ~20 GB | ❌ Không |
| `checkin_logs` | ~15 triệu lượt soát vé | ~8 GB | ❌ Không |
| `notification_logs` | ~100 triệu bản ghi | ~60 GB | ⚠️ Xét Partitioning |
| `audit_logs` | ~200 triệu bản ghi | ~120 GB | ⚠️ Xét Partitioning |

**Tổng ước tính storage sau 5 năm: ~240 GB** — Hoàn toàn nằm trong giới hạn một MySQL Primary Server với SSD NVMe 2 TB. MySQL 8.0 với InnoDB Buffer Pool 64 GB có thể phục vụ lượng này hiệu quả.

### 2.3. Tại Sao KHÔNG Dùng Sharding? (5 Lý Do Kỹ Thuật)

#### ❌ Lý do 1: Phá vỡ tính toàn vẹn ACID — Mất đi lợi thế lớn nhất của Modular Monolith

Hệ thống vừa chuyển từ Microservices sang **Modular Monolith** để có được `@Transactional` ACID. Nếu bật Sharding:
- Bảng `orders` ở Shard-1 và `payment_transactions` ở Shard-2 → không thể `@Transactional` trong một giao dịch.
- Phải quay lại dùng **Distributed Transaction (XA/2PC)** hoặc Saga Pattern → đúng là vòng tròn trở lại độ phức tạp của Microservices.

```
Modular Monolith + Sharding = Worst of both worlds
  (phức tạp như Microservices, nhưng không có lợi ích scale độc lập)
```

#### ❌ Lý do 2: Truy vấn JOIN liên bảng trở nên bất khả thi

Các truy vấn nghiệp vụ quan trọng cần JOIN nhiều bảng:
```sql
-- Báo cáo doanh thu: JOIN orders × payment_transactions × concerts × ticket_zones
SELECT c.name, tz.zone_name, COUNT(o.id), SUM(p.amount)
FROM orders o
JOIN payment_transactions p ON o.id = p.order_id
JOIN concerts c ON o.concert_id = c.id
JOIN ticket_zones tz ON o.zone_id = tz.id
WHERE p.status = 'PAID'
GROUP BY c.id, tz.id;
```
Nếu các bảng nằm trên các Shard khác nhau → **Cross-Shard JOIN không thể thực thi** → phải pull toàn bộ data về application layer để JOIN → cực kỳ chậm và tốn RAM.

#### ❌ Lý do 3: Phức tạp hóa vận hành gấp bội

| Tác vụ vận hành | Không Sharding | Có Sharding (4 shards) |
|---|---|---|
| Backup database | `mysqldump ticketdb` | Backup 4 DB riêng + merge script |
| Restore khi sự cố | Restore 1 file | Restore 4 files, re-balance |
| Schema migration (ALTER TABLE) | 1 lệnh | 4 lệnh tuần tự, lock phức tạp |
| Monitor slow queries | 1 slow log | 4 slow logs phải tổng hợp |
| Rebalance khi data lệch | Không cần | Phải viết data migration tool |

#### ❌ Lý do 4: Shard Key lựa chọn sai gây Hot Shard

Bài toán bán vé có đặc điểm: **tất cả traffic đổ vào concert_id hot nhất** (ví dụ concert BTS). Nếu Shard theo `concert_id`:
- Shard chứa BTS concert → nhận 90% traffic → **Hot Shard** → sập.
- Các Shard khác → nhàn rỗi → lãng phí.

Không có Shard Key nào phân phối đều cho bài toán flash-sale event.

#### ❌ Lý do 5: 100K CCU là tải có thể xử lý bằng vertical scaling + caching

Với thiết kế đúng:
- **Redis** xử lý 99% read traffic (concert info, seat status, queue).
- **MySQL Primary** chỉ nhận write traffic thuần túy (hold, order, payment).
- Ước tính write QPS thực tế vào MySQL: ~500–2.000 writes/giây (đặt ghế + đơn hàng).
- Một MySQL 8.0 server với NVMe SSD có thể xử lý **50.000–100.000 simple writes/giây**.

### 2.4. Chiến Lược Thay Thế Sharding — Được Áp Dụng

Thay vì Sharding ngang (Horizontal Sharding), hệ thống áp dụng **3 chiến lược tối ưu phù hợp hơn**:

| Chiến lược | Áp dụng cho | Lợi ích |
|---|---|---|
| **1. Table Partitioning** | `audit_logs`, `checkin_logs`, `notification_logs` | Partition Pruning giảm 95% I/O; DROP PARTITION thay TRUNCATE |
| **2. Read Replica (Read/Write Splitting)** | Toàn bộ SELECT queries | Đọc từ 2 Replica, ghi vào Primary → tăng read throughput 3× |
| **3. Redis Cache Layer** | Concert info, seat map, user session | 99% read traffic không chạm MySQL |

---

## 3. Chiến Lược Tối Ưu Hóa Cơ Sở Dữ Liệu (MySQL 8.0 Optimization)

### 3.1. Thiết Kế Chỉ Mục Thông Minh (Targeted & Partial Indexing)

Tránh tạo chỉ mục tràn lan gây chậm thao tác ghi (`INSERT`/`UPDATE`), hệ thống tập trung vào các chỉ mục có tính chọn lọc cao:

```sql
-- 1. Composite Index tìm ghế trống theo concert (sử dụng functional index MySQL 8.0)
CREATE INDEX idx_seats_concert_status ON seats (concert_id, status, zone_id);

-- 2. Composite Index tối ưu hóa truy vấn lịch sử đơn hàng của người dùng
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC, status);

-- 3. Unique Index chống tạo đơn trùng lặp cho cùng một lượt giữ ghế
CREATE UNIQUE INDEX uq_order_hold_token ON orders (hold_token);

-- 4. B-tree Index phục vụ quét vé siêu tốc tại cổng (covering index)
CREATE INDEX idx_tickets_qr ON tickets (qr_code_hash, status, concert_id);

-- 5. Index cho payment IPN webhook lookup
CREATE UNIQUE INDEX uq_payment_txn_ref ON payment_transactions (vnpay_txn_ref);
```

### 3.2. Phân Vùng Dữ Liệu Bảng Lớn (Table Partitioning — Thay Thế Sharding)

Đối với các bảng có khối lượng bản ghi tăng trưởng nhanh, áp dụng **Range Partitioning theo tháng**:

```sql
-- Bảng audit_logs: Phân vùng theo tháng (Range Partitioning)
CREATE TABLE audit_logs (
    id          BIGINT AUTO_INCREMENT,
    user_id     BIGINT NOT NULL,
    action      VARCHAR(100) NOT NULL,
    entity_type VARCHAR(50),
    entity_id   BIGINT,
    ip_address  VARCHAR(45),
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, created_at)  -- Partition key phải trong PK
) 
PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p2026_10 VALUES LESS THAN (202611),
    PARTITION p2026_11 VALUES LESS THAN (202612),
    PARTITION p2026_12 VALUES LESS THAN (202701),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Xóa dữ liệu cũ hơn 5 năm tức thì — không lock bảng, không tốn I/O
ALTER TABLE audit_logs DROP PARTITION p2021_10;

-- Tương tự cho checkin_logs, notification_logs
```

**Lợi ích của Partitioning (so với Sharding):**
- Vẫn trong **một MySQL instance** → ACID transactions hoạt động bình thường.
- Query Planner tự động thực hiện **Partition Pruning** — chỉ quét partition liên quan.
- Xóa data cũ bằng `DROP PARTITION` → tức thì, không block, không gây I/O spike.

### 3.3. Phân Tách Đọc / Ghi (Read/Write Splitting)

```
Writes (INSERT/UPDATE/DELETE)  →  MySQL Primary (1 server)
                                      ↓ (Binlog Replication)
Reads (SELECT - Report, Search) →  MySQL Read Replica 1
                                  →  MySQL Read Replica 2
```

Cấu hình trong Spring Boot với `@Transactional(readOnly = true)`:

```java
// Spring Boot tự động route readOnly=true transactions → Replica
@Transactional(readOnly = true)
public Page<ConcertDto> searchConcerts(ConcertSearchRequest req, Pageable pageable) {
    return concertRepository.findByFilters(req, pageable); // → Read Replica
}

@Transactional  // Default read-write → Primary
public OrderDto createOrder(CreateOrderRequest req) {
    // INSERT orders, INSERT order_items → Primary
}
```

### 3.4. Quản Lý Kết Nối Bằng HikariCP (Connection Pooling)

```yaml
# application.yml — HikariCP configuration
spring:
  datasource:
    hikari:
      maximum-pool-size: 50       # Tối đa 50 kết nối vật lý tới MySQL
      minimum-idle: 10            # Luôn giữ sẵn 10 kết nối
      connection-timeout: 3000    # 3 giây timeout nếu không lấy được connection
      idle-timeout: 600000        # Đóng connection nhàn rỗi sau 10 phút
      max-lifetime: 1800000       # Tái tạo connection sau 30 phút (tránh stale)
      leak-detection-threshold: 5000  # Cảnh báo nếu connection bị giữ > 5 giây
```

**Tại sao 50 connections đủ cho 100K CCU?**
- 100K CCU không có nghĩa 100K requests đồng thời chạm MySQL.
- Nhờ Redis cache, chỉ ~2–5% requests thực sự cần MySQL write.
- Mỗi DB transaction chỉ giữ connection ~5–50ms → 50 connections × (1000ms/50ms) = **1.000 transactions/giây**.

---

## 4. Kiến Trúc Xác Thực & Phân Quyền (Authentication & RBAC)

### 4.1. Cơ Chế Xác Thực Kép (Dual-Token Pattern)

- **Access Token:** JWT chuẩn (RS256 - ký bằng Private Key). Thời hạn sống ngắn (**15 phút**). Mang thông tin phân quyền tối thiểu (`user_id`, `role`, `email`).
- **Refresh Token:** Chuỗi ngẫu nhiên 256 bits, mã hóa băm SHA-256 lưu trong `user_sessions`. Thời hạn sống dài (**7 ngày**).
- **Bảo vệ Cookie trên Web:**
  ```http
  Set-Cookie: refresh_token=abc123xyz; Path=/api/v1/auth; HttpOnly; Secure; SameSite=Strict; Max-Age=604800
  ```
  - `HttpOnly`: JavaScript không thể đọc → miễn nhiễm XSS token theft.
  - `Secure`: Chỉ truyền qua HTTPS.
  - `SameSite=Strict`: Ngăn CSRF hoàn toàn.

### 4.2. Ma Trận Phân Quyền Theo Vai Trò (RBAC)

| Nhóm quyền hạn | Customer | Organizer | Admin | Check-in Staff |
|---|---|---|---|---|
| **Xem concert & Chọn ghế** | ✅ | ✅ | ✅ | ❌ |
| **Giữ ghế & Thanh toán** | ✅ | ❌ | ❌ | ❌ |
| **Tạo & Cập nhật Concert** | ❌ | ✅ Chỉ của mình | ✅ Toàn quyền | ❌ |
| **Phê duyệt & Hủy Concert** | ❌ | ❌ | ✅ | ❌ |
| **Xem Báo cáo Doanh thu** | ❌ | ✅ Chỉ của mình | ✅ Toàn bộ | ❌ |
| **Quét mã QR Check-in** | ❌ | ❌ | ✅ Giám sát | ✅ Cổng được gán |
| **Khóa / Mở khóa Tài khoản** | ❌ | ❌ | ✅ | ❌ |

---

## 5. Các Biện Pháp Bảo Mật Ứng Dụng (Application Security Controls)

### 5.1. Phòng Chống SQL Injection & XSS

- **Zero SQL Injection:** Nghiêm cấm 100% nối chuỗi SQL. Toàn bộ truy vấn dùng **Parameterized Queries (Prepared Statements)** qua Spring Data JPA / Hibernate.
- **Zero XSS:** Toàn bộ input từ người dùng được sanitize và HTML Entity Encoding trước khi render ra giao diện.

### 5.2. Phòng Chống IDOR (Insecure Direct Object Reference)

```java
// ĐÚNG — Luôn kèm authenticated_user_id vào mọi query tài nguyên nhạy cảm
@Transactional(readOnly = true)
public OrderDto getOrder(Long orderId, Long authenticatedUserId) {
    return orderRepository.findByIdAndUserId(orderId, authenticatedUserId)
        .map(OrderDto::from)
        .orElseThrow(() -> new AccessDeniedException("Order not found or access denied"));
}
// → SQL: SELECT * FROM orders WHERE id = ? AND user_id = ?
```

### 5.3. An Toàn Dữ Liệu Thẻ Thanh Toán (PCI-DSS Compliance)

- **HOÀN TOÀN KHÔNG** lưu số thẻ tín dụng đầy đủ (PAN), ngày hết hạn, hoặc mã CVV/CVC.
- Mọi thao tác nhập thẻ được redirect tới cổng thanh toán **PCI-DSS Level 1** (VNPay, MoMo).
- Hệ thống chỉ lưu **mã tham chiếu giao dịch (Transaction Reference)** và trạng thái.

### 5.4. Tường Lửa & Chống DDoS (WAF & Rate Limiting)

- **Cloudflare WAF** tuyến đầu — lọc OWASP Top 10, chặn scanner tự động.
- **API Gateway Rate Limit** — Redis INCR + EXPIRE: > 120 req/phút/IP → 429 Too Many Requests.
- **Cloudflare Turnstile CAPTCHA** — kích hoạt khi phát hiện hành vi bot trước khi vào hàng chờ.
