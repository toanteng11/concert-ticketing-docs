# 01 — Tiêu Chí Nghiệm Thu & Kế Hoạch Kiểm Thử Người Dùng (Acceptance Criteria & UAT Plan)

## 1. Giới Thiệu & Mục Tiêu Nghiệm Thu (UAT Scope)

Kiểm thử Chấp nhận của Người dùng (**User Acceptance Testing - UAT**) là giai đoạn quyết định trước khi hệ thống bán vé sự kiện âm nhạc chính thức phát hành ra thị trường (Go-Live). Mục tiêu của tài liệu này:
1. Thiết lập các **Tiêu chí Nghiệm thu Nghiệp vụ (Business Acceptance Criteria)** chi tiết, có thể lượng hóa và kiểm chứng được cho từng module chức năng.
2. Cung cấp quy trình bàn giao và biểu mẫu nghiệm thu (**UAT Sign-Off Template**) để Ban Quản Trị Hệ Thống (Sàn vé) và Đại diện Ban Tổ Chức Sự Kiện (Organizer) cùng ký duyệt.
3. Đảm bảo toàn bộ 15 Use Cases (UC01 – UC15) và các Quy tắc Nghiệp vụ (BR-001 đến BR-NOTI-003) được nghiệm thu đạt $100\%$ không có lỗi nghiêm trọng (Zero Blocker / Zero Critical Defects).

---

## 2. Tiêu Chí Nghiệm Thu Theo Từng Phân Hệ Chức Năng (Detailed Acceptance Criteria)

### Phân Hệ 1: Tài Khoản, Xác Thực & Phân Quyền (UC01 – UC03)
- [ ] **AC-UAT-01.1:** Khách hàng đăng ký tài khoản bằng số điện thoại Việt Nam hợp lệ; nhận mã OTP qua SMS trong vòng $\le 10\text{ giây}$ và kích hoạt tài khoản thành công.
- [ ] **AC-UAT-01.2:** Nhập sai mật khẩu 5 lần liên tiếp $\rightarrow$ Tài khoản tự động bị khóa tạm thời 15 phút; hiển thị thông báo rõ ràng kèm đồng hồ đếm lùi.
- [ ] **AC-UAT-01.3:** Đăng nhập thành công trả về JWT Token; trên trình duyệt Web, Refresh Token được lưu trong `HttpOnly Cookie`. Khi đăng xuất, Refresh Token bị hủy bỏ vĩnh viễn trên Database.
- [ ] **AC-UAT-01.4:** Phân quyền RBAC chính xác $100\%$: Khách hàng thông thường truy cập URL của Admin (`/admin/dashboard`) bị chặn ngay lập tức với mã lỗi `403 Forbidden`.

---

### Phân Hệ 2: Tìm Kiếm, Phòng Chờ Ảo & Sơ Đồ Ghế (UC04 – UC06)
- [ ] **AC-UAT-02.1:** Trước giờ mở bán $15\text{ phút}$, truy cập trang concert tự động hiển thị giao diện Phòng Chờ Ảo (Waiting Room) kèm đồng hồ đếm ngược; không cho phép nhấp chọn ghế trước giờ G.
- [ ] **AC-UAT-02.2:** Đúng giờ mở bán ($T$), hệ thống xáo trộn ngẫu nhiên số thứ tự hàng chờ và cấp số cho toàn bộ người có mặt trong phòng chờ; thời gian xử lý xáo trộn $< 3\text{ giây}$.
- [ ] **AC-UAT-02.3:** Sơ đồ khán đài hiển thị trực quan các phân khu (Zone), hàng ghế và vị trí từng ghế; trạng thái màu sắc thay đổi mượt mà theo thời gian thực (Trắng: Trống; Vàng: Đang giữ; Đỏ: Đã bán).
- [ ] **AC-UAT-02.4:** Một tài khoản chỉ được phép chọn tối đa **4 vé** trong một lần giao dịch. Cố chọn chiếc thứ 5 lập tức bị hệ thống cảnh báo và vô hiệu hóa.
- [ ] **AC-UAT-02.5 (Tuyệt đối không bán trùng):** Khi 2 người dùng cùng nhấp vào 1 ghế trống tại cùng một thời điểm: Duy nhất 1 người nhận được thông báo "Giữ ghế thành công"; người còn lại nhận thông báo "Ghế vừa được người khác giữ" trong vòng $< 300\text{ ms}$.
- [ ] **AC-UAT-02.6:** Đồng hồ đếm lùi giữ ghế 10 phút ($600\text{ giây}$) hiển thị chính xác. Khi hết $600\text{ giây}$ mà khách chưa thanh toán, ghế tự động chuyển về trạng thái `AVAILABLE` cho cộng đồng.

---

### Phân Hệ 3: Đặt Hàng, Thanh Toán & Hoàn Tiền (UC07 – UC09)
- [ ] **AC-UAT-03.1:** Tính toán hóa đơn chính xác tuyệt đối: `Tổng tiền thanh toán = (Số vé x Đơn giá) - Giảm giá Voucher + Phí dịch vụ`.
- [ ] **AC-UAT-03.2:** Khách hàng bấm thanh toán $\rightarrow$ Chuyển hướng mượt mà sang Cổng VNPay hoặc MoMo; mã QR thanh toán quét được bằng ứng dụng ngân hàng di động.
- [ ] **AC-UAT-03.3:** Nhấp đúp nút "Xác nhận thanh toán" hoặc tải lại trang thanh toán nhiều lần không bao giờ bị trừ tiền 2 lần (Nhờ Idempotency Key).
- [ ] **AC-UAT-03.4:** Sau khi chuyển tiền thành công, trạng thái đơn hàng tự động chuyển sang `PAID` trong vòng $< 3\text{ giây}$ mà không cần người dùng tải lại trang.
- [ ] **AC-UAT-03.5 (Chính sách hoàn tiền):** Khách hàng hủy vé trước ngày diễn $\ge 7\text{ ngày}$ được hoàn $100\%$ tiền; từ 3–6 ngày được hoàn $50\%$; dưới 3 ngày hoàn $0\%$. Số tiền hoàn được tự động đối soát và trả về thẻ/ví gốc trong 3–5 ngày làm việc.

---

### Phân Hệ 4: Quản Trị Concert & Báo Cáo Doanh Thu (UC11 – UC13)
- [ ] **AC-UAT-04.1:** Ban tổ chức tạo concert mới ở trạng thái `DRAFT`, tải lên đầy đủ thông tin, cấu hình sơ đồ ghế, giá vé và đính kèm Giấy phép biểu diễn $\rightarrow$ Gửi duyệt chuyển sang `PENDING_REVIEW`.
- [ ] **AC-UAT-04.2:** Admin sàn có thể xem xét inline tài liệu pháp lý, phê duyệt (`PUBLISHED`) hoặc từ chối kèm lý do chi tiết; Organizer nhận email thông báo trong $< 5\text{ giây}$.
- [ ] **AC-UAT-04.3:** Khi concert đã ở trạng thái `ON_SALE`, hệ thống khóa toàn bộ chức năng sửa đổi sơ đồ ghế, số lượng vé và giá niêm yết để bảo vệ quyền lợi người mua.
- [ ] **AC-UAT-04.4:** Dashboard của Organizer cập nhật số lượng vé đã bán và doanh thu ròng với độ trễ không quá $60\text{ giây}$ so với thời điểm thanh toán thực tế; hỗ trợ xuất file Excel báo cáo chi tiết.

---

### Phân Hệ 5: Soát Vé Tại Cổng & Thông Báo Đa Kênh (UC14 – UC15)
- [ ] **AC-UAT-05.1:** Ứng dụng khách hàng hiển thị Dynamic QR Code tự động làm mới chu kỳ 30 giây/lần; chụp ảnh màn hình gửi cho người khác quét sau 30 giây bị cổng từ chối (`QR_TOKEN_EXPIRED`).
- [ ] **AC-UAT-05.2:** Thiết bị nhân viên quét vé hợp lệ phản hồi trong vòng $\le 500\text{ ms}$: Màn hình xanh lá, âm báo thành công, hiển thị tên khách và số ghế; vé lập tức chuyển sang `CHECKED`.
- [ ] **AC-UAT-05.3:** Cùng một mã vé quét lần thứ 2 tại bất kỳ cổng nào lập tức bị còi báo động đỏ `TICKET_ALREADY_USED`.
- [ ] **AC-UAT-05.4 (Khả năng ngoại tuyến):** Khi ngắt kết nối mạng Internet tại cổng soát vé, thiết bị tự động chuyển sang chế độ Offline Mode và tiếp tục quét hợp lệ danh sách vé đã đồng bộ trước đó.
- [ ] **AC-UAT-05.5:** $100\%$ đơn hàng thanh toán thành công nhận được Email xác nhận kèm vé điện tử PDF đính kèm trong vòng không quá 5 giây.

---

## 3. Quy Trình Vận Hành Kiểm Thử Chấp Nhận Người Dùng (UAT Execution Process)

Quy trình UAT kéo dài trong vòng 2 tuần (10 ngày làm việc) trên môi trường Staging chuyên biệt:

1. **Giai đoạn Chuẩn bị (Ngày 1 – 2):**
   - Thiết lập môi trường Staging có cấu hình tương đương $50\%$ Production.
   - Nạp dữ liệu giả lập (Mock Data): 5 concert mẫu, 1 địa điểm SVĐ 50.000 chỗ, 10.000 tài khoản người dùng thử nghiệm.
   - Kết nối cổng thanh toán VNPay Sandbox và MoMo Test Environment.
2. **Giai đoạn Thực thi Kịch bản UAT (Ngày 3 – 7):**
   - Đội ngũ đại diện khách hàng (End-Users), Ban tổ chức (Organizers) và Nhân viên soát vé (Staff) trực tiếp thao tác theo bộ kịch bản kiểm thử (Test Scripts).
   - Ghi nhận lỗi vào hệ thống Jira/Bug Tracker theo 4 mức độ: Blocker, Critical, Major, Minor.
3. **Giai đoạn Sửa lỗi & Kiểm thử hồi quy (Ngày 8 – 9):**
   - Đội ngũ phát triển khắc phục $100\%$ lỗi Blocker và Critical.
   - Chạy lại toàn bộ kiểm thử hồi quy tự động (Automated Regression Testing).
4. **Giai đoạn Ký duyệt Nghiệm thu (Ngày 10):**
   - Đánh giá chỉ số vượt qua nghiệm thu (Pass Rate $\ge 98\%$, không còn lỗi nghiêm trọng).
   - Các bên liên quan ký biên bản bàn giao UAT Sign-Off.

---

## 4. Biểu Mẫu Biên Bản Ký Duyệt Nghiệm Thu (UAT Sign-Off Template)

```markdown
# BIÊN BẢN NGHIỆM THU NGƯỜI DÙNG (UAT SIGN-OFF RECORD)
Dự án: Nền Tảng Bán Vé Sự Kiện Âm Nhạc Trực Tuyến (Concert Ticketing Platform)
Phiên bản phần mềm: v1.0.0-Release Candidate
Môi trường kiểm thử: Staging Environment (https://staging.ticketplatform.vn)
Thời gian thực hiện: Từ ngày DD/MM/2026 đến ngày DD/MM/2026

1. TỔNG KẾT KẾT QUẢ KIỂM THỬ:
- Tổng số kịch bản kiểm thử (Test Cases): 120 kịch bản
- Số kịch bản Đạt (Passed): 118 kịch bản (98.3%)
- Số kịch bản Không Đạt (Failed): 0 kịch bản
- Số kịch bản Bỏ qua (Skipped - Có lý do): 2 kịch bản (Thuộc tính năng Phase 2)
- Lỗi tồn đọng mức Blocker / Critical: 0 lỗi
- Lỗi tồn đọng mức Minor (Chấp nhận khắc phục sau Go-Live): 3 lỗi giao diện

2. KẾT LUẬN NGHIỆM THU:
Hệ thống đáp ứng đầy đủ các tiêu chuẩn kỹ thuật, tính toàn vẹn nghiệp vụ bán vé, 
tuyệt đối không bán trùng ghế, bảo mật dòng tiền và sẵn sàng phát hành chính thức.

3. ĐẠI DIỆN CÁC BÊN XÁC NHẬN KÝ TÊN:

Đại Diện Ban Quản Trị Hệ Thống          Đại Diện Ban Tổ Chức Sự Kiện (Organizer)
(Ký và ghi rõ họ tên)                  (Ký và ghi rõ họ tên)
Ngày: ..../..../2026                   Ngày: ..../..../2026
```
