**Language:** [ภาษาไทย](bonus-recover-decision.md) | English

# Bonus (optional, take-home): Recover an unwritten decision (approximately 15 min)

> **Lecture link:** T5 (Dhar et al. generate decisions from code and diagrams to reduce *knowledge vaporization*), AICH4.
> **Question:** when AI explains *why* a system was built this way, how much of the explanation is evidence and how much is a confident guess?

## Run

Start a new chat or session in your agent and send:

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
```

Want the ADR in Thai? Add a line `language: Thai`.

The agent writes the reconstructed ADR, including its **Claim check** table, to [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md).

## Audit the claim check

Choose **five** claims from the agent's table: at least two it marked *Evidenced* and two it marked *Inferred*. Check each one yourself.

| # | Claim (short) | AI says | You say | Why (file path or reasoning) |
|---|---|---|---|---|
| 3 | All RPCs are unary, none streaming | Evidenced | Evidenced | `protos/demo.proto` contains no `stream` keyword at all |
| 16 | Tracing is propagated over **every** gRPC hop | Evidenced | **Not evidenced** | Propagator + `otelgrpc` handlers are set (`checkoutservice/main.go:135-139, 216`), but the agent's own claim 24 shows `convertCurrency` uses `context.TODO()` (`main.go:360`), which drops the trace context on that hop. "Every" is contradicted by its own evidence |
| 21 | Checkout availability depends on **all 6** downstreams | Evidenced | **Not evidenced** | `PlaceOrder` fails only on cart (GetCart), catalog, currency, shipping and payment = 5. Email failures are only logged (`main.go:273-277`), which the agent's own claim 23 says. 5 of 6, not 6 |
| 12 | Services are polyglot, so a language-neutral contract was needed | Inferred | Inferred (and true) | Correct label: the 3 evidence files are all Go. Outside the evidence, the source confirms it: C# (`src/cartservice`), Python (`src/emailservice`), Node.js (`src/paymentservice`), Java (`src/adservice`). The *"so it was needed"* motive is still a guess |
| 13 | gRPC was chosen for interoperability across languages | Inferred | Inferred | No rationale is written anywhere in the evidence or the code; only the original architects could confirm it |

**Count:** Evidenced according to the agent: **17**. According to you: **15** (#16 and #21 are overclaims).

## Reflect

1. Did the agent ever label a guess as *Evidenced*? Give an example, or explain how you checked that it didn't.

   Yes, twice. Claim #16 says tracing works on every gRPC hop, but `convertCurrency` uses `context.TODO()` (`checkoutservice/main.go:360`), so that call loses the trace. Claim #21 says checkout depends on all 6 services, but an email failure is only logged (`main.go:273-277`), so it is really 5. In both cases the agent's own table (#24 and #23) already contained the counter-example.

2. A new developer reads only this reconstructed ADR. What wrong belief could they form?

   They could believe that every service talks over gRPC. In fact `frontend` calls `shoppingassistantservice` over HTTP (`src/frontend/handlers.go:464`). The agent noticed the address was not a gRPC connection (#10), but the ADR's title and decision still say "all services".

3. What can AI recover from code, and what can only the original architects know? Use one example from your ADR.

   From code, the AI could recover what the system does: one shared proto, only unary calls, a sequential checkout and no deadlines. It could not recover why. For example, #13 says gRPC was chosen for multi-language support, but it could also have been chosen to show off gRPC in a demo or for performance. Only the original architects know, and the agent was right to mark every "why" as Inferred.

_Answers drafted with Claude (Opus 5.5) and reviewed by me._