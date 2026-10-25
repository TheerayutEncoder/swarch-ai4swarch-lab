> Agent: Claude Code (Anthropic) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Task 2, run b (with context)

# ADR-0002: Publish an "order placed" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Online Boutique development team (4 developers); names to be confirmed

## Context and problem

Requirement R-12 (`docs/scenario.md`) adds three actions after an order is placed: a confirmation email through the external provider **MailFast** (replacing the mock `emailservice` logic), **loyalty points** through a new `loyaltyservice`, and an **"order placed" analytics event** for marketing. Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

How it works today:

- `checkoutservice.PlaceOrder` charges the card, ships the order, empties the cart and then calls `emailservice` **synchronously over gRPC**, in sequence (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`, lines 252-277).
- An email failure only logs a warning; the order still succeeds (`main.go` lines 273-277, `log.Warnf("failed to send order confirmation ...")`).
- Every downstream address is a required env var and a gRPC connection built at startup (`mustMapEnv` / `mustConnGRPC`, lines 111-123). Each new post-order action done this way means another field in `checkoutService`, another env var and another call in `PlaceOrder`.
- The order ID is a UUID generated at the start of `PlaceOrder` (`uuid.NewUUID()`, line 233), so a stable key exists for de-duplication.

The facts that force a decision (`docs/scenario.md`, *Quality goals and constraints*):

| Fact | Why it matters |
|---|---|
| `PlaceOrder` p95 must stay **≤ 500 ms** (today ~300 ms) | Only ~200 ms of budget is left |
| MailFast p95 **1.5 s**, p99 **8 s**, ~**0.5 %** failures | A synchronous MailFast call alone breaks the 500 ms budget |
| Peak **200 orders/s** (11.11) vs MailFast limit **50 emails/s** | At peak, emails arrive 4x faster than MailFast accepts them; ~150 emails/s must wait |
| Loyalty points **within 1 minute**, **exactly once** | Needs reliable delivery plus duplicate protection |
| An order must **never fail** because email, loyalty or analytics is down | Post-order actions must be decoupled from the order's success |
| Team: Go + gRPC; **two have used RabbitMQ** | Favours a tool the team already knows |
| Kubernetes; prefer **open-source, self-hosted** | Rules out cloud-only managed queues as the default |

The problem: **how should `checkoutservice` trigger post-order actions so that checkout stays fast and never fails because of them, MailFast's rate limit is respected, loyalty points are awarded exactly once, and new actions can be added without changing checkout?**

## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms at 200 orders/s.
3. **Reliability / correctness:** loyalty points exactly once, within 1 minute; no lost emails despite MailFast's 0.5 % failures and 50 emails/s limit.
4. **Modifiability:** add SMS, fraud scoring and recommendations next year without editing `PlaceOrder` each time.
5. **Operability and team fit:** open-source, self-hosted on Kubernetes; a small team (4 developers) with Go/gRPC skills and some RabbitMQ experience.

## Considered options

1. **Synchronous gRPC fan-out:** extend `PlaceOrder` to call MailFast (via `emailservice`), `loyaltyservice` and an analytics service in sequence, as it calls `emailservice` today.
2. **In-process asynchronous calls:** `PlaceOrder` returns, then fires the three calls from goroutines inside `checkoutservice`, with in-memory retries.
3. **Publish an `OrderPlaced` event to RabbitMQ (self-hosted):** `checkoutservice` publishes one event; `emailservice` (MailFast adapter), `loyaltyservice` and an analytics consumer each subscribe through their own queue.
4. **Publish an `OrderPlaced` event to Apache Kafka (self-hosted):** same shape as option 3, using a Kafka topic with one consumer group per action.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC fan-out | Simplest; no new infrastructure; matches today's code and the team's gRPC skills | MailFast p95 1.5 s / p99 8 s alone breaks the 500 ms p95; at 200 orders/s checkout hits MailFast's 50/s limit; no buffering, so emails fail during peaks; `checkoutservice` must know every new action (low modifiability) |
| 2. In-process async (goroutines) | Removes MailFast latency from the response; no new infrastructure | Pending work lives in memory: lost on pod restart or crash, so no "exactly once" or "within 1 minute" guarantee; rate limiting across several `checkoutservice` replicas is hard; still couples checkout to every action |
| 3. RabbitMQ event | Checkout only waits for a local publish; each queue buffers work (absorbs the 200/s vs 50/s gap); per-consumer retries and dead-letter queues; new actions just add a queue (modifiability); open-source, runs on Kubernetes; two developers know it | New stateful component to run and monitor; delivery is at-least-once, so `loyaltyservice` must de-duplicate; publishing after the charge is a dual write (see Risks) |
| 4. Kafka event | Durable, replayable log (useful for analytics and future consumers like recommendations); very high throughput; open-source | More to operate (brokers, partitions, consumer groups) for a 4-person team with no stated Kafka experience; 200 events/s does not need Kafka's scale; per-message retry / delay is less direct than RabbitMQ's; still at-least-once, so de-duplication is still needed |

## Decision

Choose **option 3: `checkoutservice` publishes an `OrderPlaced` event to a self-hosted RabbitMQ**, and each post-order action consumes it independently.

- After `shipOrder` succeeds and the cart is emptied, `PlaceOrder` publishes one `OrderPlaced` message (order ID from line 233, email, items, total, shipping tracking ID) to a fanout/topic exchange, with publisher confirms. It replaces the direct `sendOrderConfirmation` gRPC call (lines 273-277).
- If the publish fails, `PlaceOrder` still returns success and logs the failure, as it does for email failures today (driver 1). See Risks for how lost events are handled.
- **Email:** `emailservice` becomes a consumer that calls MailFast, limited to **≤ 50 emails/s** across all its replicas, retrying failures with backoff and sending repeated failures to a dead-letter queue.
- **Loyalty:** `loyaltyservice` consumes its own queue and treats the **order ID as an idempotency key** (store each processed order ID; ignore duplicates), turning at-least-once delivery into exactly-once effect.
- **Analytics:** a consumer records the event for marketing.
- **Future actions** (SMS, fraud scoring, recommendations) bind a new queue to the same exchange; `checkoutservice` does not change.

Why: it is the only option that meets drivers 1-4 together. Options 1 and 2 fail the latency or reliability numbers. Kafka (option 4) would also work, but RabbitMQ fits the load (200 events/s), the team's experience and the self-hosted preference with less operational cost.

## Consequences

- **Positive:** MailFast's 1.5 s p95 / 8 s p99 no longer sits on the checkout path; `PlaceOrder` adds only a local publish (performance). **[Inferred]** the publish cost is a few milliseconds; to be measured (see Validation).
- **Positive:** email, loyalty or analytics outages no longer affect orders; messages wait in their queues until the consumer recovers (availability).
- **Positive:** the email queue absorbs the 200/s peak and drains at MailFast's 50/s, so the rate limit is respected instead of causing failures (reliability).
- **Positive:** new post-order actions are new subscribers, not changes to `checkoutservice` (modifiability).
- **Negative:** RabbitMQ is a new stateful component to deploy, monitor and upgrade on Kubernetes (operability, cost).
- **Negative:** confirmation emails become eventually consistent; during an 11.11 peak they can be delayed. A backlog grows by ~150 emails per second of peak and drains at 50/s after it (user-perceived latency of email).
- **Negative:** consumers must be idempotent and the event schema must be versioned, which adds design and testing work (modifiability, correctness).
- **Risks and mitigations:**
  - *Lost event (dual write):* the card is charged, then the publish fails or the pod crashes before it, so no email/points. Mitigate with publisher confirms plus retry; if stronger guarantees are needed, use a transactional outbox, which needs a datastore for `checkoutservice` (none is evidenced in `main.go`; see Open questions).
  - *Duplicate loyalty points:* redelivery after a consumer crash. Mitigate with the order-ID idempotency key and a unique constraint in `loyaltyservice`'s store.
  - *Loyalty delayed beyond 1 minute:* give `loyaltyservice` its own queue so the slow email backlog never blocks it; alert on queue age.
  - *RabbitMQ unavailable:* checkout must not block. Mitigate with a short publish timeout and a replicated RabbitMQ cluster (e.g. quorum queues).
  - *Poison messages / MailFast outages:* dead-letter queue and alerting.

## Validation

- Load test `PlaceOrder` at **200 orders/s** with RabbitMQ in place: p95 must stay **≤ 500 ms**.
- Under the same load, with MailFast stubbed at its p95/p99 latency, 0.5 % failure rate and 50/s limit: no order fails, no email is lost (all sent or in the dead-letter queue), and the email backlog drains after the peak.
- Loyalty: measure order-to-points time (target: **≤ 1 minute** at peak) and inject duplicate deliveries to prove points are awarded **exactly once**.
- Chaos test: stop `emailservice`, `loyaltyservice`, the analytics consumer and (separately) RabbitMQ during the load test; orders must still succeed.
- Monitor queue depth, oldest-message age and dead-letter count in production; review this ADR after the first 11.11 sale.

## Open questions

- How long does the 11.11 peak of 200 orders/s last? This sets the email backlog size and the maximum email delay (backlog grows by ~150 emails/s of peak).
- What is the acceptable delay for a confirmation email? Not given.
- Is losing a post-order event (charge succeeded, event never published) acceptable at a low rate, or is a transactional outbox required? If required, which datastore would `checkoutservice` use? None is evidenced in the provided files.
- Does MailFast support an idempotency key, to avoid duplicate emails when a send is retried after a timeout?
- Where does `loyaltyservice` store processed order IDs and points balances? Its datastore is not specified.
- Where must analytics events go (which tool or store), and do marketing need replay of past events? Replay needs would favour Kafka (option 4).
- What availability and durability does RabbitMQ itself need (number of nodes, persistent volumes)? Who on the team operates it?
- Should fraud scoring (planned next year) stay post-order, or must it run *before* the charge? A pre-charge check could not use this asynchronous path.
