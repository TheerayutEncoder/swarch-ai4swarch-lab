# ADR-0002: Publish an "order placed" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Online Boutique development team (Senior developer); Theerayut A.

## Context and problem

Requirement R-12 (`docs/scenario.md`) adds three actions after an order is placed: a confirmation email through the external provider **MailFast** (replacing the mock `emailservice` logic), **loyalty points** through a new `loyaltyservice`, and an **"order placed" analytics event** for marketing. Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

How it works today:

- `checkoutservice.PlaceOrder` makes its calls one after another over synchronous gRPC: charge card, ship order, empty cart, send confirmation (`main.go` lines 252-277). `emptyUserCart` is attempted but its error is discarded (line 263), so successful cart emptying is not guaranteed.
- An email failure only logs a warning; the order still succeeds (`main.go` lines 273-277, `log.Warnf("failed to send order confirmation ...")`).
- Existing direct integrations follow that pattern. A mandatory implementation pattern for future actions is Not evidenced in the provided files.
- The order ID is a UUID generated at the start of `PlaceOrder` (`uuid.NewUUID()`, line 233), so each invocation has a unique ID that consumers can use as a de-duplication key. Whether the ID is persisted, or reused when a client retries `PlaceOrder`, is Not evidenced in the provided files.

The facts that force a decision (`docs/scenario.md`, *Quality goals and constraints*):

| Fact | Why it matters |
|---|---|
| `PlaceOrder` p95 must stay **≤ 500 ms** (today ~300 ms) | Only ~200 ms of budget is left |
| MailFast p95 **1.5 s**, p99 **8 s**, ~**0.5 %** failures | A synchronous MailFast call alone breaks the 500 ms budget |
| Peak **200 orders/s** (11.11) vs MailFast limit **50 emails/s** | At peak, emails arrive 4x faster than MailFast accepts them; ~150 emails/s must wait |
| Loyalty points **within 1 minute**, **exactly once** | Needs reliable delivery plus duplicate protection |
| An order must **never fail** because email, loyalty or analytics is down | Post-order actions must be decoupled from the order's success |
| Team: Go + gRPC; **two have used RabbitMQ** | Favours a tool the team already knows |
| Kubernetes; prefer **open-source, self-hosted** | Favours a self-hosted broker over cloud-only managed queues. This is a stated preference; a rule that excludes managed queues is Not evidenced in the provided files. |

The problem: **how should `checkoutservice` trigger post-order actions so that checkout stays fast and never fails because of them, MailFast's rate limit is respected, loyalty points are awarded exactly once, and new actions can be added without changing checkout?**

## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms at 200 orders/s.
3. **Reliability / correctness:** loyalty points exactly once, within 1 minute; confirmation emails sent despite MailFast's 0.5 % failures and 50 emails/s limit. An explicit no-loss guarantee or acceptable loss rate for emails is Not evidenced in the provided files.
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
| 3. RabbitMQ event | Checkout only waits for a local write (outbox) or publish; each queue buffers work (absorbs the 200/s vs 50/s gap); per-consumer retries and dead-letter queues; new actions just add a queue (modifiability); open-source, runs on Kubernetes; two developers know it | New stateful component to run and monitor; delivery is at-least-once, so `loyaltyservice` must de-duplicate; publishing after the charge is a dual write, so reliable delivery needs a transactional outbox and a datastore for `checkoutservice` (see Decision) |
| 4. Kafka event | Durable, replayable log (useful for analytics and future consumers like recommendations); very high throughput; open-source | More to operate (brokers, partitions, consumer groups) for a 4-person team with no stated Kafka experience; 200 events/s does not need Kafka's scale; per-message retry / delay is less direct than RabbitMQ's; still at-least-once, so de-duplication is still needed |

## Decision

Choose **option 3: `checkoutservice` publishes an `OrderPlaced` event to a self-hosted RabbitMQ**, and each post-order action consumes it independently.

- **Transactional outbox:** after `shipOrder` succeeds and `emptyUserCart` has been attempted, `PlaceOrder` writes one `OrderPlaced` row (order ID from line 233, email, items, total, shipping tracking ID) to an outbox table in a small datastore owned by `checkoutservice`, then returns. It no longer talks to RabbitMQ on the request path. This replaces the direct `sendOrderConfirmation` gRPC call (lines 273-277). If `shipOrder` fails, `PlaceOrder` returns before this point (lines 258-261) and no event is written.
- A separate **outbox relay** publishes unsent rows to a fanout/topic exchange with publisher confirms, and marks a row sent only after the broker confirms it. A RabbitMQ outage therefore delays events instead of losing them. Why not log-and-continue: a lost event means a customer never gets their loyalty points, which breaks the *exactly once* requirement (driver 3). The relay can publish a row twice (crash after confirm, before marking), so delivery is at-least-once and consumers de-duplicate on the order ID.
- If the outbox write itself fails, `PlaceOrder` logs it with the order ID, raises an alert and **still returns success**: the card is already charged, so failing the order would be worse (driver 1). This, and a pod crash between the charge and the outbox write, is the remaining loss window (see Risks).
- **Email:** `emailservice` consumes its own queue and calls MailFast, acknowledging each message only after MailFast accepts it. The queue is declared with **single active consumer** (`x-single-active-consumer`), so only one replica consumes at a time and other replicas are hot standbys. That replica runs a local token bucket capped at **≤ 50 emails/s** and keeps enough sends in flight to reach that rate despite MailFast's 1.5 s p95 (**[Inferred]** ~75-100 concurrent sends, prefetch set to match). RabbitMQ has no built-in consumer rate limit, and splitting 50/s across N replicas breaks when the replica count changes. Failures are retried with exponential backoff through TTL retry queues; after N attempts the message goes to a dead-letter queue that is alerted on and replayed manually. Delivery is at-least-once, so a retry after a MailFast timeout can send a duplicate email.
- **Loyalty:** `loyaltyservice` consumes its own queue and treats the **order ID as an idempotency key** (store each processed order ID; ignore duplicates), turning at-least-once delivery into exactly-once effect.
- **Analytics:** a consumer records only an allow-listed projection of the event for marketing: order ID, timestamp, product IDs and quantities, total and currency. It drops the customer email and shipping address before anything is stored. Only `emailservice` needs the email address, so the broker and marketing never become a second copy of customer contact data (privacy).
- **Future actions** (SMS, fraud scoring, recommendations) bind a new queue to the same exchange; `checkoutservice` does not change.

Why: it is the only option that meets drivers 1-4 together. Options 1 and 2 fail the latency or reliability numbers. Kafka (option 4) would also work, but RabbitMQ fits the load (200 events/s), the team's experience and the self-hosted preference with less operational cost. RabbitMQ also gives per-queue retry, dead-lettering and single active consumer out of the box. The outbox works with either broker, so it doesn't favour Kafka. Kafka's main advantage, replaying past events, only matters if marketing needs it (see Open questions); if they do, revisit this decision.

## Consequences

- **Positive:** MailFast's 1.5 s p95 / 8 s p99 no longer sits on the checkout path; `PlaceOrder` adds only one local outbox write (performance). The write cost is Not evidenced in the provided files; **[Inferred]** a few milliseconds, to be measured (see Validation).
- **Positive:** email, loyalty, analytics and even RabbitMQ outages no longer affect orders; events wait in the outbox or in their queues until the component recovers (availability).
- **Positive:** the email queue absorbs the 200/s peak and MailFast is never called faster than 50/s, so the rate limit is respected instead of causing failures (reliability).
- **Positive:** new post-order actions are new subscribers, not changes to `checkoutservice` (modifiability).
- **Negative:** RabbitMQ is a new stateful component to deploy, monitor and upgrade on Kubernetes (operability, cost).
- **Negative:** `checkoutservice` gains state: an outbox datastore and a relay process to run, back up and monitor (operability, cost). Today it is stateless.
- **Negative:** confirmation emails become eventually consistent. During an 11.11 peak the email backlog grows by ~150 emails for every second of peak. Afterwards it drains at **50/s minus the ongoing order rate**, not at 50/s, because new orders keep adding emails. With a backlog *B* and a post-peak rate *r* < 50 orders/s, draining takes *B* / (50 − *r*) seconds; if *r* ≥ 50 it never drains (user-perceived latency of email).
- **Negative:** with single active consumer, email throughput does not grow with `emailservice` replicas, and a failover to a standby pauses sends briefly (performance, availability of email). This is acceptable because MailFast caps throughput at 50/s anyway.
- **Negative:** consumers must be idempotent and the event schema must be versioned, which adds design and testing work (modifiability, correctness).
- **Risks and mitigations:**
  - *Lost event:* the outbox removes the broker from this risk, but a window remains: the card is charged (an external call, outside the outbox transaction) and then the pod crashes or the outbox write fails before commit, so the order gets no email or points. Mitigate by alerting on failed outbox writes and reconciling payment transactions against outbox rows. Closing the window fully would mean writing a pending-order row *before* the charge, which is out of scope here.
  - *Outbox relay stalls:* events pile up unsent and loyalty misses its 1-minute target. Mitigate by alerting on the age of the oldest unsent row.
  - *Customer data in the event:* the broker carries email and shipping address. Restrict queue access per consumer and keep the analytics projection allow-listed (see Decision).
  - *Duplicate emails:* MailFast times out but actually sent the email, and the retry sends it again. Mitigate with a MailFast idempotency key if supported (see Open questions); otherwise accept rare duplicates.
  - *Duplicate loyalty points:* redelivery after a consumer crash. Mitigate with the order-ID idempotency key and a unique constraint in `loyaltyservice`'s store.
  - *Loyalty delayed beyond 1 minute:* give `loyaltyservice` its own queue so the slow email backlog never blocks it; alert on queue age.
  - *RabbitMQ unavailable:* checkout does not touch the broker, so orders keep succeeding; the relay retries until the broker is back. Use a replicated RabbitMQ cluster (e.g. quorum queues) so queued messages survive a node failure.
  - *Poison messages / MailFast outages:* dead-letter queue and alerting.

## Validation

- Load test `PlaceOrder` at **200 orders/s** with RabbitMQ in place: p95 must stay **≤ 500 ms**.
- Under the same load, with MailFast stubbed at its p95/p99 latency, 0.5 % failure rate and 50/s limit: no order fails, no email is silently dropped (every email is either sent or in the dead-letter queue), and the email backlog drains after the peak.
- Loyalty: measure order-to-points time (target: **≤ 1 minute** at peak) and inject duplicate deliveries to prove points are awarded **exactly once**.
- Chaos test: stop `emailservice`, `loyaltyservice`, the analytics consumer and (separately) RabbitMQ during the load test; orders must still succeed, and every outbox row must be delivered once the component is back.
- Kill the outbox relay between the broker confirm and marking the row sent; the duplicate must not award points twice.
- Kill the active `emailservice` consumer; a standby must take over and the send rate must never exceed 50/s.
- Check that the analytics store contains no email addresses or shipping addresses.
- Monitor queue depth, oldest-message age and dead-letter count in production; review this ADR after the first 11.11 sale.

## Open questions

- How long does the 11.11 peak of 200 orders/s last, and what is the order rate just after it? Together they set the email backlog size and how long it takes to drain (backlog grows by ~150 emails/s of peak and drains at 50/s minus the post-peak rate).
- What is the acceptable delay for a confirmation email? Not given.
- Which datastore holds the `checkoutservice` outbox? None is evidenced in the provided files.
- Is the remaining loss window (pod crash between the charge and the outbox write) acceptable, or must a pending-order row be written before the charge?
- Which analytics fields does marketing actually need? The allow-list in the Decision is a starting point, not a confirmed requirement.
- Does MailFast support an idempotency key, to avoid duplicate emails when a send is retried after a timeout?
- Where does `loyaltyservice` store processed order IDs and points balances? Its datastore is not specified.
- Where must analytics events go (which tool or store), and do marketing need replay of past events? Replay needs would favour Kafka (option 4).
- What availability and durability does RabbitMQ itself need (number of nodes, persistent volumes)? Who on the team operates it?
- Should fraud scoring (planned next year) stay post-order, or must it run *before* the charge? A pre-charge check could not use this asynchronous path.
