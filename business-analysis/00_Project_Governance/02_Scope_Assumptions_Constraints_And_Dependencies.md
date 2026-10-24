# Scope, Assumptions, Constraints and Dependencies

## In Scope

- Account registration, authentication and personal account management.
- Concert discovery, search, filtering and concert detail viewing.
- Waiting-room admission and queue position management.
- Seat-map viewing, seat holding and seat inventory transitions.
- Order creation, payment, payment retry handling and payment status tracking.
- Electronic ticket issuance, QR check-in, cancellation and refund processing.
- Concert creation, approval, publication, sales management and cancellation.
- Notifications, concert ratings and business reporting.
- Web, iOS and Android clients connected to backend services through REST APIs.

## Out of Scope

- Physical venue construction or venue operations unrelated to ticket validation.
- Payment-provider internal implementation.
- Detailed infrastructure deployment scripts and source-code design.
- Marketing campaigns, loyalty programs and resale/transfer workflows unless approved later.

## Assumptions

- Each account has at most one queue position for a concert, regardless of tabs or devices.
- A customer can hold at most four seats for a concert.
- A seat hold expires after ten minutes unless payment succeeds within the valid hold period.
- A waiting-room position is lost when the customer is inactive for more than two minutes.
- QR tickets are single-use and can be checked in only while the ticket is valid.
- Refund eligibility follows the approved policy based on the time remaining before the concert.

## Constraints

- The platform must support Web, iOS and Android clients.
- The target scale is approximately two million users.
- The seat-selection area supports a maximum of 5,000 concurrent customers.
- Ticket-sale traffic may reach approximately 100,000 concurrent users.
- Concert approval should be completed within 48 hours.
- The scheduled sale time must be at least one day after concert approval.

## Dependencies

- VNPay and MoMo payment and refund APIs.
- Email and in-app notification delivery providers.
- Identity and authentication components.
- Venue, seating-map and pricing information supplied by organizers.
- QR scanning devices and network connectivity at concert entrances.
- Infrastructure services for caching, messaging, storage and content delivery.

## Items Requiring Confirmation

- The exact definition of customer activity for the two-minute queue timeout.
- The handling of a successful late payment after a seat hold has expired.
- Whether ticket transfer between customer accounts is supported.
- The policy for refund rounding and partial refund failure.
