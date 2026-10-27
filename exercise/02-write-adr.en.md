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

| Criterion | Run a | Run b |
|---|---|---|
| Uses scenario numbers? | No; every missing fact listed under Open questions | Yes: 500 ms, 200 orders/s, MailFast p95 1.5 s / p99 8 s / 0.5 % / 50/s |
| Options | direct sync calls; event + broker (chosen); post-order orchestrator | sync gRPC; goroutines in checkout; RabbitMQ (chosen); Kafka |
| Exactly-once points / MailFast limit | generic: idempotent consumers, outbox, DLQ; no rate-limit knowledge | order ID as idempotency key; email consumer ≤ 50/s with retries + DLQ |
| Invented anything? | none found; current-system guesses labelled [Assumption] | verify: `main.go` line refs (111–123, 233, 252–277, 273–277) |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Lost events: replaced "log the failed publish and continue" with a transactional outbox + relay | A lost event means a customer never gets their loyalty points, which breaks the *exactly once* requirement. A RabbitMQ outage now delays events instead of losing them (ADR line 61). |
| 2 | Email rate limit: added single active consumer + a ≤ 50 emails/s token bucket | RabbitMQ has no built-in consumer rate limit, and splitting 50/s across N replicas breaks when the replica count changes (ADR line 63). |
| 3 | Analytics event: only an allow-listed projection, no email or shipping address | Only `emailservice` needs the email address, so the broker and marketing never become a second copy of customer contact data (ADR line 65). |
| 4 | Fixed the backlog maths: drains at 50/s minus the ongoing order rate, not 50/s | New orders keep adding emails after the peak; with a post-peak rate r ≥ 50 orders/s the backlog never drains (ADR line 78). |
| 5 | Removed the made-up "no lost emails" driver and corrected the cart-emptying fact | The scenario never states a no-loss guarantee for emails, and `main.go` line 263 discards the `emptyUserCart` error, so emptying isn't guaranteed (ADR lines 13, 36). |

## Extra: fact-check with Codex

Before revising the ADR, I asked Codex (`GPT-6.1-Sol`) to fact-check the AI draft against the evidence. It only reported; it did not edit the ADR, and all edits above are my own. Output: [`outputs/2-factcheck.md`](outputs/2-factcheck.md) (prompt log row 6).

```
Read AGENTS.md first.

Task: fact-check an ADR draft. Do NOT edit the ADR or any other existing file.

adr: docs/adr/0002-use-rabbitmq-for-post-order-events.md
evidence: docs/scenario.md, system/online-boutique/src/checkoutservice/main.go
output: exercise/outputs/2-factcheck.md

Read only the ADR and the evidence files listed above. Do not open any other file.

For every factual claim in the ADR that points at the evidence (line numbers, function
names, call order in PlaceOrder, numbers from the scenario), write one table row:

| ADR line | Claim | Evidence (file + line/symbol) | Verdict ✅ / ❌ / ❓ | What the evidence actually says |

Rules:
- Only check facts. Do not judge the decision, suggest rewrites, or propose new options.
- If the evidence does not show something, write "Not evidenced in the provided files".
- Also check any arithmetic in the ADR (e.g. email backlog numbers) and show the calculation.

End with a short summary listing every file you read.
```