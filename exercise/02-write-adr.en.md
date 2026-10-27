**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

> AI-assisted: this comparison was drafted with Claude Code (Opus 5.5) in a separate session from runs 2a and 2b.

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | **No.** It had none, and it didn't make any up: volume, latency target and MailFast's SLA and limits are all in its Open questions (Q3, Q4, Q6). | **Yes, all of them:** p95 ≤ 500 ms (about 300 ms today), 200 orders/s, MailFast 1.5 s / 8 s / 0.5 % / 50 per s, loyalty within 1 min, the team and the self-hosted preference. It also worked out new numbers from them: p95 ≥ 1.5 s if checkout waits for MailFast, about 200 ms of headroom, an email backlog of 9,000 per minute of peak, and at least 3 minutes to drain each minute of peak. I checked the arithmetic and it is correct. |
| Options considered (list them) | A. Synchronous direct calls. B. Publish/subscribe events, with no broker chosen. C. A post-order orchestrator service. Chose **B**. | 1. Synchronous gRPC fan-out. 2. In-process background goroutines. 3. RabbitMQ events. 4. Kafka events. Chose **3 (RabbitMQ)**, with a table scoring each option against each driver. |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | **Only in general terms.** Delivery is at-least-once, so consumers must be idempotent, using the order ID as the key. The MailFast consumer "can be throttled", but no number or method is given. | **Concretely.** `loyaltyservice` gets its own durable queue, so it can't wait behind the email backlog. It awards points in one transaction keyed on a unique `order_id`, so duplicates are ignored ("effectively exactly once"). The email consumer sends at most 50/s across all replicas, retries failures, and uses a timeout longer than the 8 s p99. It also points out that an event can be lost if checkout crashes between charging the card and publishing (the dual write), and leaves that as Open question 1. |
| Anything invented (a number, tool or fact not in the scenario)? | **No numbers invented.** But it suggests "Prefer a managed broker" (risk table), which goes against the scenario's self-hosted preference. It couldn't have known about that preference without the context. Its [Inferred] guess that checkout calls email synchronously turned out to be right (`main.go` L273). | **Mostly marked [Inferred]:** Protobuf encoding, a Kubernetes operator or Helm chart, and "about two weeks to 11.11" worked out from today's date. **Not marked:** the 3-node quorum-queue cluster, which is a design choice rather than a fact. Also, its Validation plans "a daily check of orders against awarded points", but there is no order database to compare against (`checkoutservice` has no DB, as its own Context table says). |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

> The decisions in rows 2–4 are mine. Claude Code (Opus 5.5) helped write them into the ADR and this table.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Removed the AI's "Agent" line and draft note, and named myself as Decider. | It is my ADR now, and I am the one accountable for it. |
| 2 | **Added a transactional outbox** in a small self-hosted PostgreSQL. Checkout writes the order and its `OrderPlaced` event in one local transaction, and a relay publishes it to RabbitMQ. The AI's version published straight from `PlaceOrder`. | The AI's draft named the lost-event risk but left it as an open question. With a direct publish, if RabbitMQ is down or checkout crashes at that moment, the event is lost and the customer never gets loyalty points, which breaks R-12. With the outbox, a broker outage only delays events. PostgreSQL is open source and self-hosted, so it fits the operations constraint. The cost is a second stateful component, which I added to the consequences. |
| 3 | **Removed** "a daily check of orders against awarded points" from Validation, and added an outbox failure test instead. | There was no order database to check against: `checkoutservice` has no database (`main.go` L17–43), as the AI's own Context table says. The check couldn't be run as written. |
| 4 | **Specified a token-bucket rate limiter** at 50 emails/s, with only one active email consumer and a standby. | The draft only said "at most 50/s across replicas" without saying how. If each replica throttled on its own, two replicas could send 100/s and hit MailFast's limit. One active consumer means one limiter covers every email, and retries wait for a token too. |
| 5 | Removed the AI's estimate that 11.11 is "about two weeks" away. | The scenario never says when the sale is. The AI worked it out from today's date. |
