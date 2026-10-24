# Performance and Resilience Test Plan

## Purpose

Validate that the platform remains responsive, consistent and recoverable during
normal operation, peak ticket sales and dependency failures.

## Performance Scope

- Approximately 2 million registered users.
- Approximately 100,000 concurrent users during a ticket-sale spike.
- Maximum 5,000 concurrent customers in the seat-selection area.
- High-concurrency seat selection for the same concert and seat inventory.
- Payment retries and callback bursts from VNPay and MoMo.
- Concert browsing, seat-map reads and notification delivery.

## Performance Scenarios

| ID | Scenario | Main measurements |
|---|---|---|
| PT-001 | Normal concert browsing. | Response time, throughput, error rate and cache hit rate. |
| PT-002 | 100,000 concurrent users enter the waiting room. | Admission rate, queue stability, CPU, memory and error rate. |
| PT-003 | 5,000 users select seats concurrently. | Seat allocation latency, conflict rate and duplicate allocation count. |
| PT-004 | Payment provider callback burst. | Callback processing latency, idempotency behavior and backlog. |
| PT-005 | Expiry of a large number of seat holds. | Release throughput, stale hold count and database load. |

## Resilience Scenarios

| ID | Failure | Expected behavior |
|---|---|---|
| RT-001 | Cache service is unavailable. | System falls back safely without selling duplicate seats. |
| RT-002 | Primary database is unavailable. | Writes fail clearly or wait for recovery; no false payment or seat success is returned. |
| RT-003 | Message broker is delayed. | Payment completion remains traceable; notifications retry without data loss. |
| RT-004 | Payment provider timeout. | Order remains recoverable and retrying does not create a duplicate charge. |
| RT-005 | Notification provider failure. | Delivery is retried and failure is visible to operations. |
| RT-006 | Duplicate QR scan at two entrances. | Only one atomic transition to `CHECKED` succeeds. |
| RT-007 | Application instance failure during checkout. | The customer can resume or safely retry without duplicate order or payment. |

## Required Evidence

- Load-test scripts and test-data configuration.
- Latency percentiles, throughput, saturation and error-rate reports.
- Seat-allocation conflict and duplicate-sale report.
- Payment idempotency and reconciliation report.
- Recovery time, recovery point and data-integrity evidence.
- Incident logs and actions taken during resilience tests.

## Exit Criteria

Performance and resilience testing is complete when agreed service-level thresholds
are met, no duplicate seat allocation or duplicate charge is observed, and all
critical failure scenarios have an approved recovery result.
