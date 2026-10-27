**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | the ADR's author | defends the ADR |
| **AI analyst** | | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | | role card P (business side) |
| **Site reliability engineer (SRE)** | | role card S (operations side) |

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

**ADR reviewed:** `docs/adr/0002-use-rabbitmq-for-post-order-events.md` by Theerayut Attajak
**Board members and roles:** Solo roleplay, allowed by the professor. Theerayut Attajak: architect (rehearsal and PO round). Claude Code (Claude Opus 5.5): AI analyst (read out `exercise/outputs/3-review.md`), product owner (card P) and SRE (card S). The student read the role cards and relayed them to Claude after the AI review was finished. In the SRE round the architect was played by a separate Claude subagent that saw only the ADR, the AI review, the scenario and what the SRE said aloud; the student reviewed its answers.

**Decision:** accept / accept with changes / reject

**Required changes:**
1. **Points display for the printed campaign:** the confirmation page computes and shows the new points total immediately; `loyaltyservice` credits them asynchronously, exactly once. Display and award ship together for 11.11. Refunds publish `OrderRefunded` and reverse the points (ledger entries, de-dup on order ID).
2. **Ship only what can be ready in 3 weeks:** RabbitMQ + MailFast consumer + `loyaltyservice` for the sale, with direct publish and publisher confirms; outbox, analytics and refunds after the sale. Add an Operations section (named owner, paging alerts, dashboards, backups, runbook) as a go/no-go gate; request budget for a 3-node broker now, with a single durable node + post-sale reconciliation as the accepted fallback.
3. **Make failures visible and recoverable:** circuit breaker on MailFast so an outage pauses the consumer instead of filling the DLQ; DLQ only for poison messages, with documented replay; order ID + trace context on every event and log line so support can trace "I never got my email". Plus end-to-end idempotency (key per checkout attempt, so a retry can't double-charge or double-award).

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Marketing already printed "see your points the moment you order", but points arrive within a minute | PO | No | No: the campaign isn't in the scenario or the ADR |
| 2 | Points are worth money (100 = 10 THB); refunds must reverse them | PO | No | No: the value of points and refunds aren't in the scenario |
| 3 | The sale is in 3 weeks; everything must be built and load-tested by then | PO | No | No: the date isn't in the scenario |
| 4 | Fraud scoring must cancel an order before confirmation | PO (architect had flagged it at ADR line 113) | No | Partly: the ADR raised the question, but "must cancel" is the PO's knowledge |
| 5 | Nobody has run a broker in production ("used RabbitMQ" = laptops only); on-call is 2 people | SRE | Partly: the review noted RabbitMQ ownership was open | No: the scenario says two developers "have used RabbitMQ", which the AI reasonably took as relevant experience |
| 6 | Ops rule: each new stateful component needs dashboards, alerting, backups and a runbook (~1 week) | SRE | Partly: the review noted checkout becomes stateful | No: an internal ops policy, not written anywhere the AI read |
| 7 | Cluster near node limit; a 3-node HA broker needs 2-week budget approval | SRE | No | No: infrastructure capacity and budget process aren't documented |
| 8 | Last 11.11 a 20-minute email outage went unnoticed; what happens to stuck messages, replay, DLQ? | SRE | Partly: DLQ and retries were in the ADR; the review didn't test a long provider outage | No: last year's incident isn't in the scenario |
| 9 | Tracing one order across the async boundary when a customer says "no email" | SRE | No | Yes in principle: observability is a standard async concern the review could have raised |
| 10 | A retried `PlaceOrder` gets a new order ID: double points (and a double charge) | AI review | Yes (points only) | Yes: visible from the ADR; the double charge needs `checkoutservice/main.go` |


**The AI's best finding** (one the humans had missed): a retried `PlaceOrder` creates a new order ID, so the order-ID de-dup doesn't stop double loyalty points. The architect had only considered broker redelivery.
**The AI's weakest finding** (generic, wrong or irrelevant): it treated the "two developers have used RabbitMQ" fact as a strength (Non-risk: "fits … the two developers' experience"), when in reality it was laptop-only experience. That isn't the AI's fault, but it shows a plausible-sounding finding resting on an unchecked fact.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
