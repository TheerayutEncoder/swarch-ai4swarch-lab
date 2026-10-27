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

1. **Guess labelled Evidenced?** Yes, twice: #16 ("every hop") and #21 ("all 6"). Both are over-generalisations whose counter-example sits in the agent's *own* table (#24, #23). The facts were right; the word "all/every" was the guess.
2. **Wrong belief a new developer could form:** the title says "all services communicate synchronously over gRPC", but `frontend` POSTs to `shoppingassistantservice` over **HTTP** (`src/frontend/handlers.go:464`, outside the evidence). The agent flagged the gap (#10) but the ADR's decision still reads as universal. Or: "tracing works end to end", when currency calls drop the trace.
3. **Code vs original architects:** code shows *what* (one proto, unary calls, sequential checkout, no deadlines). Only the architects know *why*: e.g. #13, was gRPC chosen for multi-language interop, for a demo of gRPC tooling, or for performance? The agent rightly left every "why" as Inferred.
