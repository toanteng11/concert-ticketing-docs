# Functional and Edge Case Test Scenarios

## Functional Scenarios

| ID | Area | Scenario | Expected result |
|---|---|---|---|
| FS-001 | Account | Register and log in with valid credentials. | Account is authenticated successfully. |
| FS-002 | Concert | Search and filter concerts by date. | Matching concerts are displayed. |
| FS-003 | Queue | Join a concert waiting room before sales open. | Customer receives one queue position. |
| FS-004 | Seats | Hold four available seats. | All four seats become `HELD`. |
| FS-005 | Order | Create an order while the hold is valid. | Order is created in `PENDING`. |
| FS-006 | Payment | Complete payment through VNPay or MoMo. | Order becomes `PAID` and ticket becomes `ORDERED`. |
| FS-007 | Refund | Request a refund under each time-based policy. | Correct refund percentage is applied. |
| FS-008 | Check-in | Scan a valid ticket QR code. | Ticket changes to `CHECKED`. |
| FS-009 | Organizer | Submit a concert for approval. | Concert enters `PENDING_REVIEW`. |
| FS-010 | Admin | Approve and publish an eligible concert. | Concert becomes available according to its schedule. |

## Edge Case Scenarios

| ID | Area | Scenario | Expected result |
|---|---|---|---|
| EC-001 | Queue | Same account opens multiple tabs or devices. | Only one queue position is retained. |
| EC-002 | Queue | Customer is inactive for more than two minutes. | Queue position is released. |
| EC-003 | Seats | Two customers select the same seat simultaneously. | Exactly one succeeds; the other receives a seat-unavailable message. |
| EC-004 | Seats | Customer attempts to hold a fifth seat. | Request is rejected with a clear limit message. |
| EC-005 | Seats | Hold expires during checkout. | Seats are released and the order cannot be created as valid. |
| EC-006 | Payment | Client retries after a network timeout. | No duplicate charge or duplicate order is created. |
| EC-007 | Payment | Provider callback arrives after hold expiry. | Late payment is reconciled according to the approved policy. |
| EC-008 | Check-in | Same QR code is scanned twice. | First scan succeeds; second scan is rejected. |
| EC-009 | Refund | Customer requests a refund after check-in. | Refund is rejected. |
| EC-010 | Concert | Organizer tries to edit seats after sales begin. | Change is rejected. |
| EC-011 | Concert | Admin cancels a concert before it occurs. | Concert is cancelled and eligible tickets are refunded. |
| EC-012 | Sales | All sellable inventory is unavailable. | Sales close and waiting customers receive a sold-out notification. |

## Execution Notes

For each scenario, capture the input data, actor, timestamp, system status before and
after execution, notification result and any payment-provider reference.
