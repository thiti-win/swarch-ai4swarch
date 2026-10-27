**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | the ADR's author (Sippakorn Thunyahan) | defends the ADR |
| **AI analyst** | Arin Ariyata | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Apimook Sarovath | role card P (business side) |
| **Site reliability engineer (SRE)** | Thitiwut Ariyaprayoon | role card S (operations side) |
| **security & privacy officer** | Akaporn Ungsittipoonporn | role card X |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-post-order-processing.md` by Sippakorn
**Board members and roles:** Sippakorn (Architect), Arin (AI analyst), Apimook (Product owner), Thitiwut (SRE), Akaporn (Security & privacy officer)

**Decision:** **accept with changes**

**Required changes:**
1. **Durable state and database:** specify the durable order state, database choice and crash-recovery behaviour before implementation, and meet the ops rule for new stateful components (dashboards, alerting, backups, runbook; about 1 week) before 11.11.
2. **Alerting and traceability:** define the alert that fires when MailFast stops working (threshold in minutes and who receives it), and a way to look up one order's email status (query, dashboard or trace ID linking checkout to the task).
3. **Privacy:** get legal approval for MailFast under PDPA before 11.11, or switch to an in-country email provider as a fallback. Analytics events carry only `order_id` (no email or name); the task table is encrypted at rest and records, including quarantined tasks, are deleted within 30 days.
4. **Loyalty business rules:** show new points on the order confirmation page immediately, as marketing promised, or agree a change with the business first. Define how points are reversed on cancel/refund, and keep idempotency so retries never award points twice.
5. **Delivery plan and fraud scoring:** a build and test plan, including a load test at 200 orders/s, that fits in the 3 weeks before 11.11. State that future fraud scoring must be able to reject an order before confirmation, not run as a post-order background action.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | MailFast stores data in the US; PDPA needs a legal basis for cross-border transfer, legal hasn't approved it, and the ADR has no fallback | Security | No | No: MailFast's data location and approval status aren't in the scenario or the ADR |
| 2 | Analytics events must not contain customer email or name (company policy), but the ADR leaves the payload undefined | Security | No | Couldn't know the policy, but the ADR itself says the payload is undefined, so the AI could have flagged a privacy risk |
| 3 | Tasks holding customer emails must be encrypted at rest and kept ≤ 30 days, but the ADR doesn't mention encryption and forbids discarding pending tasks | Security | Partly: mentions data retention criteria, not encryption | Couldn't know the 30-day limit or encryption rule, but the conflict with "never discard tasks" can be inferred from the ADR |
| 4 | New stateful component needs dashboards, alerting, backups and a runbook (about 1 week) before production; database still "to be selected" | SRE | Partly: flags the unselected database, not the ops readiness work | Saw the missing database choice in the ADR; couldn't know the company's ops rule or the 1-week effort |
| 5 | Last year's 20-minute email outage went unnoticed; ADR has no alert thresholds or recipient | SRE | Partly: verdict asks to define "escalation ownership" | Could infer that alerting is vague; couldn't know about last year's outage |
| 6 | Can't trace one order across the async boundary ("I never got my email") | SRE | No | Partly: the vague "central view of action outcomes" in the ADR could have been flagged as an observability gap |
| 7 | Team's RabbitMQ experience is laptop-only; an HA broker would need 3 nodes and 2 weeks of budget approval, which supports avoiding a broker | SRE | No | No: the scenario only says "two have used RabbitMQ", not that it was never in production |
| 8 | Loyalty "within 1 minute" doesn't match marketing's promise of "see your points immediately" | PO | No | No: the marketing promise isn't in the scenario or the ADR |
| 9 | No rule for reversing points on cancel/refund | PO | No | Partly: a domain-aware reviewer could ask, but the real business rule isn't in the files |
| 10 | Duplicate points cost the company real money; idempotency must be guaranteed | PO | Yes: non-risk on atomic loyalty dedup, plus a replay scenario | Knew the mechanism from the scenario's "exactly once", but not the money value of points |
| 11 | Can it be built and load-tested in the 3 weeks before 11.11? | PO | Partly: verdict says to run load/crash/outage tests before claiming compliance, but gives no timeline | No: the 3-week deadline isn't in the scenario or the ADR |
| 12 | Fraud scoring must happen before confirmation, so it can't be a post-order background action | PO | No (fraud is only mentioned as a future coupling concern) | Partly: R-12 lists fraud scoring as a post-order action, but general domain knowledge says fraud checks usually block orders |
| 13 | Stable order identity and payment/shipping reconciliation are still open; a crash before the order is saved could leave a paid/shipped order with no recoverable work, or repeat effects | AI review | Yes (Risk) | Yes: derived from the ADR's own open questions and the order flow |
| 14 | Checkout and the growing email backlog share the same durable storage; a long MailFast outage could fill it and stop checkout | AI review | Yes (Risk) | Yes: reasoning from the ADR's design and the scenario's MailFast limits |
| 15 | If `loyaltyservice` is down for more than 60 s, "orders never fail" and "points within 1 minute" can't both hold; needs a business decision | AI review | Yes (QA scenario 3 and verdict item 2) | Yes: a contradiction between two constraints in the scenario |

**The AI's best finding** (one the humans had missed): Checkout and the growing email backlog share the same durable storage, so a long MailFast outage could fill the storage and stop checkout itself. This quietly breaks the "an order must never fail because email is down" requirement, and none of the stakeholders raised it.
**The AI's weakest finding** (generic, wrong or irrelevant): "Source-code claims citing `checkoutservice/main.go` are Not evidenced in the provided files." This isn't a flaw in the design; the review prompt simply didn't give the AI `main.go`. Its final verdict item 3 ("set measurable budgets and run the validation") is also generic advice that applies to any ADR.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
