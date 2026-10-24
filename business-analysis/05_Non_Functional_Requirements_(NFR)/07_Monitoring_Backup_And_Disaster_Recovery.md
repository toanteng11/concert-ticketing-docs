# 07 — Giám Sát, Sao Lưu & Phục Hồi Thảm Họa (Monitoring, Backup & Disaster Recovery)

## 1. Giới Thiệu & Mục Tiêu

Một nền tảng bán vé sự kiện âm nhạc quy mô lớn không thể duy trì độ tin cậy nếu thiếu đi:
1. **Khả năng quan sát toàn diện (Full-Stack Observability):** Phát hiện sớm các bất thường về lưu lượng, độ trễ và tài nguyên trước khi khách hàng cảm nhận được sự cố.
2. **Chiến lược sao lưu đa lớp (Multi-Layer Backup Strategy):** Bảo vệ toàn vẹn lịch sử giao dịch và tài chính khỏi rủi ro hỏng hóc phần cứng, lỗi phần mềm hoặc tấn công mã độc tống tiền (Ransomware).
3. **Kế hoạch phục hồi sau thảm họa (Disaster Recovery - DR):** Đảm bảo khôi phục toàn bộ hệ sinh thái trong thời gian ngắn nhất khi cả một trung tâm dữ liệu đám mây (Cloud Region) bị sập.

Tài liệu này đặc tả chi tiết kiến trúc giám sát, quy trình sao lưu và các kịch bản diễn tập phục hồi thảm họa thực tế.

---

## 2. Kiến Trúc Giám Sát Ba Trụ Cột (The Three Pillars of Observability)

Hệ thống triển khai mô hình quan sát thống nhất chuẩn mở (**OpenTelemetry**):

### 1. Trụ Cột 1: Thu Thập Chỉ Số Định Lượng (Metrics - Prometheus & Grafana)
- **Hạ tầng thu thập:** Prometheus Operator chạy trong Kubernetes thu thập (scrape) số liệu mỗi **15 giây** từ ứng dụng Modular Monolith, Ingress Controller, Redis và MySQL 8.0 Exporters.
- **Bảng điều khiển trực quan (Dashboards):**
  - **Executive Business Dashboard:** Doanh số bán vé theo phút, tổng số vé giữ thành công, số người đang trong phòng chờ, tỷ lệ thanh toán thành công qua VNPay/MoMo.
  - **SRE & Infra Dashboard:** Tải CPU/RAM từng node, độ dài hàng đợi Kafka Lag, tỷ lệ hit cache Redis, số lượng kết nối HikariCP / ProxySQL.
  - **Application Performance Dashboard:** Thông lượng QPS, phân vị độ trễ (P50, P95, P99), tỷ lệ mã lỗi HTTP (2xx, 4xx, 5xx).

### 2. Trụ Cột 2: Thu Thập Nhật Ký Tập Trung (Logging - FluentBit & OpenSearch)
- **Nguyên tắc cấu trúc:** Mọi log ứng dụng bắt buộc phải xuất ra chuẩn **JSON có cấu trúc (Structured Logging)** qua `stdout/stderr`.
- **Trường thông tin bắt buộc trong mỗi log line:**
  - `timestamp`: Thời điểm chính xác theo chuẩn ISO 8601 UTC.
  - `trace_id`: Định danh vết duy nhất xuyên suốt ứng dụng Modular Monolith.
  - `level`: `INFO`, `WARN`, `ERROR`, `FATAL`.
  - `service_name`: Tên Microservice phát sinh log.
  - `user_id` / `order_id`: Định danh ngữ cảnh nghiệp vụ (nếu có).
  - `message`: Nội dung mô tả sự kiện.
- **Lưu trữ & Tìm kiếm:** FluentBit thu thập và đẩy vào cụm **OpenSearch Cluster**, lưu trữ nóng 30 ngày phục vụ điều tra lỗi nhanh.

### 3. Trụ Cột 3: Theo Vết Phân Tán (Distributed Tracing - OpenTelemetry & Jaeger)
- Khi một yêu cầu từ người dùng chạm vào API Gateway, hệ thống tự động sinh một mã `traceparent` (chuẩn W3C Trace Context) và truyền qua tiêu đề HTTP tới tất cả các dịch vụ nội bộ tiếp theo.
- Cho phép kỹ sư xem sơ đồ chuỗi thời gian chi tiết của một giao dịch: Gateway (10ms) $\rightarrow$ Booking Service (30ms) $\rightarrow$ Redis Lock (5ms) $\rightarrow$ Database Insert (15ms). Dễ dàng định vị chính xác điểm nghẽn độ trễ khi có sự cố.

---

## 3. Hệ Thống Cảnh Báo Chủ Động (Alerting Rules & Escalation Matrix)

Alertmanager được cấu hình để gửi thông báo tức thì qua **Slack (Kênh kỹ thuật)** và cuộc gọi khẩn cấp **PagerDuty** cho kỹ sư trực ca (On-Call SRE):

| Mã cảnh báo | Điều kiện kích hoạt | Mức độ khẩn cấp | Kênh thông báo | Hành động xử lý tức thì |
|---|---|---|---|---|
| **ALERT-5XX-SPIKE** | Tỷ lệ lỗi `HTTP 5xx > 1%` trong 2 phút liên tiếp | 🔴 P1 - Critical | PagerDuty Call + SMS | Bật Circuit Breaker; Kiểm tra log lỗi tại OpenSearch |
| **ALERT-LATENCY-P99** | Độ trễ API P99 `> 1.000 ms` trong 3 phút | 🔴 P1 - Critical | PagerDuty Call | Kiểm tra Database Locks và mở rộng số Pods tự động |
| **ALERT-REDIS-OOM** | Mức sử dụng RAM Redis `> 85%` | 🔴 P1 - Critical | PagerDuty Call | Xóa bớt cache phụ; Mở rộng RAM cụm Redis |
| **ALERT-KAFKA-LAG** | Consumer Lag của `order.paid` `> 5.000 msgs` | 🟠 P2 - High | Slack `@sre-team` | Tăng số lượng Worker Pods để tiêu thụ nhanh |
| **ALERT-GATEWAY-DOWN** | Cổng VNPay/MoMo trả về lỗi `> 30%` trong 1 phút | 🟠 P2 - High | Slack `@payment-team` | Kích hoạt Fallback sang cổng thanh toán dự phòng |
| **ALERT-DISK-WARN** | Dung lượng ổ đĩa SSD máy chủ `> 80%` | 🟡 P3 - Warning | Slack Alert | Dọn dẹp tệp tạm; Tăng dung lượng đĩa EBS |

---

## 4. Chiến Lược Sao Lưu Dữ Liệu (Backup Architecture)

Để đảm bảo mục tiêu **RPO $\le 5$ Phút (Recovery Point Objective)** và **RTO $\le 30$ Phút (Recovery Time Objective)**:

```
[MySQL 8.0 Primary Database]
   │
   ├── 01:00 AM Hàng Ngày ──→ [Full Snapshot EBS / pg_dump] ──→ AWS S3 (Standard)
   │                                                               │
   └── Mỗi 5 Phút ───────────→ [WAL Continuous Archiving]   ──→ AWS S3 (Cross-Region)
                                                                   │
                                                                   ↓
                                                       [AWS S3 Singapore (ap-southeast-1)]
                                                                   │ (Tự động nhân bản)
                                                                   ↓
                                                       [AWS S3 Tokyo (ap-northeast-1)]
```

### Chi Tiết Tần Suất & Thời Gian Lưu Trữ Bản Sao (Backup Schedule)

| Loại dữ liệu | Phương thức sao lưu | Tần suất thực hiện | Thời gian lưu giữ bản sao (Retention) | Vị trí lưu trữ vật lý |
|---|---|---|---|---|
| **Cơ sở dữ liệu MySQL 8.0** | Full Snapshot + WAL Archiving liên tục | Full: 1 lần/ngày; WAL: Mỗi 5 phút | 30 ngày gần nhất | S3 Bucket đa vùng (Multi-Region) |
| **Khóa & Bộ Đệm Redis** | RDB Snapshot + File AOF liên tục | RDB: Mỗi 6 giờ; AOF: Mỗi 1 giây | 7 ngày | Ổ cứng SSD gắn kèm cụm máy chủ |
| **Tài liệu & Vé S3** | Cross-Region Replication (CRR) | Đồng bộ sự kiện thời gian thực | 5 năm (Luật Kế toán Việt Nam) | S3 Standard chuyển dần sang Glacier |
| **Cấu hình Hạ tầng & K8s** | GitOps (ArgoCD Repository) | Mỗi khi có commit mới | Vĩnh viễn (Git History) | GitHub Private Enterprise |

---

## 5. Quy Trình Phục Hồi Sau Thảm Họa (Disaster Recovery Playbook)

### 1. Phục Hồi Dữ Liệu Về Một Thời Điểm Quá Khứ (Point-In-Time-Recovery - PITR)
- **Áp dụng khi:** Lỗi phần mềm do deploy nhầm phiên bản làm sai lệch dữ liệu giao dịch hoặc kẻ xấu xóa nhầm bảng lúc $14:32$.
- **Quy trình:**
  1. Khởi động một máy chủ MySQL 8.0 tạm thời độc lập.
  2. Nạp bản Full Snapshot của đêm hôm trước ($01:00\text{ AM}$).
  3. Áp dụng tuần tự các tệp nhật ký giao dịch ghi trước (WAL Logs) được lưu trữ trên S3 đến đúng thời khắc **$14:31:59$** (1 giây trước khi sự cố xảy ra).
  4. Xác minh tính toàn vẹn dữ liệu và chuyển đổi ứng dụng sang cơ sở dữ liệu đã phục hồi. Thời gian hoàn tất: $< 20\text{ phút}$.

### 2. Kịch Bản Sập Hoàn Toàn Vùng Đám Mây Chính (Primary Region Outage Failover)
- **Tình huống:** Toàn bộ trung tâm dữ liệu AWS Singapore (`ap-southeast-1`) bị mất điện hoặc đứt cáp viễn thông quốc tế.
- **Kế hoạch ứng phó (Multi-Region Cold/Warm Standby tại AWS Tokyo `ap-northeast-1`):**
  1. **Bước 1 (0 – 5 phút):** Đội ngũ SRE xác nhận sự cố toàn vùng thông qua trạng thái Cloud Status Dashboard.
  2. **Bước 2 (5 – 15 phút):** Kích hoạt hệ thống tại Tokyo thông qua công cụ tự động hóa Terraform / Ansible: Khởi tạo cụm Kubernetes từ cấu hình GitOps.
  3. **Bước 3 (15 – 25 phút):** Khôi phục cơ sở dữ liệu MySQL 8.0 từ các tệp WAL và Snapshot đã được nhân bản tự động sang vùng Tokyo.
  4. **Bước 4 (25 – 30 phút):** Cập nhật bản ghi DNS tại Cloudflare (Global Anycast Traffic Director), chuyển toàn bộ lưu lượng tên miền `ticket.example.com` sang địa chỉ IP của cụm Tokyo.
  5. **Hoàn tất:** Người dùng truy cập lại bình thường với dữ liệu bị tổn thất tối đa không quá 5 phút giao dịch gần nhất (đạt chỉ tiêu RPO $\le 5$ phút, RTO $\le 30$ phút).

---

## 6. Diễn Tập Sự Cố & Kỹ Thuật Hỗn Loạn (Chaos Engineering & DR Drills)

- **Nguyên tắc:** Một kế hoạch phục hồi thảm họa chưa từng được diễn tập thực tế là một kế hoạch vô giá trị.
- **Lịch diễn tập bắt buộc (Game Day):** Định kỳ **6 tháng một lần**, đội ngũ kỹ thuật tổ chức ngày hội diễn tập sự cố (Game Day) trên môi trường Staging/Production:
  - Chủ động bắn hạ ngẫu nhiên một node Redis Master hoặc MySQL 8.0 Replica trong lúc chạy kịch bản mô phỏng $50.000\text{ CCU}$ (sử dụng công cụ Chaos Mesh / Gremlin).
  - Mô phỏng ngắt kết nối mạng tới Cổng thanh toán để kiểm tra khả năng chuyển đổi mạch (Circuit Breaker).
  - Đo đạc thực tế chỉ số MTTR (Mean Time to Recover) và cập nhật bổ sung quy trình vận hành tiêu chuẩn (SOP).

