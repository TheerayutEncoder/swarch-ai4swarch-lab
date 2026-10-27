**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| `cartservice` → `redis-cart` | `cartservice.yaml` `REDIS_ADDR=redis-cart:6379` | ✅ | `src/cartservice/src/Startup.cs:29` reads `Configuration["REDIS_ADDR"]` |
| `frontend` → `shippingservice` (direct) | `frontend.yaml` `SHIPPING_SERVICE_ADDR` | ✅ | `src/frontend/main.go:137,145` maps the env var and dials it; `src/frontend/rpc.go:88` calls `ShippingService.GetQuote` directly (not only via checkout). The [Inferred] label "Gets quotes" is right |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` `PRODUCT_CATALOG_SERVICE_ADDR` | ✅ | `src/recommendationservice/recommendation_server.py:131-133` requires the env var; line 73 calls `ListProducts`. The [Inferred] label "Lists products" is right |
| `checkoutservice` → 6 services | `checkoutservice.yaml` six `*_SERVICE_ADDR` | ✅ | `src/checkoutservice/main.go:111-116`: shipping, product catalog, cart, currency, email, payment |
| `loadgenerator` → `frontend` (HTTP) | `loadgenerator.yaml` `FRONTEND_ADDR` | ✅ | `src/loadgenerator/locustfile.py:35-50` sends HTTP GET/POST to `/`, `/product/…`, `/cart`, `/setCurrency` |
| `checkoutservice` → `emailservice` on 5000 → 8080 | `emailservice.yaml` port 5000, `targetPort: 8080` | ✅ | `src/emailservice/email_server.py:129` listens on `PORT` default `8080`, matching `targetPort` |
| "6 services have no outgoing dependencies" | absence of `*_ADDR` env vars | ✅ with a caveat | No business calls found. Only optional observability: e.g. `src/currencyservice/server.js:50-61` and `src/paymentservice/index.js:34-47` connect to `COLLECTOR_SERVICE_ADDR` when `ENABLE_TRACING=1`, which the default manifests don't set |
| `frontend` → `shoppingassistantservice` (left out) | not drawn; `frontend.yaml:83-84` sets the address | ❌ (missing dependency in code) | `src/frontend/main.go:139` makes `SHOPPING_ASSISTANT_SERVICE_ADDR` **required**; the `/bot` route is always registered (`main.go:163`) and `chatBotHandler` POSTs to it over **HTTP**, not gRPC (`handlers.go:464`). Source exists in `src/shoppingassistantservice/`. Not deployed by default (`ENABLE_ASSISTANT` commented out, `frontend.yaml:93-94`), so the call would fail at runtime |

### What's wrong or missing in Run B (from the source)

- **Missing dependency:** `frontend → shoppingassistantservice` over HTTP exists in code (above). Leaving it out is defensible for the default *deployment*, but the diagram doesn't say so.
- **Missing data stores:** `productcatalogservice` loads `products.json` (`src/productcatalogservice/catalog_loader.go:45-47`); `currencyservice` reads `data/currency_conversion.json` (`src/currencyservice/server.js:110`). Both are bundled files, not databases, but they are where the data lives.
- **External systems correctly absent:** `emailservice` only runs `DummyEmailService`, which logs and returns (`email_server.py:108-122`; non-dummy mode raises "not implemented"). `paymentservice` returns a random UUID as the transaction ID (`src/paymentservice/charge.js:85`). Both are mocks, so having no `System_Ext` was right.
- **Languages (all "not evidenced" in 1B):** Go (frontend, checkout, product catalog, shipping), C# (cart), Python (recommendation, email, loadgenerator/Locust, shopping assistant), Node.js (currency, payment), Java (ad).

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
From demo.proto alone the agent found the 9 gRPC services but no dependencies at all, because a .proto only declares what each service offers, not who calls it. It also could not see the frontend (it has no gRPC contract), the redis-cart data store, or any language, since those live in deployment config and source code. Even Run B, with the Kubernetes manifests, still missed things only the source shows, so an agent can't recover the full architecture without reading the code.

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
The missing frontend → shoppingassistantservice dependency. The diagram suggests frontend only calls backend services over gRPC, but the code makes SHOPPING_ASSISTANT_SERVICE_ADDR required (src/frontend/main.go:139) and calls it over HTTP from the /bot endpoint (handlers.go:464), even though the service isn't deployed by default. A new team member trusting the diagram wouldn't know this dependency exists, and would be surprised when the chatbot fails or when they try to deploy or change the frontend.
