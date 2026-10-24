# 05 - State Models

Chuẩn hóa vòng đời và các chuyển trạng thái hợp lệ của những đối tượng nghiệp vụ chính.

## State models cần tạo

- `concert-state-model.md`
- `seat-state-model.md`
- `order-state-model.md`
- `payment-and-refund-state-model.md`
- `ticket-state-model.md`
- `state-transition-matrix.md`

## Đối tượng cần chuẩn hóa

- Concert: `PENDING_REVIEW`, `PUBLISHED`, `ON_SALE`, `SOLD_OUT`, `CLOSED`, `COMPLETED`, `CANCELLED`.
- Ghế: `AVAILABLE`, `HELD`, `SOLD`, `BLOCKED`.
- Đơn hàng: `PENDING`, `PAID`, `OUT_TIME`, `CANCELLED`.
- Hoàn tiền: `REFUND_PENDING`, `REFUNDED`.
- Vé: `ORDERED`, `CHECKED`, `CANCELLED`.
