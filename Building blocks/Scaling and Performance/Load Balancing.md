# Load Balancing

## Why this topic is a filter question

Load balancing sounds like a solved, boring problem — "round robin across servers." The filter is whether you understand what happens at the edges: how health checks can lie, why the "obviously best" routing strategy (least-connections) can misbehave with uneven request costs, why long-lived connections invalidate the request-rate mental model entirely, and how the load balancer itself becomes a single point of failure if you don't think about it as a system with its own availability requirements.

---

## 1. What a load balancer actually decides, and where it sits

A load balancer's job has two parts people conflate: **picking which backend gets a given unit of traffic**, and **detecting which backends are eligible to receive traffic at all** (health checking). Getting either one wrong breaks availability even if the other is perfect — a great routing algorithm distributing traffic evenly across five backends, one of which is actually broken, still sends 20% of traffic into a black hole.

Load balancers show up at multiple layers of a real system, each with a different job:
- **Edge/global**: routes users to a region or point of presence (DNS-based or Anycast) — see [§5](#5-global-load-balancing-and-failover "Global load balancing and failover").
- **L4/L7 in front of a service tier**: the classic case — spreading requests across stateless application servers.
- **Internal/east-west (service mesh)**: load balancing between internal services, often client-side (a sidecar or library picks a backend directly from a discovered list, rather than routing through a centralized box) — this is the same problem [gRPC's HTTP/2 multiplexing §4](../Communication/RPC%20and%20gRPC.md#4-http2-mechanics-that-actually-matter-operationally "HTTP/2 mechanics that actually matter operationally") forces you to solve explicitly, since a centralized L4 balancer doesn't work cleanly against multiplexed connections.

---

## 2. L4 vs. L7 — what each can and can't see

- **L4 (transport layer)**: routes based on IP/port, without parsing the request content. Fast, low overhead, but blind to anything inside the request — it cannot route `/api/*` differently from `/admin/*`, and (per the gRPC note above) it balances *connections*, not individual requests riding a multiplexed connection.
- **L7 (application layer)**: terminates and reads the request (HTTP method, path, headers, even body in some cases) before deciding. Enables path-based routing, header-based routing (canary releases via a header), and per-request load balancing even over multiplexed HTTP/2 connections — at the cost of more CPU per request (TLS termination, parsing) and being a more complex piece of infrastructure to operate.

The decision isn't "L7 is strictly better" — L4 is the right (and often only sensible) choice when you don't need content-aware routing and want the lowest possible overhead (e.g., a raw TCP service, or the outermost tier in front of an L7 layer that does the real routing).

---

## 3. Routing strategies — and where the "obvious" choice breaks

- **Round robin**: simplest, works well when requests are roughly uniform cost and backends are roughly uniform capacity. Breaks down when either assumption is false — a backend that's twice as slow still gets an equal share of new requests, building up a growing backlog.
- **Least connections**: routes to whichever backend currently has the fewest active connections/requests — self-correcting for backends that are temporarily slow, since a backend accumulating in-flight requests gets fewer new ones. The trap: this assumes connection count correlates with load, which breaks for **long-lived, low-activity connections** — a backend holding 10,000 idle WebSocket connections and one doing real work looks "busiest" by this metric while actually having spare capacity, so least-connections routing needs a load signal that's actually meaningful for the workload (active request count, not open connection count, for a WebSocket gateway — see [§6](#6-long-lived-connections-break-the-request-rate-mental-model "Long-lived connections break the request-rate mental model")).
- **Weighted routing**: assign traffic proportional to declared backend capacity — necessary during a mixed-instance-size rollout, or to intentionally send a small percentage of traffic to a canary version.
- **Hash-based (consistent hashing on a key)**: routes the same key (client ID, session ID) to the same backend consistently. This is what makes sticky sessions and cache-affinity routing work — and it's the same consistent-hashing mechanism used for [partition rebalancing §4](../Data%20and%20Consistency/Replication%20and%20Partitioning.md#4-resharding-the-operation-everyone-underestimates "Resharding — the operation everyone underestimates"), for the same reason: it minimizes redistribution when the backend set changes (a node added/removed only reshuffles a fraction of keys, not all of them).

A strong answer picks a strategy by naming the actual cost distribution of the workload — "requests are uniform cost and stateless, round robin is fine" vs. "this backend holds long-lived connections, so I need connection-aware or even custom load-signal-based routing," rather than defaulting to whichever strategy is most commonly cited.

---

## 4. Health checks — the part that quietly determines your real availability

A load balancer is only as good as its ability to detect a bad backend, and there's a real hierarchy of how much a check actually proves:

- **TCP check** (can we open a connection?): proves the process is up and listening. Doesn't prove it can actually serve a request — a backend deadlocked or stuck behind an exhausted database connection pool can still accept TCP connections while being functionally dead.
- **HTTP check** (`GET /health` returns 200): proves the HTTP server loop is alive. Still doesn't prove the backend can serve *real* traffic if `/health` doesn't exercise the actual dependency path.
- **Deep/readiness check**: the health endpoint itself verifies critical dependencies (can it reach its database, its cache, its downstream) before returning healthy. This is the check that actually correlates with "can serve real traffic," but it comes with its own trap: if every instance's readiness check depends on the same downstream, and that downstream degrades, every instance can flip unhealthy simultaneously — turning one dependency's partial degradation into a total outage via correlated health-check failure. This tradeoff (shallow checks under-detect; deep checks can over-correlate failures) is worth naming explicitly, since neither extreme is free.

Also worth distinguishing: **liveness** ("is the process alive, should we restart it") vs. **readiness** ("should the load balancer currently send it traffic") — a backend can be alive but not ready (still warming a cache on startup, or intentionally draining before shutdown), and conflating the two checks causes either premature traffic (not-yet-warm instance gets full traffic right after passing a liveness-only check) or unnecessary restarts (a temporarily-not-ready but perfectly healthy process gets killed).

---

## 5. Global load balancing and failover

Beyond a single cluster, load balancing has to route users to a *region*, which introduces a different set of tradeoffs:

- **DNS-based (GeoDNS)**: resolves a hostname to a region-specific IP based on the resolver's location. Simple, widely compatible, but DNS caching (client-side and at resolvers) means failover isn't instant — a region marked unhealthy can still receive traffic from clients holding a cached DNS answer until the TTL expires, which is a real, bounded-but-nonzero failover delay worth stating rather than assuming failover is immediate.
- **Anycast**: the same IP address is announced from multiple locations, and network routing (BGP) sends each client to the topologically nearest one. Failover can be near-instant (routing withdraws the announcement from a failed location), at the cost of needing the underlying network infrastructure to support it (this is largely a CDN/edge-provider-level capability, not something most teams build themselves).
- **Active-active vs. active-passive**: active-active serves real traffic from multiple regions simultaneously (better resource utilization, but requires the data layer to support multi-region writes or read-only secondary regions — see [Replication and Partitioning §1](../Data%20and%20Consistency/Replication%20and%20Partitioning.md#1-replication-why-and-by-what-mechanism "Replication: why, and by what mechanism")'s multi-leader tradeoffs); active-passive keeps a region on standby purely for failover (simpler data story, wastes standby capacity, and failover itself needs to be tested regularly or it silently rots).

---

## 6. Long-lived connections break the request-rate mental model

This deserves explicit emphasis because it's the single most common blind spot in load balancing discussions, and it connects directly to [WebSockets §5](../Communication/Websockets.md#5-scaling-connection-pinning-is-the-real-complexity-not-throughput "Scaling: connection pinning is the real complexity, not throughput"), [SSE §4](../Communication/Server-sent%20events%20%28SSE%29.md#4-scaling-connection-count-is-the-constraint-and-the-fan-out-problem-is-universal "Scaling: connection count is the constraint, and the fan-out problem is universal"), and [Long Polling §2](../Communication/Long%20Polling.md#2-the-real-cost-held-request-capacity-not-bandwidth "The real cost: held-request capacity, not bandwidth"):

- For stateless HTTP, "load" is well-approximated by requests/sec, and any backend can serve any request — trivially horizontally scalable behind a load balancer with no special care.
- For WebSocket/SSE gateways, a connection, once established, is **pinned to whatever backend instance accepted it** for its entire lifetime — the load balancer's routing decision at connect time has a much longer-lasting effect than a per-request decision, and "load" per backend is better measured by open-connection count and the resources those connections hold (memory, file descriptors) than by request throughput. This is also why a rolling deploy of a WebSocket/SSE fleet needs explicit connection draining (stop accepting new connections on an instance, wait for or force-close existing ones) rather than the simple "just stop sending it new requests" story that works for stateless HTTP.

---

## 7. Sticky sessions — usually a symptom, not a feature

Sticky sessions (routing a given client consistently to the same backend, typically via a cookie) exist to work around backend-local state (in-memory session data, local cache). The staff-level framing: **sticky sessions are almost always compensating for a backend that isn't actually stateless**, and that has real downstream costs — uneven load distribution (a backend that happens to be "sticky" for several heavy users gets disproportionate load with no rebalancing lever), harder failover (losing that backend loses whatever state was only local to it), and messier deploys (can't freely cycle instances without disrupting pinned clients).

The generally better fix is moving the state out of the backend (into a shared cache/store — [Caching §8](Caching.md#8-multi-layer-caching-and-cross-layer-coherence "Multi-layer caching and cross-layer coherence")'s distributed cache case, or a broker subscription as in the real-time transport docs) so *any* backend can serve *any* client and sticky routing becomes unnecessary rather than compensated-for. Sticky sessions are a legitimate short-term tool, but proposing them without naming the underlying statefulness they're covering for is a weaker answer than proposing statelessness directly.

---

## 8. The load balancer as a single point of failure

It's easy to design redundancy into the backend tier and forget that the load balancer sitting in front of it is itself a component that can fail — and if it's a single instance, it just moved the single point of failure one layer up instead of removing it. Real answers: managed cloud load balancers (ALB/NLB, GCP LB) are themselves built multi-zone/highly-available by the provider; a self-hosted load balancer tier (HAProxy/Envoy/Nginx) needs its own redundancy story — typically a floating/virtual IP with active-passive failover, or multiple LB instances behind DNS/Anycast at the layer above. Worth asking "is the load balancer itself highly available" as a standing question whenever a design leans on one, rather than treating it as inherently reliable infrastructure.

---

## 9. What separates senior from staff here

- **Senior**: knows L4 vs. L7, can name round robin/least-connections/weighted/hash-based, knows health checks exist.
- **Staff**: picks a routing strategy by reasoning about the actual cost/load-signal shape of the workload rather than defaulting to a commonly-cited one; distinguishes liveness from readiness and names the correlated-failure risk of deep health checks; treats long-lived connections as breaking the request-rate model entirely (pinning, connection draining, resource-based load signals); frames sticky sessions as compensating for backend statefulness rather than a neutral feature; and treats the load balancer itself as a component with its own availability requirement, not an assumed-reliable given.

---

## 10. Questions to expect, and what a strong answer sounds like

- **"Your least-connections load balancer is sending uneven traffic to your WebSocket gateway fleet — why?"** → Connection count isn't the same as load for long-lived, mostly-idle connections; need a load signal tied to actual resource use (active messages/sec, memory), not raw connection count.
- **"A backend's health check passes but it's still serving 500s to real traffic — what's wrong?"** → The check is too shallow (TCP or a trivial `/health` endpoint) and doesn't exercise the actual failing dependency path; move to a deep readiness check — while flagging the correlated-failure risk if many instances share that dependency.
- **"You mark a region unhealthy — how fast do users actually stop being routed there?"** → Not instant if failover is DNS-based; bounded by TTL and resolver/client caching behavior; Anycast fails over faster but requires different underlying infrastructure.
- **"Why do you have sticky sessions here, and what's the downside?"** → Identify what backend-local state is being preserved, and propose moving it to shared storage as the real fix rather than treating stickiness as free.
- **"Is your load balancer itself a single point of failure?"** → Explicitly address whether it's a managed multi-AZ service or a self-hosted instance needing its own HA story.
