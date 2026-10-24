# Acceptance Criteria and UAT

## Purpose

Define business-facing acceptance criteria and user acceptance testing (UAT) coverage
for the concert ticketing platform.

## Acceptance Criteria

### Waiting Room and Queue

- The waiting room opens 15 minutes before the scheduled sale time.
- Customers entering before the sale time are assigned a randomized queue position at
  the sale start time.
- Customers entering after the sale start time are placed after customers already in
  the queue.
- One account has at most one queue position for each concert across tabs and devices.
- A customer who is inactive for more than two minutes loses the queue position.
- No more than 5,000 customers are admitted to the seat-selection area concurrently.

### Seat Holding and Ordering

- A customer can hold no more than four seats.
- A seat can have only one active holder or one sold ticket at a time.
- A successful hold changes the seat from `AVAILABLE` to `HELD`.
- An expired hold releases the seat back to `AVAILABLE`.
- An order can be created only while the seat hold is valid.

### Payment and Refund

- Retrying the same payment request does not charge the customer twice.
- A successful payment changes the order to `PAID` and issues the electronic ticket.
- A late payment callback is reconciled explicitly when the seat hold has expired.
- Refund eligibility follows the approved 100% / 50% / 0% policy.
- Customers receive a notification after a completed refund.

### Ticket and Check-in

- A successful payment creates an `ORDERED` ticket with a QR code.
- A valid QR code can transition a ticket from `ORDERED` to `CHECKED` only once.
- A checked-in ticket cannot be refunded.
- A refunded ticket transitions to `CANCELLED` and cannot be checked in.

## UAT Evidence

Each UAT case should record the test ID, actor, preconditions, test data, execution
steps, expected result, actual result, evidence link and approval status.
