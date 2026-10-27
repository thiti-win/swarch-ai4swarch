**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name:** Thitiwut Ariyaprayoon
**Student ID:** 6631326521
**Agent(s) used:** Claude Code (VS Code extension). Each run was a separate, fresh Claude Code sub-agent session that was given only the worksheet message and the `AGENTS.md` rules, and was limited to reading the allowed files.
**Model(s):** Claude Opus 5.5 (`claude-opus-5-5`)
**Plan / tier:** Pro

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, Opus 5.5 | `prompts/1-…` run A, evidence `demo.proto` | `prompts/1-recover-architecture.md`, `protos/demo.proto` | Yes | 2 | Found 9 services from the `service` blocks but **0 evidenced dependencies**, so the diagram has no `Rel`. Checkout's links to other services listed under Gaps as [Inferred]. |
| 2 | 1-B | Claude Code, Opus 5.5 | `prompts/1-…` run B, evidence `demo.proto` + `kubernetes-manifests/` | `prompts/1-…`, `protos/demo.proto`, all 12 `kubernetes-manifests/*.yaml` + `kubernetes-manifests/README.md` (listed the folder first) | Yes | 4 | 11 containers (incl. `redis-cart`), 17 dependencies cited by `*_ADDR` env vars. Languages "not evidenced". `loadgenerator` drawn as `System_Ext`. |
| 3 | 2-a | Claude Code, Opus 5.5 | `prompts/2-…` run a, context none, R-12 | `prompts/2-write-adr.md`, `docs/adr/template.md` | Yes (did not open `docs/scenario.md`) | 3 | Chose broker + subscribers, broker product left open. No numbers invented; 15 open questions. |
| 4 | 2-b | Claude Code, Opus 5.5 | `prompts/2-…` run b, context scenario + ADR-0001 + `checkoutservice/main.go`, R-12 | `prompts/2-…`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go` | Yes | 5 | Chose RabbitMQ with one queue per action; compared 4 options; uses the scenario's numbers. 9 open questions. |
| 5 | 3 | Claude Code, Opus 5.5 | `prompts/3-…`, adr `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md` | `prompts/3-review-adr.md`, `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md`, `docs/scenario.md` | Yes (did not edit the ADR) | | Verdict: accept with changes. Main risks: events lost when PostgreSQL is down, payment not covered by the outbox, no timeout on the insert, duplicate emails on consumer failover. Said it couldn't check the ADR's `main.go` citations. |
| 6 | Bonus | Claude Code, Opus 5.5 | `prompts/bonus-…`, decision: sync gRPC + one shared `demo.proto` | `prompts/bonus-recover-decision.md`, `docs/adr/template.md`, `protos/demo.proto`, `src/checkoutservice/main.go`, `src/frontend/main.go` | Yes | 4 | Claim check: 48 claims (agent says 22 Evidenced, 19 Inferred, 7 Assumption). |
| 7 | Worksheets 1–2 | Claude Code, Opus 5.5 | Asked for help filling in Task 1 verification/reflection and the Task 2 a-vs-b comparison | `exercise/outputs/1B-architecture.md`, `2a-adr.md`, `2b-adr.md`, the manifests, `docs/scenario.md`, `docs/adr/0001-…`, and source under `src/` (frontend, checkout, recommendation, cart, productcatalog, shoppingassistant, email, payment, Dockerfiles) | n/a (not an evidence-limited run) | | Separate from the evidence-limited runs above. Sections it drafted are marked "AI-assisted" in the worksheets. |
| 8 | Final ADR | Claude Code, Opus 5.5 | Gave my decisions (outbox with PostgreSQL, remove the daily check, rate limiter) and asked it to write them into `docs/adr/0002-…` | `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md`, `exercise/02-write-adr.en.md` | n/a | | The decisions are mine; the agent wrote them into the ADR and the change table. |
