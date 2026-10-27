**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Theerayut Attajak / 6970142021
**Agent(s) used:** Claude Code (one fresh subagent per run, started from a main session); Codex for the fact-check
**Model(s):** Claude Opus 5.5 (`claude-opus-5-5`); Codex: `GPT-6.1-Sol`
**Plan / tier:** Claude Code: Pro; Codex: Plus


Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-…` run A, evidence: demo.proto | AGENTS.md, prompts/1-recover-architecture.md, system/online-boutique/protos/demo.proto | Yes for reads. ⚠️ ran one `ls` on exercise/outputs/ (breaks "no terminal commands"); self-reported, not mentioned in the output file | | 9 services, 0 dependencies; "Checkout calls 6 services" marked [Inferred] |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-…` run B, evidence: demo.proto + kubernetes-manifests/ | prompts/1-recover-architecture.md, demo.proto, kubernetes-manifests/: kustomization.yaml, README.md, 11 service YAMLs | Yes. No commands | | 12 containers, 17 deps via *_ADDR env vars; found manifests via kustomization.yaml (couldn't list folder); C4 diagram renders tangled |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-…` run a, context: none, R-12 | AGENTS.md, prompts/2-write-adr.md, docs/adr/template.md | Yes. Did NOT open docs/scenario.md. No commands | | Broker chosen, product left open; no invented numbers; flagged PII |
| 4 | 2-b | Claude Code + Opus 5.5 | `prompts/2-…` run b, context: scenario + ADR 0001 + checkoutservice/main.go, R-12 | AGENTS.md, prompts/2-write-adr.md, docs/adr/template.md, docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go | Yes. No commands | | RabbitMQ chosen over sync gRPC, goroutines, Kafka; uses scenario numbers + main.go line refs |
| 5 | 3 | Claude Code + Opus 5.5 | `prompts/3-…` adr: docs/adr/0002-use-rabbitmq-for-post-order-events.md | AGENTS.md, prompts/3-review-adr.md, docs/adr/0002-use-rabbitmq-for-post-order-events.md, docs/scenario.md | Yes (prompt allows the ADR + scenario.md). ADR not modified. No commands | | Verdict: accept with changes. 4 Risks, 3 Sensitivity, 3 Trade-off, 3 Non-risk |
| 6 | 2 fact-check | Codex + `GPT-6.1-Sol` | fact-check ADR 0002 vs scenario.md + checkoutservice/main.go → exercise/outputs/2-factcheck.md | <fill from Codex's report> | <yes/no; did it touch the ADR?> | | Fact-check only; edits done by student |

