Agent: Claude Code (running as a sub-agent session) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: 2a (zero-shot, `context: none`)

# ADR-0002: Publish an `OrderPlaced` event and handle post-order actions as independent subscribers

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Not specified in the requirement (see Open questions)

## Context and problem

**Evidence base.** I wrote this draft from the requirement text alone (`context: none`). I did not read any Online Boutique source code, manifests or documentation. Every statement about the current system either quotes the requirement or is marked **[Inferred]** or "Not evidenced in the provided files".

**What the requirement states**

| # | Post-order action | Integration | Source |
|---|---|---|---|
| 1 | Send the order confirmation email | **MailFast**, a new *external* email provider. It replaces the mock `emailservice` logic | Requirement, item 1 |
| 2 | Award loyalty points | New `loyaltyservice` (to be built) | Requirement, item 2 |
| 3 | Record an "order placed" analytics event | For the marketing team. The destination system is not named | Requirement, item 3 |
| Future | SMS, fraud scoring, recommendations update | "more post-order actions next year" | Requirement, last line |

**Current state**

- Which service places the order, and how it calls `emailservice` today: Not evidenced in the provided files.
- **[Inferred]** In the public Online Boutique demo, `checkoutservice` places the order and calls `emailservice` synchronously over gRPC after payment and shipping. Check this against `system/online-boutique/` before accepting this ADR.
- Whether orders are stored anywhere (an order database): Not evidenced in the provided files.

**Numbers.** The requirement gives no order volume, latency target, email-delivery deadline or MailFast SLA (see Open questions).

**Problem.** The order flow must trigger a growing set of post-order actions. One of them depends on a third-party provider (MailFast), whose availability and latency we do not control. How do we trigger these actions so that each new one does not require changing the order-placement path or add risk to it?

## Decision drivers

The ordering below is my **[Inferred]** reading of the requirement. Stakeholders should confirm it.

1. **Modifiability / extensibility:** at least three more post-order actions are expected next year (requirement, last line). Adding one should not require changing the code that places orders.
2. **Availability of order placement:** if MailFast (external), `loyaltyservice` or analytics is down or slow, order placement should still succeed. **[Inferred]** from "after an order is placed". Confirm the business rule.
3. **Reliability of post-order actions:** once an order is placed, every action must eventually happen, with no lost emails or points. Retries must be safe, so duplicates must be harmless.
4. **Performance (checkout latency):** post-order work should not add time to the customer-facing response. Target not given.
5. **Cost and operability:** new infrastructure must be worth what it costs to run. Budget and team skills are not given.
6. **Observability:** we need to see, for each order, which actions succeeded.

## Considered options

1. **A: Synchronous direct calls (extend the current flow).** The service that places the order calls the MailFast adapter, `loyaltyservice` and the analytics sink, one after another or in parallel, before it responds.
2. **B: Event-driven publish/subscribe (choreography).** The service that places the order publishes one `OrderPlaced` event to a message broker. Each action is an independent subscriber: email (MailFast), `loyaltyservice` and analytics now; SMS, fraud scoring and recommendations later.
3. **C: Dedicated post-order orchestrator.** The service that places the order hands the order to a new orchestrator service, through a durable queue or a single call. The orchestrator owns the workflow, calls each action with retries and tracks the status of each order.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Sync direct calls | Simplest option, with no new infrastructure (cost). The call trace is immediate and easy to follow. The caller knows at once whether each action succeeded | Checkout latency includes every action, including the external MailFast call (performance). Any failing dependency can fail or stall checkout unless every call has a timeout and a fallback (availability). Each new action means changing and redeploying the order service (modifiability). Retries must be built by hand, so emails or points can be lost silently (reliability) |
| B: Pub/sub events | A new action is a new subscriber, with no change to the order service (modifiability). Checkout depends only on the broker publish, not on MailFast, loyalty or analytics (availability). Post-order work runs off the request path (performance). The broker buffers messages and each consumer retries on its own, so consumers scale and fail independently. Analytics can consume the same event stream | A new broker to run, secure and pay for (cost, operability). Delivery is at-least-once, so consumers must be idempotent, e.g. no double points (reliability). Results are eventually consistent: the email and points arrive after a delay. The end-to-end flow is implicit and harder to trace (observability). Ordering between actions, such as fraud scoring before awarding points, is awkward in pure choreography. The event schema becomes a contract that must be versioned |
| C: Orchestrator | The whole post-order workflow is visible in one place. Ordering and dependencies are explicit, which suits a future fraud check that gates other actions. Retries, compensation and per-order status live in one place (observability, reliability). The order service is decoupled from the individual actions | A new service to build and own, plus a durable queue or state store (cost). Every new action still means changing the orchestrator, so the coupling moves rather than disappears (modifiability). The orchestrator is a central point of failure and a possible bottleneck unless it is made highly available (availability). It risks gathering business logic that belongs to other services |

## Decision

**Chosen: Option B.** Publish an `OrderPlaced` event and build each post-order action as an independent subscriber.

**Why it best meets the drivers**

- **Driver 1 (modifiability):** next year's SMS, fraud scoring and recommendations actions can be added as new subscribers without touching order placement. Options A and C both require changing a central component for every new action.
- **Driver 2 (availability):** order placement depends only on publishing to the broker, not on MailFast. Option A would put an external provider on the critical path.
- **Drivers 3 and 4 (reliability, performance):** the broker absorbs MailFast outages and slowdowns through retries and a backlog. Checkout latency no longer includes post-order work.

**Shape of the decision** (details to be refined in follow-up ADRs)

| Element | Proposal |
|---|---|
| Event | `OrderPlaced`, carrying the order ID (also the idempotency key), a timestamp, the customer contact, line items and totals. Exact fields are an open question because of privacy |
| Email subscriber | The `emailservice` role becomes a MailFast adapter that consumes `OrderPlaced`, replacing the mock logic (requirement, item 1) |
| Loyalty subscriber | The new `loyaltyservice` consumes `OrderPlaced` and awards points once per order ID |
| Analytics subscriber | A consumer or broker sink forwards the event to the marketing team's analytics destination (not named; open question) |
| Delivery semantics | At-least-once. Every consumer is idempotent. Failed messages are retried with back-off, then moved to a dead-letter queue |
| Not decided here | The broker product (e.g. Kafka, RabbitMQ or a managed cloud pub/sub), to be chosen in a follow-up ADR once volume, budget, hosting and team skills are known. How to make publishing reliable when the publish fails after the order is placed, e.g. a transactional outbox if an order store exists (Not evidenced in the provided files) |
| Escape hatch | If future actions depend on each other (e.g. fraud scoring must clear an order before points or email), add a second event such as `OrderVerified`, or a small orchestrator for that sub-flow (Option C), rather than going back to synchronous calls |

## Consequences

- **Positive:** adding a post-order action needs only a new subscriber. The order service stays unchanged (quality attribute: modifiability).
- **Positive:** an outage of MailFast, `loyaltyservice` or analytics no longer fails order placement. Their work queues up and resumes later (quality attribute: availability).
- **Positive:** the checkout response time no longer includes the email, loyalty or analytics calls (quality attribute: performance, latency).
- **Positive:** each consumer scales and deploys on its own, and the MailFast consumer can be throttled to respect MailFast's rate limits (quality attributes: scalability, deployability).
- **Negative:** new broker infrastructure to provision, monitor, secure and pay for (quality attributes: cost, operability).
- **Negative:** the email and points become eventually consistent. A customer may see the order confirmation page before the email or points arrive (quality attributes: usability, data consistency).
- **Negative:** messages can be delivered more than once, so every consumer must be idempotent (quality attributes: reliability, development effort).
- **Negative:** the flow is spread implicitly across services, so debugging needs distributed tracing and correlation IDs (quality attributes: observability, testability).
- **Risks and mitigations:**

| Risk | Mitigation |
|---|---|
| The order is placed but the event publish fails, so the event is lost | Retry the publish from a local buffer. Use a transactional outbox if an order store exists (to be verified). Add a reconciliation job |
| Double loyalty points or duplicate emails | Use the order ID as the idempotency key in `loyaltyservice` and the email consumer. Use MailFast's own idempotency support, if it has any (open question) |
| A poison message blocks a consumer | Bounded retries, a dead-letter queue and an alert |
| MailFast outage or rate limiting | Consumer back-off and a circuit breaker. The backlog drains after recovery. Alert on consumer lag |
| Personal data (customer email, address) reaches every subscriber, including analytics | Keep the event payload minimal. Send analytics pseudonymised data. Control access to topics (quality attributes: security, privacy) |
| An event schema change breaks consumers | A versioned schema, backward-compatible changes only, and contract tests |
| Future fraud scoring must gate other actions | Staged events, or an orchestrator for that sub-flow (see Decision) |
| The team lacks experience operating a broker | Prefer a managed broker. Write a runbook and provide training. Team skills are not given |

## Validation

- **Modifiability scenario:** add a dummy fourth subscriber (e.g. an SMS stub) in a test environment. Pass: no code change to, and no redeploy of, the service that places orders.
- **Availability fault injection:** block MailFast or stop `loyaltyservice` during a load test. Pass: the order-placement success rate is unchanged, and every queued email and points award is processed after recovery, with none lost.
- **Idempotency test:** deliver the same `OrderPlaced` event N times. Pass: exactly one email and one points award.
- **Performance:** compare checkout latency (p50, p95, p99) before and after the change. The target is not given and must be agreed.
- **Operational metrics:** consumer lag, dead-letter queue size, time from order placed to email sent and to points awarded, and duplicate rate. Set alert thresholds once SLAs are known.
- **Review:** revisit this ADR when the first of next year's actions (SMS, fraud scoring, recommendations) is designed, especially to check fraud-scoring dependencies.

## Open questions

1. **Deciders:** who owns and approves this decision?
2. **Current flow:** which service places the order, and does it call `emailservice` synchronously? Is there an order store? (Not evidenced in the provided files; verify in `system/online-boutique/`.)
3. **Volume:** what are the average and peak orders per second, and the expected growth?
4. **Latency:** what is the checkout latency target (SLO)?
5. **Business rules:** must order placement fail if the email or points step fails? What is the longest acceptable delay for the email, and before points are visible to the customer?
6. **MailFast:** what is its API style, SLA, rate limit, idempotency-key support, cost per email, sandbox environment and data-residency terms?
7. **`loyaltyservice`:** what are the points rules, and must points be visible immediately? Who will own it, and with what technology?
8. **Analytics:** which destination system, which fields, real-time or batch delivery, and how long is data retained? Can marketing read directly from the broker?
9. **Infrastructure:** what hosting platform is used? Is a message broker or managed pub/sub already available?
10. **Budget:** what can be spent on new infrastructure and on MailFast?
11. **Team skills:** what experience does the team have operating message brokers and event-driven systems?
12. **Privacy and compliance:** which data-protection rules apply to customer contact data in events and in analytics?
13. **Future dependencies:** must fraud scoring run before the email or the points award? Do any future actions need a fixed order?
14. **`emailservice`:** should it be refactored into the MailFast adapter, or retired and replaced by a new consumer?
15. **Delivery guarantee:** is "never more than one email" a hard rule that at-least-once delivery cannot meet without MailFast-side deduplication?
