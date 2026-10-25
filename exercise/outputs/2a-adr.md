Agent: Claude Code (Claude Agent SDK) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Publish an "order placed" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not given in the requirement (see Open questions)

## Context and problem

After an order is placed, Online Boutique must do three new things (from the requirement):

1. Send the order confirmation email through **MailFast**, a new external email provider. It replaces the mock `emailservice` logic.
2. Award **loyalty points** through a new `loyaltyservice` (still to be built).
3. Record an **"order placed" analytics event** for the marketing team.

Marketing also expects **more post-order actions next year**: SMS, fraud scoring, and a recommendations update.

The problem: how should the component that places the order trigger these actions, now and as more are added?

This ADR was written from the requirement alone. No system files were read, so the current order flow, the protocol between services, the existing infrastructure and the team's skills are **not evidenced**. Where the text below names how the order is placed today, it is labelled **[Assumption]**.

- **[Assumption]** One service places the order (the "order-placing service" below) and today calls `emailservice` synchronously as part of that flow.
- The requirement does not say whether any of the three actions must finish before the customer sees the order confirmation.

## Decision drivers

Most important first. The ordering is my reading of the requirement and should be confirmed.

- **Modifiability / extensibility:** three more post-order actions are expected next year (requirement). Adding one should not need a change to the order-placing service.
- **Availability of order placement:** an outage at MailFast (external), in the new `loyaltyservice` or in analytics should not make placing an order fail. **[Inferred]** from the fact that these are follow-up actions, not part of taking payment; to be confirmed.
- **Reliability of each action:** an email, a loyalty award or an analytics event should not be silently lost if a consumer is down. Exactly what loss is acceptable is not given.
- **Integrability with an external provider:** MailFast is outside our control; its API, rate limits and SLA are not given.
- **Performance (checkout latency):** extra post-order work should not add to the time the customer waits. No latency target is given.
- **Cost and operability:** a new piece of infrastructure must be run and monitored. Budget and team skills are not given.

## Considered options

1. **A: Synchronous calls from the order-placing service.** After the order is placed, the order-placing service calls the email adapter (MailFast), `loyaltyservice` and the analytics sink directly, one after another or in parallel.
2. **B: Publish an `OrderPlaced` event to a message broker (publish/subscribe).** The order-placing service publishes one event per order. Each action is a separate subscriber: an email consumer (MailFast adapter, replacing the mock `emailservice` logic), `loyaltyservice`, and an analytics consumer. New actions next year become new subscribers.
3. **C: A dedicated post-order orchestrator service.** The order-placing service makes one call to a new orchestrator, which then calls email, loyalty and analytics itself (synchronously or with retries).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous calls | Simplest; no new infrastructure. Easy to trace one request end to end. The caller knows at once whether each action succeeded. | Order-placing service is coupled to every action and must change for each new one (SMS, fraud scoring, recommendations). Checkout latency grows with each call. A MailFast or `loyaltyservice` outage either fails the order or needs custom retry/skip logic in the order path. |
| B: Event + broker | Order-placing service publishes once and does not know the consumers: new actions are added without touching it (modifiability). Consumers fail and retry independently; the broker buffers events while one is down (availability, reliability). Checkout waits only for the publish (performance). Analytics gets a natural event stream. | New infrastructure to run, secure and monitor (cost, operability). Eventual consistency: the email or the points arrive a little later and the customer may not see points immediately. Harder to debug across asynchronous hops; needs correlation IDs. Need to handle duplicate delivery (idempotent consumers) and to make sure the event is published if the order commits (e.g. outbox pattern). Event schema becomes a contract that must be versioned. |
| C: Orchestrator | Order-placing service is decoupled from the individual actions. One place holds the post-order workflow, ordering and retry policy; easy to see what should happen. No broker needed if calls are synchronous. | Every new action still changes the orchestrator, so the coupling has only moved. If the call to the orchestrator is synchronous, its availability and latency are still in the checkout path. The orchestrator can become a central "god service". Fan-out and retries must be built by hand. |

## Decision

Choose **Option B: the order-placing service publishes an `OrderPlaced` domain event to a message broker, and each post-order action subscribes to it.**

- Email: a consumer that sends through MailFast replaces the mock `emailservice` logic. Whether this is a changed `emailservice` or a new adapter is an open question.
- Loyalty: the new `loyaltyservice` subscribes and awards points.
- Analytics: a consumer records the "order placed" event for marketing.
- Next year's SMS, fraud scoring and recommendations update are added as new subscribers.

Why: it best meets the top two drivers. The requirement already expects the list of actions to grow, and B is the only option where adding an action leaves the order-placing service unchanged. It also keeps the external MailFast and the new `loyaltyservice` out of the checkout path, so their failures do not fail an order.

The broker product (e.g. RabbitMQ, Kafka, a cloud pub/sub) is **not** chosen here: the hosting platform, expected volume, retention needs and team skills are not given. It should be chosen in a follow-up ADR.

**Caveat on fraud scoring:** if fraud scoring must *block* a suspicious order before it is confirmed, it cannot be an after-the-fact subscriber and would need a synchronous step before the order is placed. The requirement lists it as a "post-order action", so it is treated as asynchronous here; this needs confirming.

## Consequences

- **Positive:** New post-order actions are added as subscribers without changing or redeploying the order-placing service. (quality attribute: modifiability)
- **Positive:** A MailFast, `loyaltyservice` or analytics outage does not fail order placement; events wait in the broker until the consumer recovers. (quality attribute: availability, reliability)
- **Positive:** Checkout only waits for the publish, not for email, loyalty and analytics. (quality attribute: performance)
- **Positive:** Each consumer scales and is deployed on its own. (quality attribute: scalability, deployability)
- **Negative:** A message broker must be provisioned, secured, monitored and backed up. (quality attribute: cost, operability)
- **Negative:** Email and loyalty points are eventually consistent; the customer may see them seconds later. (quality attribute: consistency, usability)
- **Negative:** Following one order across asynchronous consumers is harder. (quality attribute: observability, testability)
- **Risks and mitigations:**
  - *Event lost between the order commit and the publish* → use a transactional outbox or equivalent so the event is published if and only if the order is stored.
  - *Duplicate delivery (at-least-once)* → consumers are idempotent, keyed by order ID (no double email, no double points).
  - *A consumer keeps failing* → retries with backoff and a dead-letter queue, with an alert.
  - *MailFast rate limits or outage* → the email consumer throttles and retries; MailFast limits and SLA must be confirmed (Open questions).
  - *Event schema changes break subscribers* → version the `OrderPlaced` schema and only make backward-compatible changes.
  - *Personal data (email address, order details) in events sent to analytics/marketing* → decide what the event carries and who may subscribe.

## Validation

- **Fitness check for modifiability:** add a dummy subscriber (e.g. a stub SMS consumer) and confirm that no change to the order-placing service was needed.
- **Failure test:** stop the email consumer, MailFast (or a stub of it) and `loyaltyservice` during a load test; orders must still be placed, and the backlog must drain once they recover with no lost or duplicated emails/points.
- **Latency check:** compare checkout latency before and after the change under the same load. The target value is not given (Open questions).
- **Metrics:** publish failures, consumer lag, dead-letter queue size, and MailFast delivery errors, with alerts.
- **Review date:** before adding the first of next year's actions. Exact date not given.

## Open questions

- Who are the deciders for this ADR?
- How does the system place an order today, which service does it, and how does it call `emailservice`? (No system files were read for this run.)
- What protocols and infrastructure does the system already use? Is a message broker already available, and on what hosting platform?
- Expected order volume and peaks? Needed to size and choose the broker.
- Is there a checkout latency target, and an availability target for order placement?
- Must any action finish before the customer sees the confirmation (e.g. email sent, points shown)? How late may the email and points be?
- Is losing an analytics event acceptable? Is losing an email or a loyalty award acceptable?
- MailFast: API type, authentication, rate limits, SLA, cost per email, and whether it reports delivery status (webhooks)?
- Should MailFast sending replace the logic inside the existing `emailservice`, or should a new adapter replace `emailservice`?
- Who owns `loyaltyservice` and where does it store points? What are the rules for awarding points (and reversing them on cancel/refund)?
- Where does marketing want analytics events delivered (which tool or store), and in what format?
- Must fraud scoring block or cancel orders? If it must block, it needs a synchronous step before the order is confirmed.
- Team skills with message brokers and the operations budget for new infrastructure?
- Privacy rules for customer data in events shared with marketing?
