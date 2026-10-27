> Reviewer: Claude Code (running as a sub-agent session), model Claude Opus 5.5 (`claude-opus-5-5`)

# Board review: ADR-0002, publish `OrderPlaced` to RabbitMQ through a transactional outbox

**Evidence used:** only the ADR and `docs/scenario.md`. I could not check the ADR's `main.go` and ADR-0001 line citations, so for this review they count as *Not evidenced in the provided files*. Its `docs/scenario.md` citations (L7–14, L18, L22–30) are accurate.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QAS-1 Performance | Shoppers | 200 `PlaceOrder` calls/s | 11.11 peak, MailFast at p95 1.5 s | Order and outbox row committed, with no wait on email | `PlaceOrder` p95 ≤ 500 ms, ~300 ms today (scenario L24–25). MailFast never gets more than 50 emails/s, so there are 0 rate-limit rejections (L26) |
| QAS-2 Availability and reliability | Infrastructure | RabbitMQ, `loyaltyservice` **or checkout's new PostgreSQL** is down for 5 min | 200 orders/s | Orders still succeed. Events are kept and delivered after recovery | 0 orders fail (L28), 0 events are lost, 0 points are awarded twice, and points arrive within 1 min (L27). A catch-up time for the ~60,000 queued events (5 min × 200/s) must also be set |
| QAS-3 Modifiability | Dev team | Add SMS next year | Design time | Add a new queue and consumer bound to the `orders` exchange | 0 changes to `checkoutservice` and 0 redeploys of it (L14) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | If the outbox insert fails, the error is "logged and counted, never returned" (Decision, *Outbox write*). The Validation section even tests that orders succeed while PostgreSQL is down. | Every order in a PostgreSQL outage loses its email, points and analytics. That breaks "must be awarded" (L27). Events are now lost when PostgreSQL is down instead of when RabbitMQ is down, and the ADR never shows that a primary-plus-standby PostgreSQL is more available than a 3-node quorum RabbitMQ cluster. |
| Risk | The outbox commit runs after `chargeCard`, `shipOrder` and `emptyUserCart` (Risks row 1, Open question 1). | The transaction makes the order row and the event atomic, but not the **payment**. A card can be charged with no record and no event, and nothing would detect it. |
| Risk | The ADR sets no timeout for the outbox insert and does not size the connection pool. | A slow database adds its delay straight to `PlaceOrder`. That is the same flaw the ADR criticises in the email call ("no timeout of its own", Context table). QAS-1 depends on this timeout. |
| Risk | The single-active-consumer failover redelivers every unacknowledged message, up to ~75 in flight. | "None are lost" (Risks table) is true, but some of those emails are sent **twice**. Whether MailFast supports idempotency keys is still unknown (Open question 4). |
| Sensitivity point | The email consumer's concurrency and prefetch (~75) and the token-bucket burst size | The 75 comes from the p95 latency, not the mean. With an 8 s p99 and timeouts longer than 8 s, too few slots push the send rate below 50/s and make the 11.11 drain (≥ 3 min per peak minute) longer. A burst larger than 1 can go over 50 emails in some 1-second window. |
| Sensitivity point | The relay's polling interval, batch size and drain rate, and `loyaltyservice` throughput above 200/s | After an outage, the backlog drains only as fast as the relay and loyalty can work. "A broker outage only delays it" hides that the 1-minute loyalty target is missed during that time. |
| Trade-off point | Swallowing outbox failures raises checkout availability (driver 1) and lowers loyalty reliability (driver 3). | The ADR makes this choice without saying so. L28 protects orders only from email, loyalty and analytics outages, not from checkout's own database, so falling back or failing early is a valid option to weigh. |
| Trade-off point | The outbox adds a second stateful component to the request path: more reliability, but worse operability, cost, latency and team fit. | The scenario shows RabbitMQ experience only (L29) and says nothing about PostgreSQL. The fit table rated option 3 "Good" for team fit **before** the outbox was added. |
| Non-risk | Each consumer has its own durable quorum queue with a dead-letter queue. | The email backlog (9,000 per peak minute) cannot delay loyalty (L26–27), and a new action only needs a new binding (QAS-3). |
| Non-risk | Loyalty uses a unique `order_id` and awards points in the same transaction. A single active consumer with a token bucket handles email. | At-least-once delivery plus idempotent processing gives effectively exactly-once points. One global limiter fixes the per-replica limit flaw that ruled out option 2. The ADR's arithmetic is correct (1.5 s > 500 ms, 150/s × 60 = 9,000, 0.25 failures/s). |

## 3. Missing or weak

- **Options not considered**
  - "Outbox vs. direct publish with publisher confirms" was never compared as an option. The outbox appears only in the Decision, so the fit table never priced in PostgreSQL.
  - A **fallback**: if the outbox insert fails, publish directly with a short timeout. Then an event is lost only when both PostgreSQL and RabbitMQ are down.
- **Consequences not stated**
  - Duplicate emails on failover.
  - No recovery-time target for loyalty after outages.
  - Trace context must be **stored in the outbox row**, because the relay publishes later without the request `ctx`.
  - Checkout pods now carry the relay's polling load and the database connections.
  - With no retention period, the outbox table keeps growing, at up to 200 rows/s during the peak.
- **Claims that are unsupported or likely wrong**
  - "Writing to a local database is faster and more predictable than a broker publish with confirms." No numbers support it. PostgreSQL is also a network hop with a commit fsync, plus a standby round trip if replication is synchronous **[Inferred]**. Open question 3 admits the latency is unmeasured.
  - "Its list of dependencies shrinks." The *Configuration* bullet replaces one address with **two** (database and broker) and adds schema migrations. The list stops growing with each new action, but it does not shrink.
  - "A broker outage only delays it." That is true for durability, but any delay over 1 minute breaks L27.

## 4. Verdict: **Accept with changes**

1. **Close the loss windows.** Make Open question 1 part of the decision: write the order as `pending` **before** `chargeCard`, and add a reconciler for charged orders that have no event. Add the direct-publish fallback for when the insert fails. Decide explicitly what checkout does when PostgreSQL is down.
2. **Bound the request path.** Set an explicit outbox-insert timeout and pool size. Replace the "faster than a broker publish" claim with the measured p95 and p99 of the insert at 200 orders/s, with the standby enabled.
3. **State delivery targets under failure, and test them.** Set a loyalty catch-up target after an outage and size the relay and `loyaltyservice` for it, for example to drain 60,000 events. Handle duplicate emails on failover, using `order_id` as the MailFast idempotency key or a deduplication store. Store `traceparent` in the outbox row. Add each of these to Validation.
