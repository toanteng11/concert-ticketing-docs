# Risks, Open Questions and Decision Log

## Key Risks

| ID | Risk | Impact | Mitigation or response |
|---|---|---|---|
| R-001 | Sudden traffic spike at the start of ticket sales. | Slow responses or unavailable purchasing flow. | Apply queue controls, rate limits, load balancing and horizontal scaling. |
| R-002 | Concurrent customers select the same seat. | Duplicate booking or customer disputes. | Enforce atomic seat allocation and database consistency controls. |
| R-003 | Payment callback is retried or arrives late. | Duplicate charges or inconsistent order status. | Use idempotency keys and explicit late-payment reconciliation. |
| R-004 | Notification delivery fails. | Customers miss payment, refund or ticket updates. | Retry delivery, record failures and provide in-app status visibility. |
| R-005 | QR code is presented at multiple entrances. | Duplicate check-in or unauthorized entry. | Atomically transition the ticket from `ORDERED` to `CHECKED`. |

## Open Questions

- What exact event marks a customer as active in the waiting room?
- What happens to a successful payment when the seat hold expired before the provider callback arrived?
- Does an administrator cancellation always trigger an automatic 100% refund, including already checked-in tickets?
- Can a customer buy multiple orders for the same concert?
- Is offline QR scanning required at venue entrances?
- How are partial refund failures retried and communicated to the customer?

## Decision Log

| ID | Date | Decision | Rationale | Owner |
|---|---|---|---|---|
| D-001 |  | A customer may hold a maximum of four seats. | Prevents excessive inventory blocking and follows the business brief. | Product owner |
| D-002 |  | A seat hold lasts ten minutes. | Gives the customer time to complete payment while releasing abandoned inventory. | Product owner |
| D-003 |  | QR tickets are single-use. | Prevents reuse of an already checked-in ticket. | Product owner |
| D-004 |  | Payment requests must be idempotent. | Prevents duplicate charges during retries or network interruptions. | Product owner / Payment team |

## Decision Status

- Decisions without an owner or date are provisional and require confirmation.
- New decisions should be appended with a unique `D-###` identifier.
- Resolved questions should be moved from `Open Questions` into the decision log.
