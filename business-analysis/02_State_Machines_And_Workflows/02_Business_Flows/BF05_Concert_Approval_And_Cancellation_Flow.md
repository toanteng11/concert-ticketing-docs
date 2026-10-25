# BF05 — Business Flow: Phê Duyệt & Hủy Concert (Concert Approval & Cancellation Flow)

## 1. Mục Đích

Tài liệu mô tả luồng nghiệp vụ từ khi Ban tổ chức tạo concert, trải qua quy trình phê duyệt của Admin, cho đến các kịch bản hủy concert từ phía Organizer và Admin.

---

## 2. Luồng Tạo & Phê Duyệt Concert

```mermaid
sequenceDiagram
    actor ORG as Ban tổ chức (Organizer)
    participant ORG_PORTAL as Organizer Portal
    participant CON_API as Concert API
    participant DB as Database
    participant NT as Notification Service
    actor ADMIN as Admin

    ORG->>ORG_PORTAL: Đăng nhập Organizer Portal\nTạo Concert mới

    ORG_PORTAL->>ORG_PORTAL: Nhập thông tin:\n- Tên concert, ngày giờ, địa điểm\n- Upload hình ảnh banner\n- Thiết lập sơ đồ ghế & giá vé\n- Thời gian mở bán

    ORG->>ORG_PORTAL: Click "Gửi phê duyệt"
    ORG_PORTAL->>CON_API: POST /organizer/concerts/{id}/submit-review

    CON_API->>CON_API: Validate:\n- Sơ đồ ghế đã có ≥ 1 zone với giá\n- sale_start_time > current_time\n- event_date > sale_start_time\n- Thông tin pháp lý Organizer đã xác minh

    CON_API->>DB: UPDATE concerts\nSET status = 'PENDING_REVIEW',\nsubmitted_at = SYSUTCDATETIME()

    CON_API->>NT: Gửi thông báo tới Admin
    NT-->>ADMIN: Email + Dashboard Alert:\n"Concert mới cần phê duyệt:\n[Tên Concert] — [Ngày] — [Địa điểm]\nSLA: 48 giờ"

    CON_API-->>ORG_PORTAL: 200 OK — "Hồ sơ đã gửi. Admin sẽ xem xét trong 48 giờ."
    ORG_PORTAL-->>ORG: Trạng thái concert: "Chờ phê duyệt"

    Note over NT,ADMIN: Admin xem xét hồ sơ

    alt Admin Phê Duyệt
        ADMIN->>CON_API: PUT /admin/concerts/{id}/approve\nBody: {notes: "Đã kiểm tra GPKD và hợp đồng"}
        CON_API->>DB: UPDATE concerts SET status = 'PUBLISHED',\napproved_at = SYSUTCDATETIME(), approved_by = admin_id
        CON_API->>NT: Notify Organizer
        NT-->>ORG: Email + Push: "🎉 Concert đã được phê duyệt!\nHiện đang hiển thị công khai. Mở bán lúc [sale_start_time]"

    else Admin Yêu Cầu Chỉnh Sửa
        ADMIN->>CON_API: PUT /admin/concerts/{id}/request-revision\nBody: {reason: "Thiếu giấy phép biểu diễn số ..."}
        CON_API->>DB: UPDATE concerts SET status = 'DRAFT',\nrevision_note = reason
        CON_API->>NT: Notify Organizer
        NT-->>ORG: Email: "Concert cần bổ sung thông tin:\n[Lý do từ Admin]\nVui lòng cập nhật và gửi lại."

    else Admin không xử lý trong 48h (SLA Breach)
        Note over NT: Scheduler phát hiện SLA vi phạm
        NT-->>ADMIN: ⚠️ ALERT: "Concert [tên] chờ duyệt\nquá 48h — Cần xử lý ngay!"
        NT-->>ORG: Email: "Hồ sơ của bạn đang được xem xét.\nChúng tôi sẽ phản hồi trong thời gian sớm nhất."
    end
```

---

## 3. Luồng Tự Động Mở Bán (Auto ON_SALE Transition)

```mermaid
sequenceDiagram
    participant SCHED as Scheduler (Cron Job)
    participant CON_API as Concert API
    participant WR_SVC as Waiting Room Service
    participant DB as Database
    participant CDN as CDN / Edge

    Note over SCHED: Chạy mỗi phút kiểm tra concerts sắp mở bán

    SCHED->>CON_API: CheckSaleStartTime()
    CON_API->>DB: SELECT id FROM concerts\nWHERE status = 'PUBLISHED'\nAND sale_start_time <= DATEADD(MINUTE, 15, SYSUTCDATETIME())\nAND sale_start_time > SYSUTCDATETIME()

    alt Có concert sắp mở bán trong 15 phút
        DB-->>CON_API: [concert_id_list]
        CON_API->>WR_SVC: ActivateWaitingRoom(concert_id)
        WR_SVC->>CDN: Kích hoạt Waiting Room rule tại Edge
        CON_API->>DB: UPDATE concerts SET waiting_room_active = 1
    end

    CON_API->>DB: SELECT id FROM concerts\nWHERE status = 'PUBLISHED'\nAND sale_start_time <= SYSUTCDATETIME()

    alt Có concert đến giờ mở bán
        DB-->>CON_API: [concert_id_list]
        loop Với mỗi concert
            CON_API->>CON_API: Validate:\n- venue_map.all_zones_priced = true\n- contract_confirmed = true
            CON_API->>DB: UPDATE concerts\nSET status = 'ON_SALE', sale_opened_at = SYSUTCDATETIME()
            CON_API->>WR_SVC: TriggerQueueShuffle(concert_id)
        end
    end
```

---

## 4. Luồng Organizer Yêu Cầu Hủy Concert

```mermaid
flowchart TD
    A["Organizer gửi yêu cầu hủy concert<br/>POST /organizer/concerts/{id}/cancel-request"] --> REVIEW["Hiển thị số order PAID và khoản refund dự kiến<br/>Chi phí/điều kiện lấy theo hợp đồng có hiệu lực"]
    REVIEW --> ORG_CONFIRM{"Organizer xác nhận gửi yêu cầu?"}
    ORG_CONFIRM -->|Không| CANCEL_FLOW["Giữ nguyên concert; không đổi dữ liệu"]
    ORG_CONFIRM -->|Có| SEND_TO_ADMIN["Ghi cancellation_requested_at và status_before_cancellation<br/>concert.status = PENDING_CANCELLATION"]
    SEND_TO_ADMIN --> ADMIN_DECISION{"Admin quyết định theo hợp đồng/chính sách đã duyệt"}
    ADMIN_DECISION -->|Duyệt| EXECUTE_CANCEL["concert.status = CANCELLED<br/>Hoàn 100% order.final_amount cho mọi order PAID<br/>theo D-017 và BF04"]
    ADMIN_DECISION -->|Từ chối| KEEP["Khôi phục status_before_cancellation<br/>Ghi audit và thông báo Organizer"]
    EXECUTE_CANCEL --> FINAL["Vô hiệu hóa mọi ticket; giữ check-in log<br/>không mở lại ghế; reconcile refund UNKNOWN"]
```

---

## 5. Luồng Admin Hủy Khẩn Cấp Concert (Emergency Cancellation)

```mermaid
sequenceDiagram
    actor ADMIN as Admin
    participant ADMIN_DASH as Admin Dashboard
    participant CON_API as Concert API
    participant ORDER_SVC as Order Service
    participant EVENT_BUS as Event Bus
    participant NT as Notification Service

    ADMIN->>ADMIN_DASH: Tìm concert → Click "Hủy khẩn cấp"

    ADMIN_DASH-->>ADMIN: Màn xác nhận:\n"THAO TÁC KHÔNG THỂ HOÀN TÁC!\nToàn bộ {N} đơn hàng sẽ được hoàn tiền 100%.\nNhập lý do hủy bắt buộc:"

    ADMIN->>ADMIN_DASH: Nhập lý do + Confirm 2FA

    ADMIN_DASH->>CON_API: POST /admin/concerts/{id}/emergency-cancel\nBody: {reason: "...", admin_2fa_token: "..."}

    CON_API->>CON_API: Xác thực Admin 2FA token\nGhi Audit Log đầy đủ

    CON_API->>CON_API: UPDATE concerts\nSET status='CANCELLED',\ncancelled_by=admin_id,\ncancel_reason=reason,\ncancelled_at=SYSUTCDATETIME()

    CON_API->>EVENT_BUS: Publish ConcertCancelledEvent (Priority HIGH)

    par Hành động song song
        EVENT_BUS->>ORDER_SVC: Process batch refunds\n(Tất cả PAID orders → REFUND_PENDING)
        EVENT_BUS->>NT: Gửi thông báo khẩn đến tất cả khách hàng bị ảnh hưởng
    end

    ORDER_SVC->>ORDER_SVC: Batch update:\n- N eligible orders → REFUND_PENDING (không có trạng thái CANCELLED trung gian); mọi order PAID, kể cả vé CHECKED, được refund 100% theo D-017; log check-in giữ nguyên và không mở lại ghế\n- Tickets → CANCELLED kể cả CHECKED (QR vô hiệu); check-in logs bất biến\n- Không đưa ghế của concert CANCELLED về AVAILABLE

    NT->>NT: Gửi email + SMS + Push đến tất cả khách hàng\n(Sử dụng queue để không làm quá tải email server)

    CON_API-->>ADMIN_DASH: 200 OK\n{cancelled_orders: N, total_refund: X VND}
    ADMIN_DASH-->>ADMIN: "Concert đã hủy. {N} đơn hàng đang được xử lý hoàn tiền."
```

---

## 6. Quy Trình Thiết Lập Sơ Đồ Ghế Của Organizer

```mermaid
flowchart TD
    A["Organizer vào Concert Setup\n→ Tab 'Sơ đồ ghế'"] --> B["Chọn template khán phòng\n(Nhà hát, Sân vận động, Ngoài trời...)"]
    B --> C["Chia phân khu (Zone):\nVIP_A, VIP_B, Standard, Standing"]
    C --> D["Thiết lập giá vé cho từng zone:\n- VIP_A: 2.000.000 VNĐ\n- Standard: 800.000 VNĐ\n- Standing: 500.000 VNĐ"]
    D --> E["Khóa ghế kỹ thuật (BLOCKED):\nGhế che khuất, dành khách mời VIP"]
    E --> F["Preview sơ đồ hoàn chỉnh"]
    F --> G{"Xác nhận sơ đồ?"}
    G -->|Chỉnh lại| C
    G -->|Xác nhận| H["Lưu sơ đồ ghế\n(Không được sửa sau khi concert ON_SALE)\n(BR-CON-002)"]
    H --> I["Sơ đồ ghế đã khóa (frozen)\nSẵn sàng gửi phê duyệt"]
```

