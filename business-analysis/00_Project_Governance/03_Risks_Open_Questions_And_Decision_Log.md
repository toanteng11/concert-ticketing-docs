# Rủi Ro, Câu Hỏi Mở và Nhật Ký Quyết Định

## Mục đích

Tài liệu này tập hợp các rủi ro nghiệp vụ và kỹ thuật đã được nhận diện, các câu hỏi còn chưa có câu trả lời rõ ràng, và nhật ký ghi lại những quyết định thiết kế quan trọng đã được thống nhất. Mục tiêu là ghi lại baseline nghiệp vụ/kiến trúc đã được chọn theo yêu cầu cập nhật BA, cùng chủ sở hữu và cổng xác nhận cần hoàn tất trước go-live. Quyết định trong tài liệu này chưa thay thế phê duyệt pháp lý, ngân sách hay giấy phép của tổ chức.

---

## 1. Rủi Ro Đã Nhận Diện

| ID | Rủi ro | Mức độ tác động | Xác suất | Biện pháp giảm thiểu / Phản hồi |
|---|---|---|---|---|
| R-001 | Lưu lượng tăng đột biến khi mở bán vé (~100.000 CCU đồng thời) | Rất cao | Cao | Áp dụng phòng chờ (waiting room), rate limiting, load balancer và scale ngang (horizontal scaling) tự động |
| R-002 | Hai khách hàng cùng lúc chọn và giữ thành công cùng một ghế (race condition) | Rất cao | Trung bình | Redis owner-token lock là lớp tăng tốc; SQL Server conditional UPDATE/row lock trên `seats.status = AVAILABLE` là nguồn quyết định cuối. Nếu DB ghi thất bại, release đúng Redis token và reconciler kiểm tra hold. |
| R-003 | Payment callback bị retry hoặc đến trễ sau khi ghế đã hết hạn giữ | Cao | Trung bình | Sử dụng Idempotency Key; xây dựng luồng xử lý late-payment riêng biệt với reconciliation |
| R-004 | Giao dịch thanh toán thất bại khi người dùng bấm nút nhiều lần hoặc mạng gián đoạn | Cao | Cao | Client gửi kèm idempotency key; server lưu kết quả theo key và trả lại kết quả cũ nếu gọi lại |
| R-005 | Giao hàng thông báo bị thất bại (email, push notification) | Trung bình | Trung bình | RabbitMQ durable queues, retry có giới hạn, DLQ, idempotent consumer; luôn lưu thông báo trong hộp thư IN_APP và hiển thị trạng thái provider |
| R-006 | Mã QR được sử dụng tại nhiều cổng vào cùng một lúc (duplicate check-in) | Rất cao | Thấp | Chuyển trạng thái vé sang `CHECKED` bằng atomic operation; từ chối mọi lần quét tiếp theo với thông báo "đã check-in" |
| R-007 | Redis bị lỗi hoặc khởi động lại (cache miss toàn bộ) | Cao | Thấp | Fallback sang DB-level lock; rate limit dùng bộ nhớ local cache tạm thời; vẫn đúng nhưng chậm hơn |
| R-008 | Database Primary bị ngừng hoạt động trong giờ cao điểm | Rất cao | Rất thấp | Dừng/từ chối mọi thao tác cần ghi cho tới khi Primary/AG failover hoàn tất; không xếp giao dịch ghi vào queue để tự replay. Sau khôi phục, đối soát payment UNKNOWN với provider trước khi cho retry; chỉ đọc từ readable secondary cho truy vấn chấp nhận eventual consistency. |
| R-009 | Ban tổ chức không gửi thông tin sơ đồ ghế đúng định dạng | Trung bình | Trung bình | Validation schema nghiêm ngặt khi tạo concert; hiển thị lỗi rõ ràng cho organizer |
| R-010 | Admin không xử lý yêu cầu duyệt concert trong 48 giờ | Trung bình | Thấp | Alerting tự động nhắc nhở admin; hệ thống ghi nhận SLA violation |
| R-011 | Chi trả Organizer theo điều khoản thương mại chưa ký hoặc chưa đối soát đầy đủ | Cao | Trung bình | Chặn payout khi thiếu snapshot hợp đồng, reconciliation chưa MATCHED, còn UNKNOWN/discrepancy hoặc thiếu bằng chứng; Legal/Finance phê duyệt điều khoản trước go-live (D-034) |

---

## 2. Các Câu Hỏi Đã Được Giải Quyết Cho Baseline

Các kết luận dưới đây được áp dụng đồng bộ cho bản BA này theo yêu cầu của chủ dự án ngày 2026-10-25. Chúng là baseline thiết kế/nghiệm thu của tài liệu; các cổng ký duyệt pháp lý, tài chính, ngân sách và giấy phép được ghi riêng, không được hiểu là đã hoàn tất ngoài thực tế.

| ID | Chủ đề ban đầu | Kết luận baseline | Trạng thái |
|---|---|---|---|
| Q-001 | Định nghĩa hoạt động và timeout phiên chọn ghế | Hoạt động là thao tác UI hoặc heartbeat xác thực khi trang chọn ghế ở foreground; client gửi heartbeat mỗi 30 giây. WebSocket còn kết nối tự nó không gia hạn phiên. Sau 120 giây kể từ heartbeat/thao tác hợp lệ cuối, thu hồi quyền vào khu chọn ghế; seat hold đã tạo vẫn đủ 600 giây. | Đã giải quyết — D-015 |
| Q-002 | Payment thành công sau khi hold hết hạn | Không tạo đơn mới và không hồi sinh hold. Ghi SUCCESS_UNFULFILLED; sau khi đối soát xác nhận đã thu tiền, tự động hoàn 100% số tiền thực thu, dùng khóa idempotency và retry theo D-032. | Đã giải quyết — D-016 |
| Q-003 | Hủy concert khi vé đã CHECKED | Nếu concert bị hủy trước trạng thái COMPLETED, hoàn 100% order.final_amount cho mọi đơn PAID, kể cả vé CHECKED; vô hiệu hóa toàn bộ vé, giữ nguyên check-in log và không mở lại ghế của concert đã hủy. Concert đã COMPLETED không được chuyển sang CANCELLED. | Đã giải quyết — D-017; cần Legal/Finance ký chính sách trước go-live |
| Q-004 | Nhiều đơn trên cùng concert | Cho phép nhiều đơn tuần tự. Mỗi tài khoản có tối đa 4 vé đang hiệu lực/đang giữ cho một concert và chỉ có một hold/đơn PENDING hoặc payment attempt chưa rõ kết quả tại một thời điểm. Vé đã hủy và hoàn tiền hoàn tất không tính vào giới hạn. | Đã giải quyết — D-018 |
| Q-005 | Làm tròn 50% và lỗi hoàn tiền | VND xử lý số nguyên; hoàn 50% trên tổng giá vé đủ điều kiện của toàn đơn, làm tròn half-up đến 1 VND một lần. Timeout/UNKNOWN phải đối soát, không retry mù. Retry tối đa 3 lần chỉ sau lỗi thất bại xác định; sau đó chuyển hàng đợi xử lý thủ công theo D-019/D-032. | Đã giải quyết — D-019 |
| Q-006 | Check-in khi mất mạng | Đã gộp vào Q-010. Baseline không có offline check-in; khi không thể xác thực online, scanner từ chối xác nhận và staff chuyển sang quy trình xử lý sự cố tại địa điểm. | Đã giải quyết cùng Q-010 — D-023 |
| Q-007 | Kênh báo hết vé | Thông báo bắt buộc trong ứng dụng và cập nhật realtime cho người đang ở waiting room; gửi push nếu người dùng bật quyền. Email/SMS không phải kênh bắt buộc cho sold-out. | Đã giải quyết — D-020 |
| Q-008 | Organizer sửa concert đã PUBLISHED nhưng chưa ON_SALE | Cho phép sửa nội dung mô tả/media không ảnh hưởng giao dịch; đổi ngày/giờ, địa điểm, sơ đồ ghế, zone hoặc giá phải được Admin duyệt lại và thông báo; sau ON_SALE khóa các trường thương mại/lịch/địa điểm. | Đã giải quyết — D-021 |
| Q-009 | 100.000 hay 500.000 CCU | 100.000 CCU là mục tiêu baseline bắt buộc của đợt này; 500.000 CCU là mục tiêu tương lai, không dùng để sizing hoặc nghiệm thu baseline. | Đã giải quyết — D-022 |
| Q-010 | Phạm vi check-in offline | Offline QR check-in nằm ngoài baseline. Online validation là yêu cầu nghiệm thu; khi mất kết nối, hệ thống fail-closed, ghi nhận sự cố và không chấp nhận vé chỉ từ cache cục bộ. | Đã giải quyết — D-023 |
| Q-011 | Mốc 3/7 ngày hoàn tiền | Dùng chênh lệch ngày lịch tại múi giờ địa điểm: event_local_date - request_local_date. Còn từ 7 ngày lịch trở lên: 100%; từ 3 đến 6 ngày lịch: 50%; 0–2 ngày lịch: 0%. Ranh giới được xác định theo ngày địa phương, không theo giờ trôi qua. | Đã giải quyết — D-024 |
| Q-012 | RabbitMQ hay Kafka | RabbitMQ là broker baseline cho job/event bất đồng bộ và notification; không triển khai Kafka trong baseline. Message dùng durable queue, publisher confirms, retry/DLQ và consumer idempotent. | Đã giải quyết — D-025 |
| Q-013 | SQL Server version, edition, HA/backup | Production baseline dùng SQL Server 2025 Enterprise với Always On Availability Groups: Primary và readable synchronous secondary trong cùng region; asynchronous DR replica ở region độc lập. RPO/RTO, backup và restore test theo D-026; Finance/DBA phải xác nhận giấy phép và sizing trước go-live. | Đã giải quyết thiết kế — D-026; còn cổng ngân sách/DBA trước go-live |
| Q-014 | 99.95% và 99.99% availability | SLO baseline duy nhất là 99.95% mỗi tháng cho critical first-party purchase APIs; đo theo phút khả dụng, báo cáo riêng outage của payment provider. Mục tiêu 99.99% bị loại khỏi baseline; chưa tuyên bố SLA hợp đồng. | Đã giải quyết — D-027 |
| Q-015 | Partition SQL Server | Không partition vật lý trong baseline. Dùng index và retention/archival; chỉ thiết kế lại nếu benchmark chứng minh cần. Không dùng HASH(user). Nếu đạt ngưỡng, chỉ đánh giá RANGE theo created_at cho bảng lịch sử append-only và review FK/index trước khi triển khai. | Đã giải quyết — D-028 |
| Q-016 | NoSQL cho behavior/queue history | Không triển khai NoSQL riêng trong baseline. Redis giữ queue state ngắn hạn; SQL Server giữ audit và transactional outbox để retry/replay tối đa 7 ngày; ELK chỉ giữ operational logs 30 ngày, đã lọc PII và không làm nguồn sự thật. | Đã giải quyết — D-029 |
| Q-017 | Benchmark và xác nhận sizing/licensing | 100.000 CCU là release gate: phải chạy performance/resilience test trên topology production-like, đạt ngưỡng trong test plan, không double-booking/duplicate charge, và có sign-off sizing/licensing DBA/Finance. Tài liệu hiện chỉ đặt tiêu chí, không khẳng định benchmark đã chạy. | Đã giải quyết thành release gate — D-030 |
| Q-018 | Voucher có trong baseline không | Voucher, mã giảm giá và báo cáo chiến dịch voucher không thuộc baseline. discount_amount bằng 0; API không nhận voucher_code; bảng vouchers và các tiêu chí voucher bị loại khỏi migration/UAT hiện tại. | Đã giải quyết — D-031 |
| Q-019 | Refund timeout/UNKNOWN | Refund timeout chuyển UNKNOWN; đối soát tại 30 giây, 5 phút và 30 phút. Không gửi refund lần nữa khi kết quả chưa rõ. Chỉ retry khi provider xác nhận thất bại xác định, tối đa 3 lần với backoff 1/5/30 phút; vẫn UNKNOWN sau 30 phút thì vào manual queue, không tự gửi lệnh hoàn thứ hai. | Đã giải quyết — D-032 |
| Q-020 | Payment create/callback timeout | Payment timeout chuyển UNKNOWN; đối soát tại 30 giây, 5 phút và 30 phút. Không tạo attempt mới khi kết quả chưa rõ; cho phép tối đa 3 payment attempts/order chỉ sau khi provider xác nhận attempt trước thất bại/đã void. UNKNOWN quá 30 phút chuyển support/reconciliation queue. | Đã giải quyết — D-033 |
| Q-021 | Điều khoản phí, cách giữ/chuyển tiền và kỳ quyết toán Organizer | Baseline có sổ settlement; tỷ lệ/căn cứ tính, phí cố định/refund, thời gian chờ, custody/escrow và rail payout lấy từ hợp đồng đã ký. Không giả định escrow hay chuyển khoản tự động; chỉ ghi PAID sau đối soát, phê duyệt và bằng chứng chuyển tiền. | Đã giải quyết ở mức thiết kế; hợp đồng và Legal/Finance sign-off còn là go-live gate — D-034 |

## 3. Nhật Ký Quyết Định (Decision Log)

D-001–D-034 là baseline quyết định được ghi nhận theo yêu cầu cập nhật của chủ dự án. Các cổng xác nhận bên ngoài (Legal/Finance/DBA/licensing) vẫn phải hoàn tất trước go-live như ghi tại từng quyết định.

| ID | Ngày | Quyết định | Lý do | Chủ sở hữu |
|---|---|---|---|---|
| D-001 | — | Mỗi tài khoản có tối đa **4 ghế trong một hold group**; giới hạn tổng vé đang hiệu lực theo concert được làm rõ tại D-018. | Ngăn chặn độc chiếm tồn kho và bảo đảm công bằng | Product Owner |
| D-002 | — | Thời gian giữ ghế cố định **10 phút (600 giây)** từ ghế đầu tiên trong hold group; chọn thêm ghế/checkout/heartbeat không gia hạn. | Đủ thời gian thanh toán và giữ tồn kho hữu hạn | Product Owner |
| D-003 | — | Vé QR là **đơn dụng** (single-use), chỉ check-in được đúng một lần | Ngăn chặn tái sử dụng vé đã check-in, bảo đảm tính toàn vẹn của sự kiện | Product Owner |
| D-004 | — | Mọi yêu cầu thanh toán phải là **idempotent** thông qua Idempotency Key | Ngăn trừ tiền hai lần khi client retry hoặc mạng gián đoạn | Product Owner / Payment team |
| D-005 | — | Hủy tự nguyện: **100%** nếu còn ≥7 ngày lịch; **50%** nếu còn 3–6 ngày lịch; **0%** nếu còn 0–2 ngày lịch, tính theo ngày địa phương tại địa điểm sự kiện. Mức 50% làm tròn half-up đến 1 VND trên tổng giá vé đủ điều kiện; phí dịch vụ/thuế không hoàn theo yêu cầu tự nguyện. | Quy tắc xác định được tại mọi ranh giới ngày và số tiền VND | BA baseline; Legal/Finance ký trước go-live |
| D-006 | — | Hệ thống ưu tiên **CP (Consistency + Partition Tolerance)** trong mô hình CAP | Tính nhất quán dữ liệu ghế và giao dịch quan trọng hơn tính khả dụng tức thì khi có lỗi phân vùng | Kiến trúc sư hệ thống |
| D-007 | — | Dùng **hai lớp bảo vệ** chống bán trùng ghế: Redis `SET NX EX` (lớp 1) + DB constraint / pessimistic lock (lớp 2) | Redis có thể mất lock khi bị lỗi; DB là lớp bảo vệ cuối cùng đảm bảo tính nhất quán tuyệt đối | Kiến trúc sư hệ thống |
| D-008 | — | Thông báo/job bất đồng bộ đi qua **RabbitMQ** durable queues, publisher confirms, retry/DLQ; consumer idempotent. | Không chặn luồng thanh toán đồng bộ; xử lý lại an toàn | Kiến trúc sư hệ thống |
| D-009 | — | Giờ mở bán phải cách ngày được duyệt **ít nhất 1 ngày** | Đảm bảo Admin có đủ thời gian duyệt và hệ thống chuẩn bị hạ tầng cho đợt bán | Product Owner |
| D-010 | — | Admin phải xử lý yêu cầu duyệt concert **trong vòng 48 giờ** | Đảm bảo ban tổ chức không bị kẹt chờ duyệt quá lâu, ảnh hưởng đến kế hoạch bán vé | Product Owner |
| D-011 | 2026-10-25 | Microsoft SQL Server là RDBMS giao dịch. Production baseline: SQL Server 2025 Enterprise, Always On AG, readable secondary; local development dùng Developer edition. | Đáp ứng transaction ACID và topology đọc/HA đã chọn; giấy phép production là go-live gate | Product Owner / Kiến trúc sư / DBA |
| D-012 | 2026-10-25 | Thời gian giữ ghế vẫn là **10 phút (600 giây)** theo D-002; Redis lock không được tự giải phóng ở 9 phút 30 giây. | Giữ nguyên quy tắc nghiệp vụ hiện hành | Product Owner |
| D-013 | 2026-10-25 | Tách read/write connection pools; sau thao tác ghi, các truy vấn cần read-your-write được định tuyến về Primary tối đa 5 giây. Cache TTL 5 giây không được coi là bảo đảm nhất quán. | Tránh đọc trạng thái cũ từ secondary/cache | Kiến trúc sư / DBA |
| D-014 | 2026-10-25 | SQL Server là nguồn sự thật cho account, seat, order, payment, ticket, audit và outbox; Redis chỉ giữ queue state/lock có TTL; không triển khai NoSQL riêng trong baseline. Operational logs đi ELK theo D-029. | Giảm thành phần và tránh nhiều nguồn dữ liệu nghiệp vụ | Product Owner / Kiến trúc sư |
| D-015 | 2026-10-25 | Q-001: hoạt động chọn ghế là thao tác UI hoặc heartbeat xác thực mỗi 30 giây khi trang ở foreground; WebSocket đơn thuần không gia hạn. Thu hồi token sau 120 giây im lặng; không tác động seat hold 600 giây. | Tách timeout admission khỏi TTL giữ ghế | BA baseline / Product Owner |
| D-016 | 2026-10-25 | Q-002: payment đã thu tiền sau khi hold hết hạn không được hồi sinh hold hoặc tạo order mới. Ghi SUCCESS_UNFULFILLED; sau đối soát xác nhận capture, tự hoàn 100% số tiền thực thu theo D-032. | Không chiếm ghế đã hết hạn; không bán trùng; bù trừ minh bạch | BA baseline / Payment / Finance |
| D-017 | 2026-10-25 | Q-003: hủy concert trước COMPLETED hoàn 100% order.final_amount cho mọi order PAID, kể cả vé CHECKED; ticket bị vô hiệu hóa, check-in log bất biến, ghế không đưa lại vào bán. Concert COMPLETED không chuyển sang CANCELLED. | Bảo đảm khách được hoàn khi dịch vụ sự kiện bị hủy | BA baseline / Legal/Finance ký trước go-live |
| D-018 | 2026-10-25 | Q-004: cho phép nhiều order tuần tự; tối đa 4 ticket hiệu lực/đang giữ trên mỗi user-concert và tối đa một active hold/checkout/payment UNKNOWN cho cùng cặp user-concert. Đơn đã hủy và refund hoàn tất không tính quota. | Cho phép đặt bổ sung có kiểm soát, ngăn song song vượt quota | BA baseline / Product Owner |
| D-019 | 2026-10-25 | Q-005: tiền VND là số nguyên; refund 50% tính trên tổng ticket subtotal đủ điều kiện của toàn order và làm tròn half-up một lần. Chỉ retry sau provider failure xác định, tối đa 3 lần; UNKNOWN theo D-032. | Tránh lệch từng vé và ngăn hoàn trùng | BA baseline / Finance / Payment |
| D-020 | 2026-10-25 | Q-007: sold-out hiển thị trong app và phát realtime tới waiting room; push là kênh bổ sung nếu được cho phép. Không cam kết email/SMS cho sold-out. | Thông báo tức thời cho người đang chờ, không phụ thuộc delivery provider | BA baseline / Product Owner |
| D-021 | 2026-10-25 | Q-008: trước ON_SALE, Organizer được sửa mô tả/media; đổi lịch, venue, seat map, zone hoặc giá cần Admin re-approval. Sau ON_SALE, khóa các trường thương mại/lịch/venue; mọi thay đổi khẩn cấp qua luồng amendment/cancel và thông báo. | Bảo vệ thông tin khách đã mua và kiểm soát thay đổi | BA baseline / Product Owner |
| D-022 | 2026-10-25 | Q-009: 100.000 CCU là mục tiêu baseline bắt buộc; 500.000 CCU là future scale target ngoài release này. | Khớp phạm vi capacity hiện tại và ngân sách baseline | BA baseline / Product Owner / SRE |
| D-023 | 2026-10-25 | Q-010 (gộp Q-006): offline QR check-in ngoài baseline. Scanner chỉ xác nhận khi online; khi mất mạng fail-closed, dừng luồng tự động và chuyển sự cố cho Gate Supervisor; không chấp nhận chỉ dựa vào local cache. | Giữ tính nhất quán single-use giữa nhiều cổng | BA baseline / Operations |
| D-024 | 2026-10-25 | Q-011: refund window tính bằng ngày lịch tại timezone của venue: ngày concert local trừ ngày yêu cầu local. Kết quả ≥7: 100%; 3–6: 50%; 0–2: 0%. | Có ranh giới tất định theo ngày, không tùy giờ request | BA baseline / Legal/Finance ký trước go-live |
| D-025 | 2026-10-25 | Q-012: RabbitMQ là broker baseline; Kafka ngoài phạm vi. Dùng durable queue, publisher confirm, retry có giới hạn, DLQ, idempotent consumer. | Phù hợp topology Worker đã mô tả và tránh hai lựa chọn triển khai | Kiến trúc sư / SRE |
| D-026 | 2026-10-25 | Q-013: production SQL Server 2025 Enterprise Always On AG; 2 replica sync cùng region (Primary + readable secondary) và 1 async DR replica khác region. Mục tiêu local RPO 0/RTO ≤5 phút; DR RPO ≤5 phút/RTO ≤60 phút. Full backup hằng ngày, differential mỗi 6 giờ, transaction log mỗi 5 phút; giữ backup 35 ngày, mã hóa và test restore hằng tháng. DBA/Finance xác nhận topology, sizing và licensing trước go-live. | Tách HA cục bộ khỏi DR; readable secondary cần edition hỗ trợ | Kiến trúc sư / DBA / Finance |
| D-027 | 2026-10-25 | Q-014: SLO là 99.95% mỗi tháng cho critical first-party purchase APIs, đo theo phút khả dụng; external payment-provider availability đo riêng. Loại 99.99% khỏi baseline và không gọi SLO này là SLA hợp đồng. | Chỉ giữ một mục tiêu có phạm vi đo rõ | BA baseline / SRE / Product Owner |
| D-028 | 2026-10-25 | Q-015: không partition SQL Server trong baseline. Dùng index/archival và đo benchmark; chỉ xem xét RANGE theo created_at cho bảng append-only khi ngưỡng tải/dung lượng không đạt. Cấm giả định HASH(user). | Tránh partition không cần thiết và thiết kế không tương thích SQL Server | DBA / Kiến trúc sư |
| D-029 | 2026-10-25 | Q-016: không có NoSQL riêng. Queue state dùng Redis TTL; transaction/audit/outbox dùng SQL Server. Outbox hỗ trợ replay tối đa 7 ngày; operational logs ELK lưu hot 30 ngày, lọc PII, không dùng làm nguồn sự thật. | Ít nguồn dữ liệu, replay dựa trên bản ghi bền vững | Kiến trúc sư / Security |
| D-030 | 2026-10-25 | Q-017: production release chỉ sau test 100.000 CCU trên topology production-like, đạt latency/error/availability criteria và zero double-booking/duplicate charge; DBA/Finance ký sizing và license. Tài liệu BA đặt gate, không tuyên bố benchmark đã chạy. | Tách target khỏi bằng chứng nghiệm thu thực tế | SRE / DBA / Finance |
| D-031 | 2026-10-25 | Q-018: voucher, discount campaign và voucher reports ngoài baseline; không có API/DB/UAT voucher trong release hiện tại; discount_amount bằng 0. | Tuân thủ scope đã giới hạn | BA baseline / Product Owner |
| D-032 | 2026-10-25 | Q-019: refund timeout là UNKNOWN; reconcile ở 30s, 5m, 30m. Không retry khi chưa rõ kết quả. Retry sau provider failure xác định tối đa 3 lần, backoff 1/5/30 phút; UNKNOWN quá 30 phút đưa manual queue; giữ nguyên idempotency key của logical refund. | Không tạo refund kép do timeout | Payment / Finance / DBA |
| D-033 | 2026-10-25 | Q-020: payment create/callback timeout là UNKNOWN; reconcile ở 30s, 5m, 30m. Không mở attempt khác tới khi provider xác nhận attempt trước thất bại/đã void; tối đa 3 attempts/order. UNKNOWN quá 30 phút đưa support/reconciliation queue; retry client dùng lại cùng idempotency key. | Không thu tiền hai lần khi kết quả gateway chưa rõ | Payment / Finance / DBA |
| D-034 | 2026-10-25 | Q-021: settlement lưu sổ dòng và snapshot điều khoản có phiên bản. Không hard-code commission rate, payout cooldown, chi phí refund hoặc custody/escrow; không giả định automated bank transfer. Chỉ cho payout sau concert COMPLETED, reconciliation MATCHED, không còn UNKNOWN/discrepancy/hold, Admin/Finance phê duyệt và lưu transfer reference/evidence. Hợp đồng/Legal/Finance approval bắt buộc trước production payout. | Làm rõ mô hình dữ liệu mà không tự đặt điều khoản thương mại/pháp lý | Product Owner / Legal / Finance |

---

## 4. Hướng Dẫn Cập Nhật Tài Liệu

- **Quyết định mới** phải được thêm vào bảng Decision Log với ID duy nhất theo định dạng `D-###` và ghi rõ ngày, người quyết định.
- **Câu hỏi đã được giải quyết** phải được chuyển từ bảng Open Questions sang Decision Log.
- **Rủi ro mới** được thêm vào bảng Rủi ro với ID `R-###`, phải kèm biện pháp giảm thiểu cụ thể.
- Mọi quyết định thay đổi phạm vi phải được cập nhật đồng thời trong [`02_Scope_Assumptions_Constraints_And_Dependencies.md`](./02_Scope_Assumptions_Constraints_And_Dependencies.md).

---

## 5. Tài Liệu Liên Quan

- [`00_Project_Governance/01_Stakeholders_And_Business_Objectives.md`](./01_Stakeholders_And_Business_Objectives.md)
- [`00_Project_Governance/02_Scope_Assumptions_Constraints_And_Dependencies.md`](./02_Scope_Assumptions_Constraints_And_Dependencies.md)
- [`01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md`](<../01_Business_Overview/03_Business_Rules_Catalog_(Business_Rules).md>)
- [`02_State_Machines_And_Workflows/`](../02_State_Machines_And_Workflows/)

