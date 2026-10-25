# 03 — Cơ Chế Chống Bán Trùng Ghế & Chống Thanh Toán Trùng (Double-Booking & Duplicate Payment Prevention)

## 1. Giới Thiệu & Bản Chất Cốt Lõi Của Vấn Đề

Trong các hệ thống thương mại điện tử thông thường, nếu bán quá số lượng một sản phẩm (Over-selling), cửa hàng có thể hủy đơn và đền bù cho khách hàng một món hàng tương đương. Tuy nhiên, trong lĩnh vực bán vé biểu diễn âm nhạc (Concert Ticketing):
- **Tính chất duy nhất không thể thay thế (Unique Physical Asset):** Mỗi số ghế tại một hàng cụ thể trong khán đài chỉ có một vị trí thực tế duy nhất ngoài đời thực (Ví dụ: Ghế VIP 1, Hàng A, Số 12).
- **Rủi ro bạo lực & vi phạm an ninh:** Việc bán trùng một chiếc ghế cho 2 khán giả khác nhau sẽ dẫn đến xung đột trực tiếp tại cửa kiểm soát an ninh hoặc tranh chấp chỗ ngồi bên trong sân vận động, gây thiệt hại nghiêm trọng về uy tín thương hiệu và pháp lý cho Ban tổ chức.

Tài liệu này đặc tả cơ chế kỹ thuật bảo đảm **Tuyệt đối không bán trùng ghế (Zero Double-Booking)** và **Chống thanh toán trùng lặp (Duplicate Payment Prevention)** trong môi trường phân tán chịu tải cao.

---

## 2. Cơ Chế Khóa Giữ Ghế Phân Tán Bằng Redis & Lua Script

Để ngăn chặn hàng nghìn người cùng chọn một ghế tại cùng một mili-giây, hệ thống áp dụng cơ chế khóa phân tán hai tầng (**Two-Tier Distributed Locking**):

### 1. Tầng Khóa Siêu Tốc (Fast In-Memory Lock - Redis)
Mỗi ghế được đại diện bởi một khóa duy nhất trong Redis Cluster:
- Định dạng khóa: `lock:seat:{concert_id}:{seat_id}`
- Giá trị khóa: `user_id:{hold_session_token}`
- Thời gian sống (TTL): Đúng $600\text{ giây}$ (10 phút).

Lệnh thực thi nguyên tử:
```redis
SET lock:seat:con_102:seat_888 "usr_555:tok_abc123" NX EX 600
```
- `NX`: Chỉ thiết lập khóa nếu khóa này **CHƯA TỒN TẠI** (Not eXists). Nếu ghế đã bị người khác chọn trước dù chỉ 1 phần triệu giây, lệnh trả về `nil` (Thất bại) ngay lập tức.
- `EX 600`: Tự động thu hồi khóa sau 600 giây để phòng ngừa trường hợp người dùng đóng trình duyệt hoặc mất kết nối mạng.

### 2. Thu Hồi Khóa An Toàn Bằng Lua Script (Atomic Lock Release)
Nghiêm cấm việc xóa trực tiếp khóa bằng lệnh `DEL` thông thường, vì nếu người dùng A giữ ghế quá 10 phút, khóa tự hết hạn và người dùng B vừa nhận được khóa, việc người dùng A bấm nút "Hủy ghế" có thể vô tình xóa nhầm khóa của người dùng B.

Hệ thống sử dụng đoạn kịch bản **Lua Script** nguyên tử chạy trực tiếp trong Redis để đối chiếu quyền sở hữu trước khi xóa:
```lua
-- KEYS[1]: lock:seat:{concert_id}:{seat_id}
-- ARGV[1]: expected_user_token (usr_555:tok_abc123)
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

---

## 3. Orchestration Giữ Ghế Redis + SQL Server Có Compensation

Redis và SQL Server không tham gia chung một distributed transaction. Luồng đặt ghế là orchestration nhiều bước có compensation: SQL Server conditional update/transaction là nguồn sự thật bền vững; nếu ghi SQL thất bại sau khi lấy Redis lock, service phải release đúng lock token bằng Lua và reconciler đối chiếu các hold còn ACTIVE. Không tuyên bố atomicity tuyệt đối xuyên Redis/SQL.

### Bước 1: Giữ ghế tạm thời
1. **Bước 1.1:** Người dùng bấm chọn 1–4 ghế trên sơ đồ.
2. **Bước 1.2:** Service gọi lệnh Redis `SET NX EX 600` cho toàn bộ các ghế được chọn trong một lệnh Multi-Exec hoặc Lua Script tổng thể. Nếu có bất kỳ ghế nào thất bại $\rightarrow$ Hủy bỏ toàn bộ (All-or-Nothing).
3. **Bước 1.3:** Khi Redis xác nhận khóa thành công, Service ghi bản ghi vào cơ sở dữ liệu SQL Server:
   ```sql
   INSERT INTO seat_holds (id, user_id, seat_id, concert_id, hold_token, expires_at, status)
    VALUES ('00000000-0000-0000-0000-000000000101', '00000000-0000-0000-0000-000000000555', '00000000-0000-0000-0000-000000000888', '00000000-0000-0000-0000-000000000777', 'hold-token-101', DATEADD(SECOND, 600, SYSUTCDATETIME()), 'ACTIVE');
   ```
4. **Bước 1.4:** Hệ thống phát sự kiện WebSocket tới tất cả người dùng khác đang xem sơ đồ ghế: Chuyển màu ghế sang **Màu Vàng (Đang được giữ)**.

### Bước 2: Xác nhận bán trong SQL Server
1. **Bước 2.1:** Khách hàng thanh toán thành công qua cổng thanh toán VNPay/MoMo.
2. **Bước 2.2:** Webhook của cổng thanh toán gọi về hệ thống với trạng thái `SUCCESS`.
3. **Bước 2.3:** Hệ thống mở một Transaction ACID trong SQL Server với mức cô lập `READ COMMITTED`:
   ```sql
   BEGIN TRANSACTION;
   -- 1. Khóa hàng ghế để kiểm tra trạng thái
   SELECT id, status FROM dbo.seats WITH (UPDLOCK, ROWLOCK) WHERE id = '00000000-0000-0000-0000-000000000888';
   
   -- 2. Cập nhật trạng thái ghế sang SOLD vĩnh viễn
   UPDATE seats SET status = 'SOLD', updated_at = SYSUTCDATETIME() WHERE id = '00000000-0000-0000-0000-000000000888';
   
   -- 3. Cập nhật bản ghi giữ ghế sang COMPLETED
   UPDATE seat_holds SET status = 'COMPLETED' WHERE id = '00000000-0000-0000-0000-000000000101';
   
   -- 4. Cập nhật đơn hàng sang PAID và tạo vé điện tử
   UPDATE orders SET status = 'PAID' WHERE id = '00000000-0000-0000-0000-000000000123';
   INSERT INTO tickets (id, order_id, seat_id, qr_code_hash, status)
   VALUES ('00000000-0000-0000-0000-000000000999', '00000000-0000-0000-0000-000000000123', '00000000-0000-0000-0000-000000000888', 'hash_xyz', 'ORDERED');
   COMMIT TRANSACTION;
   ```
4. **Bước 2.4:** Xóa khóa tạm thời trong Redis và cập nhật trạng thái cache ghế sang `SOLD` vĩnh viễn (Màu Đỏ).

---

## 4. Xử Lý Điều Kiện Tranh Chấp Đặc Biệt: Thanh Toán Đến Muộn Sau Khi Hết Hạn Giữ Ghế (Late Payment Race Condition)

Đây là kịch bản ngoại lệ nguy hiểm nhất trong thương mại điện tử bán vé sự kiện:
- **Thời điểm 09:59:** Khách hàng A đang ở cổng thanh toán VNPay và quét mã QR ngân hàng.
- **Thời điểm 10:00:** Đúng 10 phút hết hạn, Redis TTL hết hiệu lực, hệ thống tự động giải phóng ghế `seat_888` về trạng thái `AVAILABLE`.
- **Thời điểm 10:01:** Khách hàng B nhanh tay chọn và giữ thành công ghế `seat_888`.
- **Thời điểm 10:02:** Cổng VNPay gửi Webhook thông báo Khách hàng A đã trừ tiền thành công $2.000.000\text{ VNĐ}$ cho đơn hàng của ghế `seat_888`.

### Quy tắc xử lý đã chốt (D-016, D-032, D-033):
Khi callback/reconciliation xác nhận thanh toán của đơn `ord_A`: hold đã hết hạn nên không fulfill và không hồi sinh ghế.
1. **Kiểm tra tính toàn vẹn ghế:** Hệ thống thực hiện lệnh kiểm tra xem ghế `seat_888` hiện có đang được gắn hợp lệ với `ord_A` hay đã bị gán cho người khác.
2. **Phát hiện xung đột (Conflict Detected):** Ghế đã bị Khách B giữ hoặc mua mất.
3. **Hành động đền bù tự động (Compensating Action):**
   - Không cho phép ghi đè giao dịch của Khách B.
   - Không tạo order mới và không hồi sinh hold. Ghi payment status SUCCESS_UNFULFILLED; order giữ OUT_TIME; sau xác nhận capture, tạo refund 100% số tiền thực thu; retry/reconciliation theo D-016/D-032.
   - Không thay ghế đã được khách khác giữ/mua. Ghi audit, tạo cảnh báo vận hành và thông báo tình trạng giao dịch theo chính sách đã duyệt.
   - Không hứa voucher bồi thường vì voucher ngoài baseline (D-031).

---

## 5. Cơ Chế Chống Thanh Toán Trùng Lặp (Idempotency Key Architecture)

Để ngăn chặn việc khách hàng bị trừ tiền 2 lần do mạng chập chờn, nhấp đúp nút "Thanh toán", hoặc cổng thanh toán gửi Webhook IPN lặp lại:

### 1. Khóa Bất Biến Yêu Cầu (Client Idempotency Key)
- Khi người dùng bấm nút xác nhận thanh toán trên giao diện, Client tự sinh một mã UUID duy nhất:
  ```http
  POST /api/v1/orders/checkout HTTP/1.1
  Idempotency-Key: 7b9f3e2a-8c1d-4f5a-9e2b-1a3c5d7e9f0a
  ```
- API có thể dùng Redis làm cache trạng thái xử lý; SQL Server lưu `orders.idempotency_key` với unique constraint và retention tối thiểu 24 giờ (BR-ORD-008) để bảo đảm sau restart/cache eviction:
  - Nếu key chưa tồn tại: Đánh dấu trạng thái `IN_PROGRESS` và chuyển tiếp yêu cầu vào Backend.
  - Nếu key đang ở trạng thái `IN_PROGRESS`: Từ chối yêu cầu lặp với mã lỗi `409 Conflict: Giao dịch đang được xử lý, vui lòng không gửi lại`.
  - Nếu key đã ở trạng thái `COMPLETED`: Trả về ngay lập tức kết quả đã lưu trong bộ đệm mà không tạo thêm bất kỳ giao dịch nào mới trong cơ sở dữ liệu.

### 2. Xử Lý Webhook Cổng Thanh Toán Bất Biến (Idempotent Webhook Processing)
Các cổng thanh toán (VNPay, MoMo) thường có cơ chế gửi lại Webhook nhiều lần (Retry policy) nếu máy chủ nhận không phản hồi kịp trong 5 giây.
- Khi nhận Webhook, hệ thống kiểm tra trường `transaction_code` của đối tác trong bảng `payment_transactions`.
- Sử dụng ràng buộc duy nhất trong cơ sở dữ liệu (**Database Unique Constraint**):
  ```sql
  ALTER TABLE payment_transactions ADD CONSTRAINT uk_provider_transaction_id UNIQUE (provider, provider_transaction_id);
  ```
- Nếu bản ghi đã tồn tại và đơn hàng đã ở trạng thái `PAID`: Hệ thống bỏ qua bước xử lý trừ vé, ngay lập tức trả về mã phản hồi thành công `HTTP 200 / RspCode: 00` cho Cổng thanh toán để đối tác dừng gửi lại.

<!-- BEGIN MERMAID DIAGRAM NFR-BOOKING-PAYMENT -->
### Trình tự giữ ghế, checkout và callback thanh toán

```mermaid
sequenceDiagram
  actor C as Customer
  participant API as Booking / Order API
  participant R as Redis
  participant DB as SQL Server Primary
  participant PG as VNPay or MoMo
  participant Q as Message Broker
  C->>API: Hold seats with queue token
  API->>R: SET NX EX 600 with hold token
  alt lock acquired
    API->>DB: Conditional seat update and seat_holds transaction
    alt SQL commit succeeds
      DB-->>API: Hold persisted
      API-->>C: Hold confirmed; expiry fixed at first selection + 600s
    else SQL write fails after Redis lock
      API->>R: Lua release only if lock token matches
      API-->>C: Retryable failure; reconciler checks ACTIVE holds
    end
  else seat already locked
    API-->>C: 409 seat unavailable
  end
  C->>API: Checkout with Idempotency-Key
  API->>DB: Create PENDING order and payment attempt
  API->>PG: Create payment request
  PG-->>C: Redirect / payment result
  PG->>API: Signed webhook or IPN
  API->>API: Verify signature, amount, provider transaction and idempotency
  alt verified success and fulfillment is possible
    API->>DB: Commit payment, order PAID, seats SOLD, tickets issued
    API->>Q: Publish notification event after commit
  else duplicate callback
    API-->>PG: Idempotent acknowledgement
  else payment arrives after hold expired
    API->>DB: Record SUCCESS_UNFULFILLED
    Note over API,DB: Late-payment compensation is a 100% refund after confirmed capture; no hold revival (D-016).
  end
```
<!-- END MERMAID DIAGRAM NFR-BOOKING-PAYMENT -->
