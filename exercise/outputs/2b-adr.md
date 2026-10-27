Agent: Claude Code (running as a sub-agent session). Model: Claude Opus 5.5 (`claude-opus-5-5`).

# ADR-0002: Publish an `OrderPlaced` event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** To be confirmed. The team is 4 developers (`docs/scenario.md` L29); who decides is not evidenced in the provided files.

> Draft for Task 2, run b. Evidence: `docs/scenario.md` (requirement R-12 and its numbers), `docs/adr/0001-use-redis-for-cart-storage.md` (style), and `system/online-boutique/src/checkoutservice/main.go`, called **`main.go`** below. Anything marked **[Inferred]** is not shown by these files.

## Context and problem

**Requirement R-12** (`docs/scenario.md` L7–14): after an order is placed, (1) send the confirmation email through **MailFast**, an external provider that replaces the mock `emailservice` logic; (2) award **loyalty points** through a new `loyaltyservice`; (3) record an **"order placed" analytics event**. Marketing expects more post-order actions next year (SMS, fraud scoring, a recommendations update).

**How checkout works today**

| Fact | Evidence |
|---|---|
| `PlaceOrder` does all its work synchronously and in sequence, on the incoming request's `ctx`: it gets the cart, catalog prices, a shipping quote and currency conversion, then calls `chargeCard` (paymentservice), `shipOrder` (shippingservice), `emptyUserCart` (cartservice) and `sendOrderConfirmation` (emailservice) | `main.go` L230–280 (L238, L252, L258, L263, L273) |
| Every downstream call is gRPC. The connections are created at startup | `main.go` L118–123, L210–220 |
| If payment or shipping fails, the order fails. If email fails, checkout only logs a warning, and it ignores the error from emptying the cart | `main.go` L253–255, L259–261, L263, L273–277 |
| The email call has no timeout of its own. It uses the request `ctx`, so email latency adds straight to `PlaceOrder` latency | `main.go` L273, L379–384 |
| Each downstream service needs a struct field, an env var and a connection. Adding loyalty, analytics or SMS as more synchronous calls means editing all three places plus `PlaceOrder` | `main.go` L66–86, L111–116, L118–123 |
| The order ID is a UUID created before payment, so it can serve as an idempotency key | `main.go` L233, L266 |
| The order total is computed in checkout but is **not** in `OrderResult`. The user ID and email come from the request | `main.go` L243–250, L265–271, L231, L273 |
| `checkoutservice` has no database client and no message-broker client | `main.go` L17–43 (imports) |
| OpenTelemetry trace context is propagated over gRPC | `main.go` L135–140, L214–216 |

The scenario's summary ("payment, shipping, then email", `docs/scenario.md` L18) leaves out the cart, catalog and currency calls and the `EmptyCart` call that the code makes.

**Forces** (numbers from `docs/scenario.md` L22–30)

| Force | Number | What it means for the design |
|---|---|---|
| `PlaceOrder` latency | p95 ≤ 500 ms; about 300 ms today | Anything new on the request path has about 200 ms of headroom |
| Peak load | 200 orders/s (11.11 sale) | |
| MailFast latency | p95 1.5 s, p99 8 s | If every order waits for MailFast, `PlaceOrder` p95 is at least 1.5 s, three times the budget |
| MailFast rate limit | 50 emails/s | At peak, 150 emails/s more than the limit allows. The backlog grows by 9,000 emails per minute of peak, and each minute of peak takes at least 3 minutes to drain (9,000 ÷ 50/s) |
| MailFast failures | about 0.5 % | About 0.25 failed calls/s at 50 emails/s, so retries are routine |
| Loyalty points | within 1 min, exactly once | Needs durable delivery, retries **and** duplicate suppression. Loyalty must not queue behind the email backlog |
| Availability | an order must never fail because email, loyalty or analytics is down | Post-order actions must not affect whether the order succeeds |
| Team and operations | 4 developers, strong in Go and gRPC; 2 have used RabbitMQ; Kubernetes; open-source, self-hosted preferred | Favours a familiar self-hosted broker and rules out cloud-only managed queues |

**Problem:** how should `checkoutservice` trigger post-order actions so that checkout stays fast and available, MailFast's limits are respected, loyalty points are awarded exactly once within a minute, and new actions can be added without changing checkout each time?

## Decision drivers

1. **Availability:** an order must succeed even when MailFast, `loyaltyservice` or analytics is down.
2. **Performance:** `PlaceOrder` p95 stays ≤ 500 ms at 200 orders/s, which leaves about 200 ms of headroom.
3. **Reliability of loyalty:** points are awarded within 1 minute, exactly once.
4. **Integration with MailFast:** never send more than 50 emails/s, and cope with its 8 s p99 and 0.5 % failures through retries.
5. **Modifiability (extensibility):** SMS, fraud scoring and recommendations can be added without editing `PlaceOrder` each time.
6. **Team fit and cost:** Go and gRPC skills, existing RabbitMQ experience, Kubernetes, open-source and self-hosted.

## Considered options

1. **Synchronous fan-out:** extend `PlaceOrder` to call MailFast (through `emailservice`), `loyaltyservice` and analytics over gRPC with timeouts, as it calls email today.
2. **In-process async:** `checkoutservice` sends its response and runs the post-order calls in background goroutines. No new infrastructure.
3. **`OrderPlaced` event on RabbitMQ (self-hosted):** checkout publishes to one exchange, and each consumer (email to MailFast, loyalty, analytics) has its own durable queue.
4. **`OrderPlaced` event on Apache Kafka (self-hosted):** checkout publishes to one topic, and each consumer reads it with its own consumer group.

Cloud-only managed queues are left out because the company prefers self-hosted components (`docs/scenario.md` L30).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync fan-out | No new infrastructure. Pure gRPC, which the team knows well. Easy to trace and debug | With MailFast on the request path, p95 is at least 1.5 s. A timeout short enough to fit the ~200 ms headroom would cut off more than 5 % of emails. 200 orders/s cannot be sent at 50/s, so calls get throttled or rejected. Loyalty retries would run inside the request. Every new action means editing `checkoutservice` (L66–123, L230–280) and adds latency |
| 2. In-process async (goroutines) | No new infrastructure. MailFast latency leaves the response path. Small code change | Work held in memory is lost when a pod restarts or crashes, so loyalty points can be missed, which breaks "exactly once". The 50/s limit applies to all replicas together, but each replica would throttle on its own. The 11.11 backlog (9,000 emails per minute) would sit in memory. Checkout stays coupled to every action |
| 3. RabbitMQ events | One fast publish on the request path. Durable queues survive consumer outages. Separate queues mean the email backlog cannot delay loyalty. The email consumer can throttle to 50/s, retry and dead-letter. A new action only needs a new queue binding, with no change to checkout. Open source, runs on Kubernetes, and 2 of the 4 developers have used it | A new stateful component to run (cluster, storage, upgrades). Delivery is at-least-once, so consumers must handle duplicates. Publishing after payment is a dual write: the event is lost if checkout crashes or the broker is down at that moment. Async flows are harder to trace. The event schema becomes a contract between services |
| 4. Kafka events | Durable, replayable log, which suits analytics and a later recommendations update. Very high throughput. Consumer groups keep consumers independent | The scenario gives the team no Kafka experience. Heavier to run than RabbitMQ for a 200 msg/s peak. Per-message retry, delay and dead-lettering would have to be built, for example with extra retry topics **[Inferred]**. The same duplicate and dual-write issues as option 3 |

**Fit against the drivers**

| Driver | 1. Sync | 2. Goroutines | 3. RabbitMQ | 4. Kafka |
|---|---|---|---|---|
| Orders don't fail because of post-order actions | Partly (only if every failure is ignored) | Yes | Yes (publish failure must not fail the order) | Yes (same rule) |
| p95 ≤ 500 ms | **No** | Yes | Yes, if the publish fits in ~200 ms (its latency is not evidenced) | Same as 3 |
| Loyalty within 1 min, exactly once | Partly | **No** (lost on crash) | Yes, with an idempotent consumer | Yes, with an idempotent consumer |
| MailFast 50/s | **No** | **No** (each replica throttles alone) | Yes | Yes |
| Extensibility | **No** | **No** | Yes | Yes |
| Team fit and operating cost | Best | Good | Good | Weakest |

## Decision

Choose **option 3: `checkoutservice` publishes one `OrderPlaced` event to a self-hosted RabbitMQ cluster, and each post-order action is a separate consumer with its own durable queue.**

Design outline:

- **Producer.** In `PlaceOrder`, replace the `sendOrderConfirmation` gRPC call (`main.go` L273–277, L379–384) with a publish of `OrderPlaced` to exchange `orders`, routing key `order.placed`. Use publisher confirms and a short timeout that fits in the ~200 ms headroom. **If the publish fails, checkout logs it and counts it but never returns the error.** Today's email failure follows the same rule (L274). One broker address replaces `EMAIL_SERVICE_ADDR` and `emailSvcConn` (L81–82, L115, L122).
- **Event.** It carries `order_id` (L233), `user_id`, `email`, the order `total` (computed at L243–250 but missing from `OrderResult` today), the `OrderResult` fields (L265–271), a timestamp and a `schema_version`. Encode it with Protocol Buffers so it can reuse the existing `pb` types (L34). **[Inferred]** This suits a gRPC team, but the evidence doesn't require it.
- **Queues.** Each consumer gets one durable, replicated queue with a dead-letter queue: `email`, `loyalty` and `analytics`. A future action such as SMS or a recommendations update adds only a queue and a consumer.
- **Email consumer.** This replaces the mock `emailservice` logic. It sends at most 50 emails/s to MailFast in total across replicas. It retries the ~0.5 % failures with backoff, uses a timeout longer than MailFast's 8 s p99, and acknowledges a message only after MailFast accepts it. The queue holds the 11.11 backlog, about 9,000 emails per minute of peak.
- **`loyaltyservice`.** It reads the `loyalty` queue and awards points in one transaction keyed on a unique `order_id`, so redelivered or duplicate events are ignored. That makes it *effectively* exactly once: delivery is at-least-once and processing is idempotent. It must handle at least 200 events/s at peak to meet the 1-minute target.
- **Analytics consumer.** It writes the event to the marketing team's store. That store is not specified (see Open questions).

**Why:** option 3 is the only option that meets drivers 1–5 together and also fits the team:

- Option 1 misses the latency budget (p95 at least 1.5 s) and the MailFast rate limit.
- Option 2 loses loyalty events when a pod restarts and cannot enforce a single 50/s limit across replicas.
- Option 4 meets the drivers but costs more to run and to learn. Two developers already know RabbitMQ, and it fits the open-source, self-hosted, Kubernetes constraints (`docs/scenario.md` L29–30).

## Consequences

- **Positive:** `PlaceOrder` no longer waits for email. It adds only one publish (quality attribute: performance).
- **Positive:** orders succeed while MailFast, `loyaltyservice` or analytics is down. Their messages wait in the queues and are processed when the consumer recovers (quality attribute: availability).
- **Positive:** the queue smooths the 200/s peak down to MailFast's 50/s, and retries plus dead-lettering handle the 0.5 % failures (quality attribute: reliability).
- **Positive:** separate queues isolate the consumers, so the email backlog cannot hold up loyalty points (quality attributes: performance against the 1-minute target, fault isolation).
- **Positive:** adding a post-order action needs no change to `checkoutservice`. Its list of dependencies (L66–86) shrinks instead of growing (quality attribute: modifiability).
- **Positive:** open source and self-hosted on the existing Kubernetes, and it uses the team's RabbitMQ experience (quality attributes: cost, team fit).
- **Negative:** RabbitMQ is a new stateful component that post-order actions depend on. It needs a cluster, persistent volumes, monitoring and upgrades (quality attributes: operability, cost).
- **Negative:** delivery is at-least-once, so every consumer must be idempotent. Duplicate emails are still possible if MailFast times out after it has sent the email (quality attribute: correctness).
- **Negative:** email becomes eventually consistent. At the 11.11 peak, a confirmation can arrive minutes after checkout (quality attribute: user experience).
- **Negative:** a trace stops at the publish unless the trace context (L135–137) is copied into message headers **[Inferred]** (quality attribute: observability).
- **Negative:** the `OrderPlaced` schema is shared between services, so changes need versioning (quality attribute: modifiability).
- **Risks and mitigations:**

| Risk | Mitigation |
|---|---|
| The event is lost between a successful payment and the publish (checkout crashes or the broker is down), so there is no email and no points. For loyalty, a lost event breaks "exactly once". Email already has the same gap today: the call comes after the charge (L252 vs L273) | Run RabbitMQ as a 3-node cluster with replicated (quorum) queues and publisher confirms. Retry the publish a few times within the latency budget, and alert on publish failures. A **transactional outbox** would close the gap fully, but `checkoutservice` has no database (`main.go` L17–43). See Open questions |
| Broker outage | Orders still succeed because a failed publish doesn't fail the order. Post-order actions for that window are delayed or lost (see the row above) |
| Email backlog during a long 11.11 peak | Monitor queue depth and the age of the oldest message. Ask MailFast for a higher limit if the delay is unacceptable |
| Duplicate loyalty points | Unique `order_id` in the `loyaltyservice` store, plus the duplicate-delivery test under Validation |
| A poison message blocks a queue | Move it to the dead-letter queue after N attempts and alert on the dead-letter queue's size |
| First production broker for half the team **[Inferred]** | Pair the two developers who know RabbitMQ with the other two. Deploy with a maintained Kubernetes operator or Helm chart **[Inferred]** |
| Fraud scoring may need to **block** an order, which a post-order event cannot do | Make fraud scoring its own decision when it arrives |

## Validation

- **Load test** at 200 orders/s with the Locust `loadgenerator` (as in ADR-0001, L49). `PlaceOrder` p95 must stay ≤ 500 ms with the publish added. Also measure the p95 of the publish itself.
- **Failure injection** under load: stop the MailFast stub, `loyaltyservice`, the analytics consumer and one RabbitMQ node in turn. No order may fail because of them, and queued messages must be processed after recovery.
- **Loyalty:** at peak, 100 % of points are awarded within 60 s of the order. Redelivering the same event several times awards points only once. A daily check of orders against awarded points shows none missing and no duplicates.
- **MailFast:** sending never goes above 50/s, so there are no rate-limit rejections. A dashboard shows email queue depth, oldest-message age and dead-letter queue size.
- **Review** after the first 11.11 sale that uses this design, or when the first new post-order action is added (date to be confirmed).

## Open questions

1. **Lost events:** is it acceptable to lose an email or loyalty event when `checkoutservice` crashes or RabbitMQ is down at publish time? If not, which store would back a transactional outbox, given that `checkoutservice` has none?
2. **Email delay:** what is the longest acceptable email delay during 11.11, and how long does the 200/s peak last? Both are needed to size the queue and judge the 50/s limit. Can MailFast raise the limit?
3. **Publish latency:** how long does a RabbitMQ publish with confirms take in this cluster? This is not evidenced and must be measured against the ~200 ms headroom.
4. **MailFast API:** does it accept idempotency keys, to avoid duplicate emails on retry? Which errors are safe to retry? What is its mean latency? That number sizes the consumer: by Little's law, 50/s at 1.5 s per call is about 75 calls in flight.
5. **Analytics:** where does the marketing team want "order placed" events stored? May the events contain personal data such as the email address and shipping address?
6. **`loyaltyservice`:** what data store does it use? What is the points rule, and does it need the order total? What happens to points when an order is refunded or cancelled? None of this is given.
7. **Fraud scoring:** is it advisory only, or must it be able to stop an order? If it must stop orders, this design changes.
8. **Deciders and deadline:** who approves this ADR? If the next 11.11 sale is on 2026-11-11, there are about two weeks to build it **[Inferred from today's date]**.
9. **`emailservice`:** its language and current code are not in the provided files. They decide whether the MailFast consumer is a rewrite or a change to the existing service.
