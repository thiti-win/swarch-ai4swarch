Agent: Claude Code (Anthropic), running as a sub-agent session. Model: Claude Opus 5.5 (`claude-opus-5-5`).

# ADR-NNNN: Use synchronous unary gRPC with one shared `demo.proto` for service-to-service communication

- **Status:** Reconstructed
- **Date:** 2026-10-26 (date of reconstruction; the original decision date is Not evidenced in the provided files)
- **Deciders:** Not evidenced in the provided files (they only carry `Copyright 2018/2020 Google LLC` headers). Reconstructed by Claude Code for the student.

> **Evidence scope.** Only these files were read: `system/online-boutique/protos/demo.proto`, `system/online-boutique/src/checkoutservice/main.go`, `system/online-boutique/src/frontend/main.go`. The code of the other services, generated code, build scripts and deployment manifests were **not** read. Claim IDs (C1, C2, …) map to the **Claim check** table at the end.

## Context and problem

**What the evidence shows**

| ID | Fact | Evidence |
|---|---|---|
| C1 | One proto file, `demo.proto` (package `hipstershop`), defines 9 gRPC services: Cart, Recommendation, ProductCatalog, Shipping, Currency, Payment, Email, Checkout, Ad. | `demo.proto` L17, L23, L56, L71, L107, L140, L177, L199, L224, L243 |
| C2 | Every RPC is unary request/response. The file contains no `stream` keyword. | `demo.proto`, all `rpc … returns (…)` lines |
| C3 | Message types are shared across service boundaries. `Money` appears in the messages of 6 services (catalog, shipping, currency, payment, email, checkout). `Address` and `CartItem` are also shared. | `Money`: L82, L118, L142, L169, L189, L205, L211. `Address`: L113, L122, L212, L232. `CartItem`: L36, L49, L114, L123, L204 |
| C4 | Checkout is a gRPC server (port 5050) and a gRPC client of 6 services: shipping, product catalog, cart, currency, email, payment. | `checkoutservice/main.go` L46, struct `checkoutService` L66-86, `mustConnGRPC` calls L118-123, `grpc.NewServer` L138, `RegisterCheckoutServiceServer` L142 |
| C5 | `PlaceOrder` runs the order as a chain of blocking calls, one after another: GetCart, then GetProduct + Convert for each item, then GetQuote, Convert, Charge, ShipOrder, EmptyCart, SendOrderConfirmation. That is **2N + 7 RPCs for N cart items**, with no parallelism. | `checkoutservice/main.go` `PlaceOrder` L230-280, `prepareOrderItemsAndShippingQuoteFromCart` L288-311, loop in `prepOrderItems` L343-355 |
| C6 | The checkout response needs results from downstream calls. `OrderResult` carries `ShippingTrackingId` (from ShipOrder) and `ShippingCost` (from GetQuote + Convert), and Charge must succeed before ShipOrder is called. | `checkoutservice/main.go` L252-271 |
| C7 | Some results are not needed. A failed email confirmation is only logged, and the EmptyCart error is thrown away. | `checkoutservice/main.go` L263 (`_ = cs.emptyUserCart`), L273-277 |
| C8 | Frontend is an **HTTP** server for the browser (gorilla/mux, port 8080). It is a gRPC client of 7 services: currency, product catalog, cart, recommendation, shipping, checkout, ad. When tracing is on, it also uses gRPC to reach an OTLP collector. | `frontend/main.go` L20, L25, L39, L141-147, L149-171, L178-182 |
| C9 | Frontend also reads `SHOPPING_ASSISTANT_SERVICE_ADDR`, but opens **no** gRPC connection to it, and `demo.proto` defines no shopping-assistant service. | `frontend/main.go` L88, L139 (no matching `mustConnGRPC`), routes L158, L163; `demo.proto` has no such service |
| C10 | Both Go services add OpenTelemetry handlers to every gRPC client and server, and always set W3C trace-context propagation. Traces are only exported when `ENABLE_TRACING=1`. Checkout also registers the standard gRPC health service. | `checkoutservice/main.go` L90, L134-140, L143-144, L216; `frontend/main.go` L107-109, L113, L231 |
| C11 | All gRPC connections use plaintext transport. | `insecure.NewCredentials()`: `checkoutservice/main.go` L215, `frontend/main.go` L230 |
| C12 | Service addresses come from environment variables. If one is missing, the service panics at startup. | `mustMapEnv`: checkout L111-116 and L202-208, frontend L132-139 and L217-223 |
| C13 | Checkout imports generated code from its **own** package `…/src/checkoutservice/genproto`, but `demo.proto` declares `go_package = "…/microservices-demo/hipstershop"`. | `checkoutservice/main.go` L34; `demo.proto` L19 |
| C14 | Neither Go file imports a message broker, queue or event library. | import blocks: checkout L17-43, frontend L17-36 |

**What the evidence does *not* show**

- **"All services" is not supported by the evidence (C15).** Only frontend and checkout code was read. Whether the other 8 services implement their `demo.proto` services as gRPC servers is Not evidenced in the provided files. `frontend/main.go` also never imports generated `demo.proto` code, so its use of those types is not shown in that file. C9 points to a service that `demo.proto` does not cover.
- **No rationale is written down (C16).** The three files contain no comment or document that says why gRPC or a single proto was chosen. Everything under *Decision drivers* and *Decision* is therefore **[Inferred]**.
- **Languages of the other services (C17):** Not evidenced. Only Go is shown (both `.go` files, and `demo.proto` declares only `go_package`).

**Problem (reconstructed):** how should the shop's services call each other, and where should their API contracts live?

## Decision drivers

None of these are stated in the evidence. All are reconstructed, most important first.

1. **Request/response semantics for checkout (C18):** the order response needs the tracking ID and costs from downstream services (C6). [Inferred driver, based on an Evidenced fact]
2. **Typed, explicit contracts between separately deployed services** (modifiability, interoperability) (C19). [Inferred]
3. **Simplicity of a demonstration system (C20):** one file shows the whole API. The names `demo.proto` and `microservices-demo` (`demo.proto` L19) suggest the system is a demo. [Inferred]
4. **Observability of call chains across services (C21):** see C10. [Inferred driver]
5. **Fast internal calls (C22):** binary encoding over HTTP/2. [Assumption]
6. **Language neutrality (C23):** a `.proto` IDL can generate code for many languages. [Inferred. Use of more than one language is Not evidenced.]

## Considered options

1. **A. Synchronous unary gRPC, all contracts in one shared `demo.proto`** (what the code shows)
2. **B. Synchronous gRPC, one `.proto` per service**, owned by that service
3. **C. Synchronous REST/JSON over HTTP**, e.g. an OpenAPI spec per service
4. **D. Hybrid:** gRPC for queries plus asynchronous messaging (a broker or events) for side effects such as email confirmation and cart emptying

The evidence does not show that B, C or D were ever considered. They are reconstructed alternatives (C24).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A | Typed contract and generated clients (`pb.New…Client` in checkout). Shared `Money`/`Address`/`CartItem` keep data formats consistent (C3). One file shows the whole API. Uniform tracing per hop (C10). (C25) | Temporal coupling: checkout fails when 5 of its 6 dependencies fail (C40). Latency adds up over 2N + 7 sequential calls (C5). One file is a shared change point for every service. Generated code is copied per service and can drift from the proto (C13). (C26) |
| B | Same typed gRPC benefits. Each service owns and versions its own contract, so a change to Ad cannot touch Payment's file. (C27) | Shared types (`Money`, `Address`) must be copied or moved into a common proto, which brings the coupling back. More files and import paths to manage. (C27) |
| C | Universal tooling, readable payloads, easy to test with a browser or curl, no code generation needed. (C28) | Weaker contracts unless OpenAPI is enforced. Larger, slower-to-parse payloads. Still synchronous, so the same temporal coupling as A. (C28) |
| D | Steps whose result isn't needed (C7) stop blocking the user. An email outage cannot slow checkout. Load is levelled. (C29) | Needs a broker: more infrastructure, cost and operations. Eventual consistency, and harder debugging. The response still needs ShipOrder's tracking ID (C6), so the core flow stays synchronous anyway. (C30) |

## Decision

[Reconstructed] **Option A.** All service-to-service calls shown are synchronous unary gRPC, and every contract lives in one shared `demo.proto` (C31).

Why it probably fits the drivers:

- Checkout must return downstream results in the same request (C6), so request/response suits most steps (C32). [Inferred]
- One proto with shared `Money`/`Address`/`CartItem` types keeps the system's whole API in one readable place, which suits a demo (C33). [Inferred]
- gRPC provides generated, typed clients, and OpenTelemetry plugs in with one stats handler per connection (C10, C34). [Inferred]
- Asynchronous messaging was not used even for steps whose result is ignored (C7, C14). A likely reason is to keep the demo free of extra infrastructure (C35). [Inferred]

## Consequences

- **Positive:** One typed, generated contract covers all service APIs, and shared types prevent mismatched money or address formats (C36). (quality attribute: **modifiability / interoperability**)
- **Positive:** The flow is easy to follow: `PlaceOrder` reads top to bottom as a sequence of calls (C37). (quality attribute: **understandability / maintainability**)
- **Positive:** Each gRPC hop can be traced through the `otelgrpc` handlers, with context propagation. Traces are exported only when `ENABLE_TRACING=1` (C38). (quality attribute: **observability**)
- **Positive:** The user gets a complete order result, with tracking ID and shipping cost, in one response (C39). (quality attribute: **usability / functional suitability**)
- **Negative:** If GetCart, GetProduct, Convert, GetQuote, Charge or ShipOrder fails, the order fails. So checkout depends on 5 of its 6 dependencies; only a failure in email (or EmptyCart) is tolerated (C40). (quality attribute: **availability**)
- **Negative:** Checkout latency grows with cart size: 2N + 7 sequential RPCs, none run in parallel (C41). (quality attribute: **performance**)
- **Negative:** The shared proto is a single change point. A change to `Money` affects the contracts of 6 services (C42). (quality attribute: **modifiability**, team autonomy)
- **Negative:** Nothing undoes earlier steps when a later one fails. If ShipOrder fails after Charge succeeded, checkout returns `codes.Unavailable` but the card stays charged, and `PaymentService` has only `Charge` with no refund RPC (C43). (quality attribute: **reliability / data consistency**). Evidence: `checkoutservice/main.go` L252-261; `demo.proto` L177-179
- **Negative:** gRPC traffic between services is plaintext (C11, C44). (quality attribute: **security**)
- **Risks and mitigations:**
  - **R1 – Deadlines are not propagated (C45).** The code sets no per-call deadline or retry. The 3 s timeout context in `mustConnGRPC` is thrown away (`_, cancel := …`, checkout L212, frontend L227). `convertCurrency` passes `context.TODO()` instead of `ctx` (checkout L360), which drops the caller's deadline and cancellation, and probably its trace parent too [Inferred]. *Mitigation:* pass `ctx`, set a deadline on each RPC, add a retry/back-off policy.
  - **R2 – Generated code may drift from `demo.proto` (C46).** [Inferred from C13] *Mitigation:* regenerate in CI and fail the build on any diff.
  - **R3 – A slow or failing service can cascade through the synchronous chain (C47).** [Assumption] *Mitigation:* timeouts and circuit breakers, or move only email and cart-emptying to async (Option D for side effects).
  - **R4 – Traffic is plaintext (C48).** *Mitigation:* TLS or mTLS, for example through a service mesh [Assumption]. Whether a mesh is used is Not evidenced in the provided files.

## Validation

- Trace `PlaceOrder` end to end (tracing is already wired, C10) and plot p95 latency against cart size N. If C41 holds, latency should rise linearly.
- Fault injection: stop the email service and orders should still succeed (C7). Stop currency or shipping and orders should fail (C40).
- Contract check: regenerate Go code from `demo.proto` and diff it against each service's `genproto` folder (R2).
- To confirm or refute "all services use gRPC and `demo.proto`" (C15) and the shopping-assistant exception (C9), read the other services' entry points and the deployment manifests. This is outside the evidence allowed for this task.
- If the original maintainers can be reached, review the Inferred rationale with them (C16).

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| C1 | One `demo.proto` defines 9 gRPC services | Evidenced | `demo.proto` `service` declarations L23–L243 |
| C2 | All RPCs are unary, with no streaming | Evidenced | `demo.proto`: no `stream` keyword in any `rpc` |
| C3 | `Money`, `Address`, `CartItem` are shared across services | Evidenced | `demo.proto` `message Money` L146 and its uses (see C3 row above) |
| C4 | Checkout is a gRPC server and a client of 6 services | Evidenced | `checkoutservice/main.go` `checkoutService`, `mustConnGRPC` L118-123, `RegisterCheckoutServiceServer` L142 |
| C5 | `PlaceOrder` makes 2N + 7 sequential blocking RPCs | Evidenced | `checkoutservice/main.go` `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `prepOrderItems` loop L343-355 |
| C6 | The order response needs downstream results (tracking ID, cost) | Evidenced | `checkoutservice/main.go` `OrderResult` built at L265-271 |
| C7 | Email and EmptyCart failures are tolerated | Evidenced | `checkoutservice/main.go` L263, L273-277 |
| C8 | Frontend serves HTTP and is a gRPC client of 7 services (plus the OTLP collector) | Evidenced | `frontend/main.go` `mux.NewRouter` L149, `ListenAndServe` L171, `mustConnGRPC` L141-147, `initTracing` L178-182 |
| C9 | The shopping-assistant service is outside `demo.proto` and has no gRPC connection in `frontend/main.go` | Evidenced | `frontend/main.go` `shoppingAssistantSvcAddr` L88, L139; `demo.proto` (absent). How frontend reaches it: Not evidenced |
| C10 | OTel handlers on all gRPC hops, trace export only when `ENABLE_TRACING=1`, gRPC health service on checkout | Evidenced | `otelgrpc.NewServerHandler` checkout L139; `otelgrpc.NewClientHandler` checkout L216, frontend L231; `healthpb.RegisterHealthServer` L144 |
| C11 | Plaintext gRPC transport | Evidenced | `insecure.NewCredentials()` checkout L215, frontend L230 |
| C12 | Addresses come from env vars, and a missing one causes a panic | Evidenced | `mustMapEnv` in both files |
| C13 | Checkout uses its own `genproto` package, whose path differs from the proto's `go_package` | Evidenced | `checkoutservice/main.go` L34; `demo.proto` L19 |
| C14 | No broker or queue library in either file | Evidenced | Import blocks of both files (absence) |
| C15 | "All services communicate over gRPC using `demo.proto`" | Inferred | True only for frontend → 7 services and checkout → 6 services. The other servers are Not evidenced, and C9 is a counter-example |
| C16 | No written rationale exists in the evidence | Evidenced | Absence in all three files |
| C17 | Only Go is evidenced. Other languages are unknown | Evidenced | Two `.go` files; `demo.proto` has only `option go_package` |
| C18 | Driver: checkout needs request/response semantics | Inferred | Based on C6. No stated rationale |
| C19 | Driver: typed contracts between services | Inferred | Typed contracts exist (C1), but no stated reason |
| C20 | Driver: simplicity of a demo | Inferred | Naming (`demo.proto`, `microservices-demo` L19) only |
| C21 | Driver: observability | Inferred | Tracing is wired (C10), but no stated reason |
| C22 | Driver: gRPC is fast (binary, HTTP/2) | Assumption | General industry knowledge |
| C23 | Driver: language neutrality | Inferred | Proto IDL is language-neutral (general knowledge). Multi-language use is Not evidenced |
| C24 | B, C and D are realistic alternatives, though not evidenced as considered | Inferred | No evidence of alternatives in the files |
| C25 | Pros of A: typed generated clients, shared types, uniform tracing | Evidenced | `pb.NewShippingServiceClient` etc. in checkout; C3; C10 |
| C26 | Cons of A: temporal coupling, additive latency, shared change point, copy drift | Inferred | Structure is Evidenced (C5, C13, C40). The size of the impact is not measured |
| C27 | Pros and cons of B (per-service proto) | Assumption | General industry knowledge |
| C28 | Pros and cons of C (REST/JSON) | Assumption | General industry knowledge |
| C29 | Pros of D (async side effects) | Assumption | General industry knowledge |
| C30 | Under D the core flow would still be synchronous | Inferred | Follows from C6 (`ShippingTrackingId` in the response) |
| C31 | Option A is what the system implements (for the files read) | Evidenced | C1, C2, C4, C8 |
| C32 | Rationale: request/response suits checkout | Inferred | No stated rationale. Consistent with C6 |
| C33 | Rationale: one readable proto suits a demo | Inferred | Naming hint only (C20) |
| C34 | Rationale: gRPC makes typed clients and tracing easy | Inferred | Consistent with C10 and C25. Not stated |
| C35 | Rationale: async was avoided to keep infrastructure minimal | Inferred | Only the absence of async is evidenced (C14) |
| C36 | Positive: modifiability/interoperability through one typed contract | Inferred | Typed shared contract is Evidenced (C1, C3). The benefit is not measured |
| C37 | Positive: `PlaceOrder` is easy to follow | Inferred | Readability is a judgement on `checkoutservice/main.go` L230-280 |
| C38 | Positive: every gRPC hop can be traced | Evidenced | C10 (handlers on client and server, propagator always set) |
| C39 | Positive: complete order result in one response | Evidenced | `PlaceOrderResponse{Order: orderResult}` checkout L278 |
| C40 | Negative: order fails if any of 5 of the 6 dependencies fails | Evidenced | Error returns in `PlaceOrder` L238-261 and `prepareOrderItemsAndShippingQuoteFromCart` L290-305. Only email and EmptyCart failures are ignored |
| C41 | Negative: latency grows with cart size (2N + 7 sequential calls) | Evidenced | `prepOrderItems` loop L343-355. Actual latency figures: Not evidenced |
| C42 | Negative: a `Money` change affects 6 services' contracts | Evidenced | `demo.proto` uses of `Money` (C3) |
| C43 | Negative: no compensation or refund when ShipOrder fails after Charge | Evidenced | `checkoutservice/main.go` L252-261; `PaymentService` has only `Charge` (`demo.proto` L177-179) |
| C44 | Negative: plaintext traffic is a security weakness | Evidenced | C11. Whether a mesh adds mTLS: Not evidenced |
| C45 | Risk R1: no deadlines or retries, and `context.TODO()` drops the caller's context | Evidenced | `checkoutservice/main.go` L212, L360; `frontend/main.go` L227. Loss of the trace parent is Inferred |
| C46 | Risk R2: `genproto` copies may drift from `demo.proto` | Inferred | Based on C13. The `genproto` folder was not read |
| C47 | Risk R3: cascading failure in synchronous chains | Assumption | General industry knowledge |
| C48 | Risk R4: mitigate plaintext with TLS/mTLS or a mesh | Assumption | General industry knowledge |
