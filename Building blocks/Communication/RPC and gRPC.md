# RPC and gRPC

## Why this topic is a filter question

Anyone can say "gRPC uses Protobuf and is faster than JSON." The filter is whether you understand *why* internal service meshes standardize on RPC frameworks: deadline propagation across a call chain, what retrying actually costs an already-struggling downstream, and how you evolve a strongly-typed contract without a synchronized deploy across a dozen teams. Those are the questions that come up once you've operated a real microservice fleet, and they're where this doc goes deep.

---

## 1. RPC vs. REST: a modeling choice, not just a technology choice

REST asks you to model a domain as **resources** manipulated by a uniform verb set. RPC asks you to model it as **named operations** — `ReserveInventory(request)` rather than `POST /reservations`. Neither is strictly better; they fit different call shapes:

- An operation that's naturally a resource lifecycle (create/read/update/delete an order) fits REST cleanly.
- An operation that's naturally a verb with no obvious resource identity — `RecomputeShippingEstimate`, `MergeAccounts`, `ValidateCoupon` — gets contorted into REST (`POST /shipping-estimates:recompute`) but is a direct method call in RPC.

gRPC is the specific RPC framework worth knowing deeply: Protobuf schemas + code generation + HTTP/2 transport + native streaming. See [REST](REST.md) for the resource-modeling side of this comparison.

---

## 2. Deadlines, retries, and cascading failure — the section that actually gets probed

This is the highest-signal area of this topic in a staff interview, because it's where naive designs quietly create outages.

**Deadlines must propagate, not reset.** If service A calls B with a 2s deadline, and B calls C, C should inherit A's *remaining* budget (say, 1.3s after A→B latency), not get a fresh 2s of its own. gRPC propagates deadlines through call context for exactly this reason. Without propagation, a caller that's already given up can leave the entire downstream chain still working on a request nobody's waiting for anymore — burning capacity for zero benefit, and in the worst case queueing behind it while the real, live requests wait.

**Retries amplify load exactly when the downstream can least afford it.** If B is slow because it's overloaded, and every caller of B retries on timeout, you've just multiplied B's incoming load at the worst possible moment — a classic retry storm that turns a slow dependency into a fully down one. The mitigations, in order of how often they show up:
- **Only retry idempotent/safe operations** — retrying `ReserveInventory` blindly can double-reserve stock; retrying `GetInventory` is free.
- **Exponential backoff with jitter** — prevents synchronized retry waves across many callers.
- **Retry budgets** — cap total retry volume as a fraction of original traffic (e.g., retries may add at most 10% extra load), so retries can't multiply load unboundedly under sustained failure.
- **Circuit breakers** — after a failure-rate threshold, stop calling the dependency entirely for a cooldown window instead of continuing to retry into it, giving it room to recover.

A strong answer connects all three (deadlines, retries, circuit breakers) as one system: deadlines bound how long any single attempt can burn resources, retry budgets bound how much amplification a failure can cause, and circuit breakers stop the bleeding entirely once a dependency is clearly unhealthy.

---

## 3. Streaming — and when it's actually the right call

gRPC's four call shapes:

| Shape | Pattern | Real use |
|---|---|---|
| Unary | 1 request → 1 response | `GetUser`, `ReserveInventory` |
| Server streaming | 1 request → N responses | Subscribing to low-stock alerts, streaming query results too large for one response |
| Client streaming | N requests → 1 response | Uploading telemetry batches, then getting one acknowledgment |
| Bidirectional streaming | Both sides stream independently | Live coordination, chat-like sync between services |

The judgment call interviewers probe: **streaming isn't free**. A long-lived stream holds a connection and often server-side state for its duration, which complicates load balancing (a stream is pinned to whichever server instance accepted it — see the L7/connection-pinning point below) and makes deploys harder (rolling a new version means either draining active streams gracefully or cutting them). Reach for streaming when the *shape* of the data genuinely is a sequence over time (log tailing, live alerts) — not merely because "it's an available RPC type." A batch of 500 known records is usually better served by a single response with pagination than by client streaming.

---

## 4. HTTP/2 mechanics that actually matter operationally

- **Multiplexing**: many concurrent RPCs share one TCP connection, avoiding the connection-per-request overhead of HTTP/1.1. This is a real throughput win for chatty internal call graphs.
- **The load-balancing catch**: multiplexing means a single TCP connection now carries many logical requests, which breaks traditional L4 (connection-level) load balancing — if all your traffic to a backend rides one persistent connection, standard connection-based LB sends 100% of it to one instance. Real gRPC deployments need either **client-side load balancing** (client resolves a list of backend addresses and spreads RPCs across multiple connections itself, e.g., via `grpc-lb` or a service mesh sidecar) or an **L7-aware proxy** (Envoy, and similar) that load-balances at the individual-request level, not the connection level. This is a genuinely easy trap to fall into and a good thing to volunteer if you're describing a gRPC deployment at scale.

---

## 5. Schema evolution — where strong typing cuts both ways

Protobuf's strong typing catches integration bugs at compile time, but it also means a careless schema change can break every generated client simultaneously, which is a much bigger blast radius than a loosely-typed JSON API silently ignoring an unknown field.

Rules that keep evolution safe:
- **Never reuse a field number.** Field numbers, not names, are what's on the wire. Reusing number `3` for a different field after removing the old one means old binaries still running will misinterpret new data as the old type.
- **Mark removed fields `reserved`** so nobody accidentally reuses their number or name.
- **Add fields as new, optional numbers; never repurpose or change the type of an existing field.**
- **Treat the `.proto` file as a versioned artifact with its own review process** — in a polyglot org, a breaking schema change effectively requires a coordinated multi-service, multi-language rollout unless you follow additive-only evolution.

This is the RPC analog of REST's expand-contract pattern (§5 in [REST](REST.md)) — the mechanism differs, the underlying discipline (never break what's already deployed) is identical, and drawing that parallel explicitly is a good signal in an interview.

---

## 6. Error model and identity propagation

gRPC status codes (`NOT_FOUND`, `UNAVAILABLE`, `DEADLINE_EXCEEDED`, `PERMISSION_DENIED`, ...) are a fixed enum, not free-text like HTTP status codes with arbitrary bodies — this is deliberate, so callers across every language binding can branch on failure type without parsing strings. `UNAVAILABLE` specifically is worth knowing precisely: it means "the call didn't get a real answer" (could be a transient network blip, could be the server rejecting load) and is the canonical *safe-to-retry* signal, whereas something like `INVALID_ARGUMENT` retrying is pointless — the request will fail identically every time.

Identity propagation is a real design point in service meshes: if a gateway authenticates an end user and then fans out to five internal gRPC services, each of those services often needs to know *both* which service is calling it (service identity, usually via mTLS) *and* which end user the request is ultimately for (propagated via metadata, e.g. a signed user-context token) — for authorization checks like "can this user see this data," not just "is this caller a trusted internal service."

---

## 7. When gRPC is the wrong choice

- **Public/partner APIs** — most third-party developers expect JSON/REST tooling; requiring `.proto` consumption and codegen raises integration friction significantly.
- **Direct browser clients** — browsers can't speak native gRPC (no fine-grained HTTP/2 frame control from JS); you'd need gRPC-Web plus a proxy layer, which is extra infrastructure for no benefit if REST/JSON would do.
- **Low request volume, simple CRUD, or a small team without mesh/proto tooling maturity** — the operational cost (proto pipeline, codegen in CI, schema review discipline) isn't worth it below a certain organizational scale.

---

## 8. Worked example: inventory service contract

A realistic internal service, illustrating unary reads/writes plus server streaming:

```proto
syntax = "proto3";
package inventory.v1;

service InventoryService {
  rpc GetInventory(GetInventoryRequest) returns (GetInventoryResponse);
  rpc ReserveInventory(ReserveInventoryRequest) returns (ReserveInventoryResponse);
  rpc ReleaseInventory(ReleaseInventoryRequest) returns (ReleaseInventoryResponse);
  rpc StreamLowStockAlerts(StreamLowStockAlertsRequest) returns (stream LowStockAlert);
}

message GetInventoryRequest {
  string sku = 1;
  string warehouse_id = 2;
}

message GetInventoryResponse {
  string sku = 1;
  string warehouse_id = 2;
  int32 available_quantity = 3;
  int32 reserved_quantity = 4;
}

message ReserveInventoryRequest {
  string request_id = 1;   // idempotency key — see §2 and REST.md §2
  string order_id = 2;
  string sku = 3;
  string warehouse_id = 4;
  int32 quantity = 5;
}

message ReserveInventoryResponse {
  string reservation_id = 1;
  string status = 2;       // RESERVED | INSUFFICIENT_STOCK
  int32 reserved_quantity = 3;
}

message StreamLowStockAlertsRequest {
  string warehouse_id = 1;
}

message LowStockAlert {
  string sku = 1;
  int32 available_quantity = 2;
  int32 reorder_threshold = 3;
  string triggered_at = 4;
}
```

Design notes worth being able to explain out loud:
- `request_id` on `ReserveInventory` exists because this is a write with real-money consequences (over-reserving stock) — same idempotency-key pattern as REST `POST`, just carried as a field instead of a header since gRPC doesn't have HTTP-style headers as the primary contract surface.
- `ReserveInventory` should carry a caller-set deadline (e.g., 300ms) — if the order service can't get an answer that fast, it should fail the checkout fast rather than hang, and should **not** blindly retry without checking `request_id` dedup on the server side first.
- `StreamLowStockAlerts` is server-streaming because the data genuinely is a live sequence over time — a good example of streaming used for the right reason ([§3](#3-streaming-and-when-its-actually-the-right-call "Streaming — and when it's actually the right call")), not just because it's available.

---

## 9. What separates senior from staff here

- **Senior**: knows what gRPC is, knows Protobuf gives strong typing and codegen, can name unary vs. streaming call shapes.
- **Staff**: reasons about deadline propagation and retry amplification as a system, not isolated features; knows why HTTP/2 multiplexing breaks naive connection-based load balancing and what fixes it; treats schema evolution as an org-wide coordination discipline; picks streaming only when the data shape justifies the operational cost of a pinned long-lived connection; distinguishes retryable failure classes (`UNAVAILABLE`) from non-retryable ones (`INVALID_ARGUMENT`) explicitly.

---

## 10. Questions to expect, and what a strong answer sounds like

- **"Service A calls B calls C — A gives up after 2s, but C is still doing work 5s later. Why, and how do you fix it?"** → Deadlines weren't propagated down the call chain; fix by passing remaining budget at each hop so downstream work is cancelled once the original caller has given up.
- **"A downstream dependency degrades and your error rate spikes even higher than the dependency's actual failure rate — why?"** → Retry amplification; walk through retry budgets, backoff with jitter, and circuit breakers as the layered fix.
- **"You load-balance gRPC traffic with a plain L4 load balancer and one backend instance gets 90% of traffic — why?"** → HTTP/2 connection multiplexing pins many RPCs onto one persistent connection; L4 balances connections, not requests. Fix with client-side LB or an L7-aware proxy.
- **"You need to add a field to a widely-used request message — what's the actual process?"** → Add as a new, additive, optional field number; never reuse or repurpose an existing one; treat it like the expand phase of REST's expand-contract, coordinated across consuming services before anything depends on it being present.
- **"Would you use gRPC for this public developer API?"** → No by default — name the browser-compatibility and third-party integration friction costs, and prefer REST/JSON at that boundary even if internal services behind it use gRPC.
