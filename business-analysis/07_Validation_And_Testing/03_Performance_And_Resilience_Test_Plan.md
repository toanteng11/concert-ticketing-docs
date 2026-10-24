# 03 — Kế Hoạch Kiểm Thử Tải & Khả Năng Phục Hồi (Performance & Resilience Test Plan)

## 1. Giới Thiệu & Mục Tiêu Thử Nghiệm

Mục tiêu của kế hoạch kiểm thử hiệu năng và khả năng phục hồi (**Performance & Resilience Testing Plan**):
1. **Kiểm chứng khả năng chịu tải $100.000\text{ CCU}$:** Đảm bảo hệ sinh thái hạ tầng đám mây (Kubernetes, Redis Cluster, Kafka, PostgreSQL, Kong Gateway) vận hành mượt mà, không sập nguồn khi bước vào giờ mở bán vé sự kiện quy mô lớn.
2. **Đo lường độ trễ chuẩn hóa (Latency Benchmarking):** Xác minh các chỉ số độ trễ P50, P95, P99 đạt đúng cam kết mục tiêu trong tài liệu SLO/SLA (P99 $< 150\text{ ms}$ cho hàng chờ, $< 350\text{ ms}$ cho giữ ghế).
3. **Thực nghiệm kỹ thuật hỗn loạn (Chaos Engineering):** Chủ động đánh sập các thành phần phần cứng/dịch vụ quan trọng (Redis Master, PostgreSQL Primary, Cổng thanh toán) để kiểm chứng khả năng tự phục hồi (Self-Healing) và đảm bảo **Tuyệt đối không bán trùng ghế trong mọi tình huống sự cố**.

---

## 2. Quy Chuẩn Môi Trường Kiểm Thử (Test Environment Setup)

- **Môi trường:** Staging Dedicated Performance Cluster (Cách ly hoàn toàn với môi trường Development).
- **Hạ tầng cụm máy chủ:** Cấu hình phần cứng tương đương **$100\%$ quy mô Production** (12 Worker Nodes `c6g.2xlarge`, Redis Cluster 6 nodes, PostgreSQL Primary 16 vCPU / 64 GB RAM + 2 Read Replicas).
- **Công cụ phát tải phân tán (Distributed Load Generators):**
  - Cụm máy chủ tạo tải gồm 10 máy ảo độc lập chạy công cụ **k6 (Grafana)** và **Locust**.
  - Băng thông mạng máy phát tải: Đạt tối thiểu $10\text{ Gbps}$ ra vào Internet để không bị nghẽn cổ chai tại máy test.
- **Công cụ tiêm sự cố hỗn loạn (Fault Injection):** Sử dụng **Chaos Mesh** cài đặt trực tiếp trong cụm Kubernetes.

---

## 3. Các Hồ Sơ Kiểm Thử Tải (Performance Testing Profiles)

Hệ thống được kiểm thử qua 4 bài kiểm tra tải tiêu chuẩn:

| Loại hình kiểm thử | Mục đích thực hiện | Kịch bản mô phỏng tải | Tiêu chí đánh giá thành công |
|---|---|---|---|
| **1. Baseline Load Test (Tải cơ sở)** | Đo đạc hiệu năng ngày thường | Duy trì ổn định $5.000\text{ CCU}$ trong 2 giờ liên tục | P99 Latency $< 100\text{ ms}$; CPU máy chủ $< 30\%$; Tỷ lệ lỗi $= 0\%$ |
| **2. Stress Test (Thử thách cực hạn)** | Tìm điểm gãy (Breaking Point) của hệ thống | Tăng dần tải từ $10.000\text{ CCU} \rightarrow 150.000\text{ CCU}$ trong 30 phút | Hệ thống không sập toàn bộ; Cơ chế Load Shedding kích hoạt chính xác tại ngưỡng $120.000\text{ CCU}$ |
| **3. Spike / Flash-Sale Test (Sốc tải giờ G)** | Mô phỏng giờ mở bán Mega Concert | Đột biến từ $0 \rightarrow 100.000\text{ CCU}$ chỉ trong vòng **$60\text{ giây}$** đúng thời khắc mở bán | Phòng chờ ảo hấp thụ $100\%$ tải; Không rớt kết nối TCP; Không bán trùng ghế |
| **4. Soak / Endurance Test (Tải ngâm dài hạn)**| Phát hiện rò rỉ bộ nhớ (Memory Leak) | Duy trì $20.000\text{ CCU}$ liên tục trong suốt **$24\text{ giờ}$** | RAM của các Pod không tăng tuyến tính; Không cạn kiệt Connection Pool PgBouncer |

---

## 4. Kịch Bản Kiểm Thử Hiệu Năng Trọng Yếu (Performance Test Scenarios)

| Mã | Tên kịch bản | Mô tả chi tiết thử nghiệm | Chỉ số đo lường chính | Ngưỡng đạt chuẩn (Pass Criteria) |
|---|---|---|---|---|
| **PT-001** | **Xếp hàng phòng chờ $100.000\text{ CCU}$** | $100.000$ client ảo đồng thời gửi request thăm dò vị trí `GET /queue/status` chu kỳ 3 giây/lần ($\approx 32.000\text{ QPS}$) | Thông lượng tiếp nhận (QPS), độ trễ phản hồi, mức sử dụng CPU Redis | Throughput $\ge 32.000\text{ QPS}$; P95 Latency $< 60\text{ ms}$; P99 $< 120\text{ ms}$; Tỷ lệ lỗi $< 0.01\%$ |
| **PT-002** | **Tải sơ đồ ghế $5.000\text{ CCU}$** | $5.000$ người trong khu vực chọn ghế tải cấu trúc sơ đồ khán đài SVG và nhận cập nhật diff trạng thái qua WebSocket/Polling | Thời gian tải trang ban đầu, tỷ lệ trúng bộ đệm Edge CDN | CDN Hit Rate $\ge 98\%$; Thời gian tải sơ đồ $< 300\text{ ms}$ cho $99\%$ người dùng |
| **PT-003** | **Tranh chấp giữ ghế cao độ ($3.000\text{ TPS}$)** | $5.000$ người dùng đồng thời nhấp chọn các ghế trong khu vực VIP (tổng 500 ghế VIP) | Thời gian thực thi Lua Script khóa ghế Redis; Tỷ lệ xung đột; Tính duy nhất | **Số vé bán trùng $= 0$**; P99 Latency khóa ghế $< 350\text{ ms}$; Không có deadlock DB |
| **PT-004** | **Bão Webhook thanh toán ($100\text{ TPS}$)** | Cổng thanh toán giả lập bắn dồn dập $100$ Webhook IPN callback mỗi giây sau khi khách hàng chuyển tiền | Độ trễ xử lý cập nhật trạng thái đơn hàng, độ trễ sinh vé điện tử | Xử lý hoàn tất trong $< 50\text{ ms}$/request; Không bị nghẽn Kafka Consumer Lag |
| **PT-005** | **Giải phóng ghế hết hạn hàng loạt** | Mô phỏng $10.000$ ghế đang giữ cùng hết hạn 10 phút tại cùng một mốc thời gian | Tốc độ quét và thu hồi khóa của tiến trình nền; Tải I/O cơ sở dữ liệu | Toàn bộ $10.000$ ghế chuyển về `AVAILABLE` trong $< 5\text{ giây}$; Không làm nghẽn CPU DB |
| **PT-006** | **Soát vé tại cổng tốc độ cao ($100\text{ TPS}$)** | Giả lập 40 cổng soát vé đồng thời quét mã QR của $50.000$ khán giả đến sân dồn dập | Thời gian phản hồi hiển thị màn hình thiết bị máy quét cầm tay | Thời gian phản hồi $\le 500\text{ ms}$ (P99 $\le 800\text{ ms}$); Tỷ lệ quét thành công $100\%$ |

---

## 5. Kịch Bản Kiểm Thử Khả Năng Phục Hồi & Kỹ Thuật Hỗn Loạn (Resilience & Chaos Scenarios)

| Mã | Thành phần bị tấn công | Kịch bản tiêm lỗi hỗn loạn (Fault Injection) | Hành vi dự kiến của hệ thống | Tiêu chuẩn nghiệm thu phục hồi |
|---|---|---|---|---|
| **RT-001** | **Sập Node Redis Master** | Dùng Chaos Mesh tiêu diệt (Kill Pod) đột ngột node Redis Master đang giữ khóa ghế của Zone Hot nhất | Cụm Redis tự động bầu cử Replica lên làm Master trong vòng 3 giây; PostgreSQL bảo vệ chốt chặn cuối | **Không bán trùng ghế nào**; Hệ thống trở lại bình thường trong $< 5\text{ giây}$ |
| **RT-002** | **Sập PostgreSQL Primary** | Ép tắt nguồn (Hard Reboot) máy chủ Database Master trong lúc đang có $50$ transaction thanh toán dở dang | Patroni tự động phong cấp Read Replica lên làm Master; PgBouncer chuyển hướng kết nối | RPO $\le 5$ giây; RTO $\le 20$ giây; Dữ liệu các đơn hàng đã `COMMIT` được bảo toàn nguyên vẹn $100\%$ |
| **RT-003** | **Chậm mạng Cổng Thanh Toán** | Tiêm độ trễ mạng thêm $3.000\text{ ms}$ vào mọi kết nối gọi ra ngoài VNPay | Circuit Breaker chuyển sang trạng thái `OPEN` sau 20 request bị chậm; Tự động kích hoạt Fallback | Thread pool không bị cạn kiệt; Khách hàng được chuyển hướng sang cổng MoMo mượt mà |
| **RT-004** | **Sập Message Broker Kafka** | Đánh sập ngẫu nhiên 1 trong 3 Broker Kafka trong giờ mở bán vé | 2 Broker còn lại tiếp tục nhận sự kiện bình thường (nhờ Replication Factor = 3, Min In-Sync Replicas = 2) | Không mất mát bất kỳ tin nhắn sự kiện nào (Zero Message Loss) |
| **RT-005** | **Mất mạng cổng soát vé** | Cắt đường truyền Internet của thiết bị máy quét cầm tay tại Cổng A | Ứng dụng Staff chuyển sang chế độ Offline Mode trong $< 2\text{ giây}$; Tiếp tục quét vé hợp lệ bằng SQLite | Đồng bộ dữ liệu ngược lên server ngay khi kết nối lại mà không phát sinh xung đột |
| **RT-006** | **Chết Pod Backend giữa chừng** | Kill Pod của `Order Service` đúng lúc đang xử lý lưu đơn hàng | Kubernetes tự động khởi động Pod mới (Self-healing); Client gửi lại request với cùng `Idempotency-Key` | Không phát sinh đơn hàng trùng lặp; Giao dịch hoàn tất thành công sau khi retry |

---

## 6. Tiêu Chí Hoàn Tất & Điều Kiện Đủ Để Triển Khai Thực Tế (Go-Live Exit Criteria)

Hệ thống chỉ được phép chuyển từ môi trường Staging sang Production (Go-Live) khi thỏa mãn đồng thời **100% các tiêu chí bắt buộc** sau:
1. **Zero Double-Booking:** Không ghi nhận bất kỳ trường hợp bán trùng ghế nào trong toàn bộ các đợt kiểm thử tải đỉnh và thử nghiệm sập node Redis.
2. **Đạt chỉ tiêu thông lượng:** Chịu tải thành công $100.000\text{ CCU}$ trong kịch bản mở bán vé giả lập kéo dài tối thiểu 60 phút.
3. **Đạt chỉ tiêu độ trễ:**
   - Hàng chờ ảo: P99 Latency $< 150\text{ ms}$.
   - Khóa giữ ghế: P99 Latency $< 350\text{ ms}$.
   - Soát vé tại cổng: Phản hồi máy quét $\le 500\text{ ms}$.
4. **Độ ổn định khi phục hồi:** Các bài kiểm tra Chaos Engineering (RT-001 đến RT-006) đều tự phục hồi thành công trong khung thời gian quy định (RTO $< 30$ phút, RPO $< 5$ phút).
5. **Ký duyệt biên bản:** Báo cáo kiểm thử tải được Trưởng nhóm Kiến trúc Hệ thống (Lead Architect), Trưởng nhóm SRE và Quản lý Dự án (Project Manager) cùng ký xác nhận nghiệm thu.
