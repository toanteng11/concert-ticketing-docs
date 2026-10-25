# Chuẩn biên tập và cách đọc bộ tài liệu BA

## Mục đích

Quy ước này giúp phân biệt yêu cầu đã xác nhận với giả định, đề xuất thiết kế và câu hỏi còn mở. Đây là hướng dẫn biên tập, không tự tạo quyết định nghiệp vụ.

## Thứ tự ưu tiên khi tài liệu mâu thuẫn

1. Decision Log đã được xác nhận.
2. Business Rules Catalog và Scope đã được Product Owner xác nhận.
3. Use case, state machine, business flow và acceptance criteria.
4. Ước lượng tải, kiến trúc, ERD và hướng dẫn vận hành.

Nếu chưa xác định được ý định, ghi vấn đề vào Open Questions và đánh dấu nội dung phụ thuộc là **Tạm thời — chờ xác nhận**.

## Nhãn trạng thái

- **Đã chốt**: có quyết định hoặc xác nhận rõ ràng; ghi nguồn tham chiếu.
- **Giả định**: dùng cho phân tích/ước lượng, chưa phải cam kết sản phẩm.
- **Đề xuất kỹ thuật**: phương án kiến trúc cần review trước khi triển khai.
- **Chưa quyết định**: còn câu hỏi trong Decision Log; không dùng làm tiêu chí nghiệm thu bắt buộc.

## Chuẩn thuật ngữ và đơn vị

- Dùng nhất quán concert, order, ticket và seat hold.
- Lưu thời điểm máy theo UTC; hiển thị theo múi giờ sự kiện (mặc định Asia/Ho_Chi_Minh) nếu chưa có quyết định khác.
- CCU là người dùng đồng thời; QPS là yêu cầu mỗi giây. Ghi rõ phạm vi đo, thời gian duy trì và loại tải.
- Phân biệt thời hạn phiên/hàng chờ với TTL giữ ghế.
- Giữ nguyên mã BR-*, UC*, BF*, SM*, FS-* và EC-* khi tham chiếu chéo.

## Sửa đổi và truy vết

Khi cập nhật yêu cầu, cập nhật đồng bộ quy tắc, use case/flow/state machine, acceptance criteria và RTM liên quan. Không tạo mã BR trong RTM nếu mã đó không tồn tại trong Business Rules Catalog. Nếu chưa có test/AC cụ thể, ghi **Chưa ánh xạ** thay vì ngụ ý đã được kiểm thử.

## Điểm cần xác nhận

Danh sách câu hỏi và trạng thái được quản lý trong Decision Log. Các quyết định baseline đã chốt gồm giữ ghế 600 giây, payment/refund reconciliation, phạm vi online-only check-in, mốc hoàn tiền theo ngày địa phương, RabbitMQ, SQL Server 2025 Enterprise HA/DR, SLO 99.95%, không partition/NoSQL, release gate 100.000 CCU và loại voucher khỏi baseline. Các mục còn cần xác nhận bên ngoài được ghi rõ là điều kiện go-live, không phải quyết định nghiệp vụ còn mở.
