# Stakeholders and Business Objectives

## Purpose

This document identifies the stakeholders involved in the concert ticketing system and
defines the business objectives that guide the analysis and delivery of the product.

## Stakeholders

| Stakeholder | Responsibility and interest |
|---|---|
| Customers | Discover concerts, purchase tickets, receive electronic tickets, request refunds and check in. |
| Organizers | Create concerts, configure venues and seats, manage ticket sales and review reports. |
| Administrators | Review and publish concerts, manage accounts, approve cancellation requests and handle platform-level cancellation. |
| Check-in staff | Scan QR codes and validate tickets at the concert entrance. |
| Payment providers | Process payments and refunds through VNPay, MoMo or other supported providers. |
| Platform operations team | Monitor availability, capacity, incidents and operational health. |

## Business Objectives

1. Provide a reliable online ticket purchasing experience across Web, iOS and Android.
2. Support high-demand ticket sales for a large user base without duplicate seat allocation.
3. Manage waiting-room admission and seat inventory fairly and transparently.
4. Ensure payment, refund and ticket issuance processes are safe, traceable and idempotent.
5. Give organizers and administrators the controls needed to manage the concert lifecycle.
6. Enable fast, one-time QR ticket validation at the concert entrance.
7. Provide timely notifications and business reports for relevant stakeholders.

## Success Indicators

- No seat is successfully sold or held by more than one customer at the same time.
- A retried payment request does not charge the customer twice.
- A valid ticket QR code can be checked in only once.
- Customers receive clear status and failure notifications.
- Organizers and administrators can complete their required approval and management actions.

## Related Documents

- `01_Business_Overview/01_Business_Context_And_System_Scope.md`
- `01_Business_Overview/02_Actor_Permissions.md`
- `02_State_Machines_And_Workflows/`
