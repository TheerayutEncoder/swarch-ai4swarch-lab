Agent: Claude Code (Claude Agent SDK subagent) · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-R001: Use synchronous gRPC with one shared `demo.proto` contract for service-to-service communication

- **Status:** Reconstructed
- **Date:** 2026-10-27 (reconstruction date; original decision date not evidenced in the provided files)
- **Deciders:** Not evidenced in the provided files. Copyright headers name Google LLC (`demo.proto` line 1, `checkoutservice/main.go` line 1), which shows ownership, not who decided.

## Context and problem

Online Boutique is split into many small services that must call each other to serve one user request.

- The single contract file `system/online-boutique/protos/demo.proto` (package `hipstershop`) defines **9 services** in one file: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`. All RPCs are unary (request/response); there are no `stream` RPCs.
- Messages are shared across services: `Money`, `Address`, `CartItem` and `OrderResult` are used by several services (e.g. `CartItem` in `CartService`, `ShippingService`, `OrderItem`; `Money` in `ProductCatalogService`, `CurrencyService`, `ShippingService`, `PaymentService`, `EmailService`).
- `checkoutservice` orchestrates an order by making **sequential, blocking** gRPC calls to 6 services inside one `PlaceOrder` request (`src/checkoutservice/main.go`, `PlaceOrder` lines 230–280; addresses from `*_SERVICE_ADDR` env vars, lines 111–123).
- `frontend` is the only HTTP entry point (`gorilla/mux` routes, `src/frontend/main.go` lines 149–171) and talks to 7 backends over gRPC (lines 141–147).
- Services are written in different languages. **Not evidenced in the provided files** beyond the two Go services; `checkoutservice` imports its own generated package `.../src/checkoutservice/genproto` (line 34), which suggests per-service code generation from the shared proto **[Inferred]**.

Problem: which inter-service communication style and contract-management approach should the services use?

## Decision drivers

Most important first (the ranking itself is **[Inferred]**; no rationale is written in the evidence):

1. **Interoperability / modifiability** across polyglot services: one typed, language-neutral contract that generates clients and servers.
2. **Simplicity of the request flow:** the shopper waits for the page or the order result, so a request/response model matches the user interaction.
3. **Performance:** low per-call overhead for a request fan-out (frontend → 7 services, checkout → 6 services).
4. **Observability:** trace context propagated across every hop (`otelgrpc` stats handlers on client and server).
5. **Demo/teaching value:** a small, easy-to-read system. **[Inferred]**

## Considered options

1. **A. Synchronous gRPC, one shared `demo.proto` for all services** (chosen, as observed).
2. **B. Synchronous gRPC, one `.proto` per service** (each team owns and versions its own contract).
3. **C. REST/JSON over HTTP** between services (OpenAPI or ad-hoc contracts).
4. **D. Asynchronous messaging** (message broker such as Kafka/RabbitMQ) for the order flow, e.g. publish `OrderPlaced` and let shipping/email consume it.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A. gRPC + one shared proto | One place to see every API; shared types (`Money`, `Address`) defined once; typed stubs in every language; HTTP/2 + protobuf is compact; simple mental model for a demo | Every service is coupled to one file, so any change touches a shared artifact; no per-service versioning (single package `hipstershop`, no `v1`); temporal coupling: callers block and fail when a callee is down |
| B. gRPC + per-service protos | Independent ownership and versioning per service; smaller blast radius for contract changes | Shared types (`Money`, `Address`) must be duplicated or put in a common proto anyway; more files and build wiring |
| C. REST/JSON | Human-readable, easy to debug with curl; no codegen required; universal tooling | Weaker typing unless OpenAPI is maintained; larger payloads and JSON parsing cost; contracts drift more easily across languages |
| D. Async messaging | Decouples availability (shipping/email can be down without failing checkout); natural retries and buffering; better for side effects like email | Extra infrastructure (broker) to run; eventual consistency; harder to return an immediate order result to the user; harder to trace and debug |

## Decision

Use **Option A**: every service exposes a unary gRPC API defined in one shared `protos/demo.proto`, and callers (`frontend`, `checkoutservice`) hold one `grpc.ClientConn` per dependency and call it synchronously.

Why (reconstructed, mostly **[Inferred]**): it gives every language the same typed contract with no extra infrastructure, keeps the call graph easy to follow for a reference/demo application, and matches the request/response shape of a shopper waiting for a page or an order confirmation. Observability is designed in through `otelgrpc` handlers on every connection.

## Consequences

- **Positive:** One typed contract for all services; shared value types (`Money` with ISO-4217 `currency_code`, `units`, `nanos`) are defined once and used consistently. (quality attribute: interoperability, modifiability of a single API view)
- **Positive:** End-to-end tracing works across hops because both client and server use `otelgrpc` stats handlers and a W3C TraceContext propagator. (quality attribute: observability / testability)
- **Positive:** Compact binary protocol with persistent connections created at start-up. (quality attribute: performance) **[Assumption]** for the size/speed claim.
- **Negative:** Checkout availability is the product of 6 downstream services' availability; any failure in cart, catalog, currency, shipping quote or payment fails the whole order. (quality attribute: availability)
- **Negative:** Checkout latency is the **sum** of its sequential calls, and `prepOrderItems` makes 2 calls per cart item (`GetProduct` + `Convert`) in a loop. (quality attribute: performance)
- **Negative:** One shared proto with no version suffix means any contract change is a cross-team coordination point. (quality attribute: modifiability, deployability)
- **Negative:** Payment is charged **before** `ShipOrder`; if shipping fails, `PlaceOrder` returns `Unavailable` with no refund/compensation call. (quality attribute: reliability / data consistency)
- **Risks and mitigations:**
  - No per-call deadlines or retries are visible; `convertCurrency` even uses `context.TODO()` instead of the request `ctx`, dropping the caller's deadline and trace context for that hop. Mitigation: pass `ctx`, add deadlines and retry policy (e.g. gRPC service config or a mesh). **[Inferred]** mitigation
  - Connections use `insecure.NewCredentials()` (no TLS between services). Mitigation: mTLS, e.g. via a service mesh. **[Inferred]** mitigation
  - Side effects that the user doesn't need to wait for are already treated as non-critical in code (`emptyUserCart` error ignored; `sendOrderConfirmation` failure only logged). Mitigation: move them to async messaging (Option D) for those steps only. **[Inferred]**

## Validation

Not evidenced in the provided files. Suggested **[Inferred]**:

- Load test the checkout path and measure p95/p99 latency of `PlaceOrder` against the number of cart items.
- Fault-injection test: stop `shippingservice` or `currencyservice` and confirm the expected `Unavailable`/`Internal` errors, and that no card is charged without a shipment.
- Contract check in CI: regenerate stubs from `demo.proto` for every service and fail the build on breaking changes.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | All service APIs are defined in one file, package `hipstershop` | Evidenced | `protos/demo.proto` lines 17, 23–244 (`package hipstershop`, 9 `service` blocks) |
| 2 | The proto defines 9 services | Evidenced | `protos/demo.proto` (`CartService` … `AdService`) |
| 3 | All RPCs are unary request/response, none streaming | Evidenced | `protos/demo.proto` (no `stream` keyword in any `rpc`) |
| 4 | Message types such as `Money`, `Address`, `CartItem` are shared across services | Evidenced | `protos/demo.proto` `Money` used in `Product.price_usd`, `GetQuoteResponse`, `ChargeRequest`, `OrderResult`; `CartItem` in `Cart`, `GetQuoteRequest`, `OrderItem` |
| 5 | The contract has no version in its package name | Evidenced | `protos/demo.proto` line 17 (`package hipstershop;`) |
| 6 | `checkoutservice` calls 6 services over gRPC | Evidenced | `src/checkoutservice/main.go` lines 111–123 (`mustMapEnv`, `mustConnGRPC`), struct fields lines 66–86 |
| 7 | `checkoutservice` calls are synchronous and sequential in `PlaceOrder` | Evidenced | `src/checkoutservice/main.go` `PlaceOrder` lines 230–280, `prepareOrderItemsAndShippingQuoteFromCart` lines 288–311 |
| 8 | `prepOrderItems` makes 2 calls per cart item | Evidenced | `src/checkoutservice/main.go` `prepOrderItems` lines 339–357 (`GetProduct`, `convertCurrency` in loop) |
| 9 | `frontend` is the HTTP entry point and calls 7 backends over gRPC | Evidenced | `src/frontend/main.go` lines 141–147 (`mustConnGRPC`), 149–171 (`mux` routes, `http.ListenAndServe`) |
| 10 | `frontend` maps a shopping-assistant address but opens no gRPC connection to it | Evidenced | `src/frontend/main.go` line 139 (`SHOPPING_ASSISTANT_SERVICE_ADDR`), no matching `mustConnGRPC`. So "all services use gRPC" is not fully shown; the protocol used for it is Not evidenced in the provided files |
| 11 | Services use per-service generated code from the shared proto | Inferred | `src/checkoutservice/main.go` line 34 imports `.../checkoutservice/genproto`; the generation step itself is not in the evidence |
| 12 | Services are polyglot, so a language-neutral contract was needed | Inferred | Only Go services are in the evidence |
| 13 | gRPC was chosen for interoperability across languages | Inferred | No rationale written in the evidence |
| 14 | Request/response was chosen because the user waits for a result | Inferred | Consistent with `PlaceOrder` returning `OrderResult`, but no written rationale |
| 15 | The system is a demo/reference app, which favoured simplicity | Inferred | Hinted by file name `demo.proto` and module path `microservices-demo` (`checkoutservice/main.go` line 34); motive not stated |
| 16 | Tracing is propagated over every gRPC hop | Evidenced | `otelgrpc.NewServerHandler` (`checkoutservice/main.go` line 139), `otelgrpc.NewClientHandler` (line 216; `frontend/main.go` line 231), TraceContext propagator (`checkoutservice/main.go` lines 135–137) |
| 17 | gRPC/protobuf is more compact and faster than REST/JSON | Assumption | General industry knowledge |
| 18 | Per-service protos allow independent versioning | Assumption | General industry knowledge |
| 19 | REST/JSON is easier to debug but weaker typed | Assumption | General industry knowledge |
| 20 | Async messaging decouples availability but adds a broker and eventual consistency | Assumption | General industry knowledge |
| 21 | Checkout availability depends on all 6 downstreams | Evidenced | `src/checkoutservice/main.go` `PlaceOrder` returns errors on cart/catalog/currency/shipping/payment failure (lines 238–261) |
| 22 | Card is charged before shipping, with no compensation | Evidenced | `src/checkoutservice/main.go` `chargeCard` line 252 then `shipOrder` lines 258–261, no refund call |
| 23 | Email and empty-cart failures are treated as non-critical | Evidenced | `src/checkoutservice/main.go` line 263 (`_ = cs.emptyUserCart`), lines 273–277 (`log.Warnf`) |
| 24 | `convertCurrency` drops the caller's context | Evidenced | `src/checkoutservice/main.go` line 360 (`context.TODO()`) |
| 25 | No per-call deadlines or retries are configured | Evidenced | `mustConnGRPC` (`checkoutservice/main.go` lines 210–220; `frontend/main.go` lines 225–235) sets only credentials and stats handler; the 3 s timeout there is cancelled and not applied to calls |
| 26 | Inter-service traffic is not encrypted | Evidenced | `insecure.NewCredentials()` in both `mustConnGRPC` functions |
| 27 | Proposed mitigations (mTLS, deadlines, async side effects) and validation steps | Inferred | Suggestions; not present in the evidence |
| 28 | Original deciders and date | Inferred | Not evidenced; only Google LLC copyright headers |
