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

> AI-assisted: these answers were drafted with Claude Code (Opus 5.5) in a separate session from runs 1A and 1B. Every row cites the file and line that was checked.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| `frontend` → `cartservice` | `frontend.yaml` `CART_SERVICE_ADDR` (L73–74) | ✅ | Env var is at L73–74. The code really calls it: `frontend/rpc.go` L58, L63, L68 (`GetCart`, `EmptyCart`, `AddItem`). |
| `frontend` → `checkoutservice` | `frontend.yaml` `CHECKOUT_SERVICE_ADDR` (L79–80) | ✅ | Env var is at L79–80. `frontend/handlers.go` L354 calls `PlaceOrder` over gRPC. |
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` `PAYMENT_SERVICE_ADDR` (L61–62) | ✅ | Env var is at L61–62. `checkoutservice/main.go` L252 → `chargeCard` (L370) calls `Charge`. If it fails, the whole order fails (L253–255). |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` `EMAIL_SERVICE_ADDR` (L63–64) | ✅ | Env var is at L63–64. `main.go` L273 → L380 `SendOrderConfirmation`. Unlike payment, a failure only logs a warning (L274). The diagram can't show this difference. |
| `checkoutservice` → `currencyservice` | `checkoutservice.yaml` `CURRENCY_SERVICE_ADDR` (L65–66) | ✅ | Env var is at L65–66. `main.go` L360 calls `Convert`, but with `context.TODO()` instead of the request's `ctx`. |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` `PRODUCT_CATALOG_SERVICE_ADDR` (L60–61) | ✅ | Env var is at L60–61. `recommendationservice/recommendation_server.py` L131–136 creates a `ProductCatalogServiceStub`. |
| `cartservice` → `redis-cart` | `cartservice.yaml` `REDIS_ADDR` (L50–51) | ✅ | Env var is at L50–51. `cartservice/src/Startup.cs` L29–40 reads `REDIS_ADDR` and uses `RedisCartStore`. |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question):
  - **No false dependencies.** All 17 dependencies match an env var in the manifests, and the 7 checked above also match a real client call in the source code.
  - **Missing: `frontend` → `shoppingassistantservice` over HTTP, not gRPC.** `frontend/handlers.go` L464 sends an HTTP POST to `SHOPPING_ASSISTANT_SERVICE_ADDR` (`frontend.yaml` L83–84). The service's code exists in `src/shoppingassistantservice/` (Python/Flask) and depends on Google Gemini and AlloyDB, which are external cloud systems. No manifest in `kubernetes-manifests/` deploys it, so the AI couldn't confirm it. It listed it under Gaps (gap 3), but the diagram doesn't show it.
  - **Missing: languages.** The AI wrote "language not evidenced" for every service, which was honest for its evidence. The Dockerfiles show Go (`frontend`, `checkoutservice`, `productcatalogservice`, `shippingservice`), C#/.NET (`cartservice`), Node.js (`currencyservice`, `paymentservice`), Python (`emailservice`, `recommendationservice`, `loadgenerator`) and Java (`adservice`).
  - **Containers to question:**
    - `loadgenerator` is a test tool, not part of the shop. It's commented out of `kustomization.yaml` (L24), so drawing it outside the boundary is right.
    - `redis-cart` keeps its data on an `emptyDir` volume, so carts are lost when the pod restarts.
    - Optional data stores that the code supports but the default manifests don't use are missing: `cartservice` has `AlloyDBCartStore.cs` and `SpannerCartStore.cs`, and `productcatalogservice/catalog_loader.go` L37 can load from AlloyDB.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?

   From `demo.proto` alone the agent could list the 9 services and their operations, but it found **zero** dependencies. It also couldn't see the `frontend` (an HTTP server with no proto service), the `redis-cart` data store, the external entry point or any ports. This is because a `.proto` file only describes what each service **offers** (its interface), not what it **calls**. The wiring lives in deployment config (the `*_ADDR` env vars in the manifests) and in client code, which only Run B could read.

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?

   The missing `frontend` → `shoppingassistantservice` call. The diagram says the frontend only talks gRPC to 7 internal services, but the code also makes a plain-HTTP call to an AI assistant service that depends on Google Gemini and AlloyDB. A new team member would not know the system can forward shoppers' requests to an external cloud AI service, which matters for cost, privacy and the company's "self-hosted" preference.
