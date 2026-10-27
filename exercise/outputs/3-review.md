Reviewer: Claude Code agent, model Claude Opus 5.5 (`claude-opus-5-5`)

# Architecture board review: ADR-0002 (RabbitMQ + outbox for post-order events)

Evidence read: `docs/adr/0002-use-rabbitmq-for-post-order-events.md`, `docs/scenario.md`. Line references below are to the ADR.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | Shoppers | 200 `PlaceOrder` calls/s | 11.11 peak; MailFast at p95 1.5 s, p99 8 s | Order completes with only the outbox write added to the path | `PlaceOrder` p95 ≤ 500 ms (today ~300 ms, so the outbox write + datastore must fit in ~200 ms at p95) |
| QA2 Availability | Ops failure | RabbitMQ, `emailservice`, `loyaltyservice` or analytics is down | Normal or peak load | Orders still succeed; events wait in outbox/queues and are delivered on recovery | 0 orders failed because of a post-order action; 0 outbox rows left undelivered after recovery |
| QA3 Correctness | `loyaltyservice` consumer | Same `OrderPlaced` delivered twice (relay crash after confirm, line 61) | Peak load, consumer restart | Second delivery ignored by order-ID key | Points awarded exactly once, ≤ 1 minute after the order |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | The "transactional outbox" (line 60) has no business transaction to join: `checkoutservice` writes no order row, and the charge is an external call. The outbox is really a durable local queue written *after* the charge. | The guarantee the pattern name implies does not exist. Charge succeeded + outbox write failed/crashed (line 62) = order with **no points and no email**, which violates "exactly once" (scenario, *Loyalty points*): it is at-most-once in that window. |
| Risk | On outbox-write failure the ADR only logs and alerts (line 62). If the outbox datastore is down, **every** order during the outage loses its events. | The ADR moves the single point of failure from RabbitMQ to an unnamed datastore (Open questions, line 106) and has no fallback (e.g. direct publish with confirms). |
| Risk | De-duplication is keyed on a UUID generated per `PlaceOrder` call (line 16). A client retry of `PlaceOrder` creates a new ID. | Redelivery is protected, but a retried checkout yields two order IDs, two events and double points; the ADR marks this "Not evidenced" but does not assign it as a risk. |
| Risk | The analytics consumer has no de-duplication (line 65), yet delivery is at-least-once. | Relay re-publishes (line 61) inflate marketing's "order placed" counts. |
| Sensitivity point | Email backlog vs **peak duration**: grows ~150 emails/s (line 78). A 10-minute peak gives ~90,000 queued emails; at a post-peak rate of 30 orders/s it takes 90,000 / 20 = 75 min to drain. | Email delay depends almost entirely on peak length and post-peak rate, neither of which is known (line 104). |
| Sensitivity point | In-flight sends / prefetch on the single active email consumer: 50/s × 1.5 s p95 ≈ 75 (line 63); MailFast's 8 s p99 tail needs more slots to keep 50/s. | Too low a prefetch caps throughput below 50/s and the backlog never drains at the expected rate; too high risks bursts beyond the token bucket after failover. |
| Sensitivity point | Outbox relay poll interval and batch size (line 61). | Loyalty ≤ 1 minute is the sum of relay delay + broker + consumer; a single relay instance stalling (line 83) consumes the whole budget. |
| Trade-off point | Outbox: + reliability (broker outages don't lose events), − performance (new write on the hot path) and − operability (`checkoutservice` becomes stateful, line 77). | Write cost is only **[Inferred]** "a few milliseconds" (line 72) against a ~200 ms budget. |
| Trade-off point | Single active consumer: + correct 50/s cap, − email throughput and availability (failover pause, no scale-out, line 79). | Acceptable only because MailFast is the bottleneck; breaks if a second provider or higher limit is added. |
| Trade-off point | At-least-once email with retries: + no lost confirmations, − duplicate emails after MailFast timeouts (line 85). | Customer-visible; depends on an unconfirmed MailFast idempotency key. |
| Non-risk | MailFast removed from the synchronous path (line 72). | Its p95 1.5 s alone exceeds the 500 ms budget, so options 1 and 2 are correctly rejected. |
| Non-risk | One queue per consumer bound to one exchange (lines 64-66, 87). | The email backlog cannot delay loyalty; SMS/fraud/recommendations add a queue without touching `PlaceOrder`. |
| Non-risk | RabbitMQ over Kafka at 200 events/s. | Fits the load, the two developers' experience and the self-hosted preference; replay need is correctly left as a revisit trigger (line 68). |

## 3. Missing or weak

- **Options not considered:** direct publish with publisher confirms, falling back to the outbox only on broker failure (keeps checkout stateless in the normal path); CDC from the outbox instead of a polling relay; RabbitMQ Streams, which would cover the analytics replay question without switching to Kafka; a shared rate limiter so several email consumers can run instead of a single active consumer.
- **Consequences not stated:** event schema format and versioning are mentioned (line 80) but no choice is made (protobuf would match the team's gRPC skills); outbox cleanup/retention; who owns RabbitMQ on a 4-person team (line 112 leaves it open).
- **Unsupported or likely wrong claims:**
  - "the only option that meets drivers 1-4 together" (line 68) contradicts the same paragraph, which says Kafka "would also work".
  - "turning at-least-once delivery into exactly-once effect" (line 64) holds only for redelivery, not for the loss window or `PlaceOrder` retries.
  - "No email is silently dropped" (Validation, line 94) cannot be tested while the loss window in line 82 exists.

## 4. Verdict: **Accept with changes**

1. **Name the outbox datastore and measure the write at 200 orders/s**, and define what happens when it is unavailable (direct-publish fallback or pending-order row before the charge). Stop calling it "transactional" unless an order row shares the transaction.
2. **Make idempotency end-to-end:** an order ID that survives `PlaceOrder` retries, plus de-duplication in the analytics consumer, not only `loyaltyservice`.
3. **Get the missing numbers before deploying:** 11.11 peak duration, post-peak order rate and an email-delay target; then size the email backlog, prefetch and relay poll interval against the 1-minute loyalty target.
