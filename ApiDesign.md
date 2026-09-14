# API Design

## Why this topic is a filter question

Every candidate can list HTTP verbs and status codes. The filter is whether you can design the parts of an API that only bite you in production: authorization that's actually enforced server-side and can't be bypassed by a crafted request, idempotency that survives a concurrent retry race, tenant isolation that holds up against a deliberately malicious client, and rate limiting that works correctly across a fleet of stateless servers instead of just on one box. This doc assumes you already know REST mechanics (see [REST](Building%20blocks/Communication/REST.md) for verb semantics, pagination mechanics, and versioning) and goes straight at the harder cross-cutting concerns: auth, isolation, concurrency, and organizational scale.

---

## 1. Resource modeling — the one-line version, with the real caveat

Model nouns, use verbs as HTTP methods; reach for action-style endpoints (`POST /jobs/{id}:cancel`) only when the operation genuinely isn't a CRUD transition on a resource's state. The caveat worth stating explicitly: **over-fitting everything into CRUD is its own mistake.** Forcing "approve this invoice" into `PATCH /invoices/{id}` with `{"status":"approved"}` loses the ability to validate the specific transition (who can approve, under what invoice state, with what side effects) at the API boundary — an explicit `POST /invoices/{id}:approve` endpoint can enforce state-machine rules and emit a specific audit event, where a generic `PATCH` can't distinguish "approve" from "some other update that happens to also set status." Pick the resource-vs-action modeling per operation based on whether the operation has its own validation/audit/side-effect surface, not as a blanket rule.

---

## 2. Authentication vs. authorization — and where each actually gets enforced

Authentication answers "who is this," authorization answers "what can they do." The failure mode worth naming explicitly: **frontend permission checks are a UX feature, not a security control.** A permissions payload sent to the client so it can hide a delete button is fine and good UX — it is never a substitute for the server independently re-checking authorization on every request, because a client is fully under the attacker's control (a modified request, a replayed token, a direct API call bypassing the UI entirely). Any API where "the button was hidden" is doing security work has a vulnerability.

### RBAC vs. ABAC vs. PBAC — picking one is a maintainability decision, not a features decision

- **RBAC** ("can this user do X because of their *role*?"): cheap to reason about, cheap to audit ("who can delete tags → anyone with `admin`"), but coarse — it can't express "only if you own it" or "only during business hours" without either exploding the role count (`tag_owner_admin`, `tag_owner_viewer`, ...) or leaking that logic into application code anyway, defeating the point of centralizing it.
- **ABAC** ("can this user do X because their and the resource's *attributes* satisfy a rule?", e.g., `user.tenantId == resource.tenantId && resource.ownerId == user.id`): expresses ownership/tenant/context rules RBAC can't, at the cost of rules that are harder to enumerate and audit — "who can delete tag 123" now requires evaluating the rule against that specific resource's attributes rather than reading a static role list.
- **PBAC**: externalizes the rules into a policy engine (e.g., Open Policy Agent) evaluated against a structured request (subject, resource, action, context) rather than hand-written `if` statements scattered through the codebase — the real win is **auditability and centralization** (one place to see and change every access rule) and **consistency across services** (many services call the same policy engine instead of each reimplementing similar-but-subtly-different checks), at the cost of an added network hop (policy evaluation latency) and a new critical-path dependency whose own availability now matters.

The staff-level answer isn't "PBAC is more sophisticated so use it" — it's matching the model to how the org's access rules actually grow: a small system with stable, coarse roles doesn't need a policy engine; a system with ownership/tenant/context-dependent rules across many services benefits from centralizing that logic once it starts getting reimplemented inconsistently in three different codebases.

---

## 3. Multi-tenancy and isolation — assume an adversarial client, not just a well-behaved one

This is one of the highest-signal areas to go deep on, because the failure mode is a real, common vulnerability class (broken object-level authorization, effectively an IDOR — insecure direct object reference), not a hypothetical:

- **Never trust a tenant/owner identifier supplied by the client** — in the URL, the query string, or the body. `GET /invoices/{id}` must check that the authenticated caller's tenant matches invoice `{id}`'s actual tenant *server-side*, every single time, even if the ID "looks like" it belongs to them. The attack this defends against is trivial to execute: enumerate or guess adjacent IDs (`invoice_1001`, `invoice_1002`, ...) and see which ones return data instead of a `403`/`404`.
- **`403` vs. `404` for cross-tenant access is a deliberate leak-prevention choice**, same reasoning as [REST §7](Building%20blocks/Communication/REST.md#7-status-codes-and-error-contracts "Status codes and error contracts") — returning `404` for "exists but you can't see it" avoids confirming to an attacker that a given ID is valid at all.
- **Tenant scoping has to reach the data layer, not stop at the API layer.** If every query needs an explicit `WHERE tenant_id = ?`, a single missing clause in one code path is a full cross-tenant data leak — this is exactly the kind of bug that a **row-level security policy at the database** (enforced regardless of which application code path issues the query) or a **query-builder/ORM layer that makes tenant scoping structurally mandatory** (impossible to construct a query without it) closes far more reliably than "we remember to add the filter everywhere."
- **Pagination/index design changes under tenant partitioning** — if data is partitioned by `tenant_id` (see [Replication and Partitioning §2](Building%20blocks/Data%20and%20Consistency/Replication%20and%20Partitioning.md#2-partitioning-the-key-choice-is-the-whole-design "Partitioning: the key choice is the whole design")), a list endpoint scoped to one tenant is a clean single-partition query; a hypothetical cross-tenant admin query becomes a scatter-gather, which is worth flagging as a real cost of an "admins can see everything" requirement rather than assuming it's free.

---

## 4. Idempotency — the mechanism, and where it actually gets designed wrong

The concept ("retries should be safe") is table stakes; see [REST §2](Building%20blocks/Communication/REST.md#2-idempotency-keys-how-they-actually-work "Idempotency keys — how they actually work") for the full mechanics of the idempotency-key pattern, including the concurrent-request race and why it needs a unique constraint, not just an application-level cache check. Two API-design-specific points worth adding here:

- **Scope the key correctly.** An idempotency key should be scoped per-endpoint (or per-endpoint-and-tenant) — reusing the *same* key across two different operations (e.g., a client library bug that reuses a UUID for both a "create order" and a later "cancel order" call) must not let the second call resolve against the first's cached result. The dedup lookup should key on `(endpoint, idempotency_key)`, not `idempotency_key` alone.
- **Dedup windows have a real cost/correctness tradeoff.** Retaining every idempotency key forever is unbounded storage growth; expiring too early means a legitimately slow retry (client waited 25 hours before retrying after a 24-hour TTL) creates a duplicate anyway. Size the window against realistic client retry behavior (e.g., mobile clients retrying after being offline), not an arbitrary round number.

---

## 5. Concurrency control — same mechanism as caching's hardest problem, different layer

Optimistic concurrency via `ETag`/`If-Match` (see [REST §3](Building%20blocks/Communication/REST.md#3-partial-updates-and-concurrency-control "Partial updates and concurrency control")) is the standard tool. The design decision underneath "do we use optimistic locking" is: **what should happen when two writers actually conflict** — reject-and-let-the-client-retry (safest, worst UX — the second writer's work is discarded and they have to redo it), last-write-wins (simplest, silently loses data, only acceptable when the field genuinely doesn't need conflict detection), or field-level merge (best UX, meaningfully more implementation work — you need to know which fields are safe to merge independently vs. which represent one atomic decision that can't be split). This is the same shape of decision as [Consistency and CAP §4](Building%20blocks/Data%20and%20Consistency/Consistency%20and%20CAP.md#4-conflict-resolution-when-replicas-diverge "Conflict resolution when replicas diverge")'s conflict-resolution strategies, just surfaced at the API layer instead of the storage-replica layer — worth drawing that connection explicitly if asked.

---

## 6. Async APIs — polling vs. webhooks, and the idempotency problem this creates for the *client*

The `202 Accepted` + poll-a-status-URL pattern is standard for long-running work. Two things worth adding beyond the mechanics:

- **Webhooks flip the reliability problem onto the API provider.** If you notify a client via a webhook instead of making them poll, you've become a message producer with the exact same at-least-once delivery obligations as a message broker (see [Message Queues and PubSub §3](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#3-delivery-semantics-the-part-interviewers-actually-probe "Delivery semantics — the part interviewers actually probe")) — the client's webhook endpoint can be down, slow, or flaky, so you need retry-with-backoff on the sender side, and the client needs to handle the webhook arriving more than once (a delivery attempt ID the client can dedupe on is standard, exactly analogous to an idempotency key, just running in the opposite direction).
- **Polling has its own design decision**: a fixed poll interval trades freshness for load, same tradeoff as [Long Polling §1](Building%20blocks/Communication/Long%20Polling.md#1-the-mechanism "The mechanism") vs. plain polling — for a job-status endpoint specifically, returning a suggested poll interval in the response (`{"status":"pending","pollAfterMs":2000}`) lets you tune client behavior server-side without a client deploy.

---

## 7. Rate limiting — the algorithm and the distributed-state problem

"Return `429`" is the surface answer. The actual design has two layers:

**Algorithm** (what counts as "too many"):
- **Fixed window** (e.g., "100 requests per minute, reset on the minute"): simplest, but allows a burst of 2x the limit right at the window boundary (100 requests in the last second of one window, another 100 in the first second of the next).
- **Sliding window / sliding window log**: tracks requests in a rolling window rather than a fixed reset point — smooths out the boundary-burst problem, at the cost of more state to track per client.
- **Token bucket**: a bucket refills at a steady rate and each request consumes a token; naturally allows bursts up to the bucket size while enforcing a steady-state average rate — this is the most common real-world choice because it matches how clients actually behave (bursty, not perfectly steady) while still bounding sustained rate.

**The distributed problem**: rate limiting is meaningless if each of your N stateless API servers independently tracks "requests from this client" in local memory — a client gets N times the intended limit by virtue of being load-balanced across N servers. The fix is a **shared counter store** (Redis is the standard answer, using atomic increment-with-expiry or a Lua script for the token-bucket check) so the limit is enforced against one shared view of the client's request history, not N independent, under-counting views. This is worth stating explicitly, because "we rate limit at 100 req/min" is a materially different (and wrong) claim if it's actually 100 req/min *per server*.

---

## 8. Versioning and organizational-scale API governance

The mechanics (URI vs. header versioning, additive-first, expand-contract) are covered in [REST §5](Building%20blocks/Communication/REST.md#5-versioning-what-actually-breaks-clients "Versioning — what actually breaks clients"). The layer specific to *API design as an organizational discipline*, worth raising when a design needs to serve multiple teams:

- **A style guide alone doesn't hold** across many teams over time without enforcement — real orgs back it with **API linting in CI** (e.g., Spectral against an OpenAPI spec) that fails a PR introducing an inconsistent naming convention or a missing pagination pattern, rather than relying on code review discipline alone.
- **A schema/contract registry** (OpenAPI/Protobuf definitions checked into a central, versioned location) lets consuming teams detect a breaking change *before* it ships, via automated contract diffing in CI, rather than discovering it in production.
- **Decide up front what's centrally mandated vs. team-discretion** — pagination shape, error envelope, and auth header conventions are worth mandating org-wide (client tooling can be shared only if these are consistent); endpoint-specific business logic obviously isn't. Naming this split explicitly (rather than "we'll have a style guide") is what actually holds up as an org scales past a handful of teams.

---

## 9. Observability and auditability — SLOs, not just dashboards

Beyond "add logs and metrics": a mature API defines **SLOs per endpoint or endpoint class** (e.g., "reads: p99 < 200ms, 99.9% availability; the checkout write path: p99 < 500ms, 99.95% availability") and treats those as the actual target, with an **error budget** — a defined amount of acceptable failure per period — that governs how aggressively you can ship risky changes (a healthy error budget means room to take some risk; a nearly-exhausted one means freeze and stabilize). This reframes observability from "we have dashboards" to "we have a number we're accountable to," which is the distinction interviewers are listening for. Audit logs for security-sensitive actions (who deleted what, when, from where) are a separate, compliance-driven concern from operational metrics — worth not conflating the two, since audit logs typically need longer retention, different access controls (an engineer debugging latency shouldn't need audit-log access, and vice versa), and tamper-resistance guarantees operational logs don't.

---

## 10. Worked example: tag API with the hard parts made explicit

```
POST /api/v1/tags
Authorization: Bearer <token>
Idempotency-Key: 6c9f2e...

{ "tagName": "business_expense", "contentId": "JST-123" }
```

- **Tenant**: derived from the authenticated token, never from the body — closes the IDOR risk in [§3](#3-multi-tenancy-and-isolation-assume-an-adversarial-client-not-just-a-well-behaved-one "Multi-tenancy and isolation — assume an adversarial client, not just a well-behaved one").
- **Idempotency**: keyed on `(POST /tags, 6c9f2e...)`; a concurrent duplicate submission (client double-click, or a client-side retry firing before the first response returns) is resolved by a unique constraint on that composite key at the database, not just an in-memory check ([§4](#4-idempotency-the-mechanism-and-where-it-actually-gets-designed-wrong "Idempotency — the mechanism, and where it actually gets designed wrong"), and [REST §2](Building%20blocks/Communication/REST.md#2-idempotency-keys-how-they-actually-work "Idempotency keys — how they actually work")).
- **Authorization**: `tag:create` permission check server-side, independent of whether the UI happened to show the "create" button.
- **Response**: `201 Created` with the resource, a stable `tagId`, and (if this is a write with downstream side effects worth tracing) enough context in structured logs to trace the write end-to-end.

```
DELETE /api/v1/tags/tag_123
Authorization: Bearer <token>
```

- **Authorization**: `tag:delete`, likely restricted beyond ownership (e.g., admin-only) given deletion is harder to undo than a create.
- **Cross-tenant check**: server verifies `tag_123`'s tenant matches the caller's tenant before deleting — returns `404`, not `403`, if it doesn't ([§3](#3-multi-tenancy-and-isolation-assume-an-adversarial-client-not-just-a-well-behaved-one "Multi-tenancy and isolation — assume an adversarial client, not just a well-behaved one")).
- **Audit**: a destructive action gets its own audit event (who, when, what) distinct from ordinary operational logging ([§9](#9-observability-and-auditability-slos-not-just-dashboards "Observability and auditability — SLOs, not just dashboards")).
- **Response**: `204 No Content` — and if this delete is a soft-delete with async cleanup fan-out, that's a case for the `202`-plus-status-URL pattern ([§6](#6-async-apis-polling-vs-webhooks-and-the-idempotency-problem-this-creates-for-the-client "Async APIs — polling vs. webhooks, and the idempotency problem this creates for the client")) instead of a synchronous `204`, worth naming as a real design fork rather than assuming synchronous delete is always right.

---

## 11. What separates senior from staff here

- **Senior**: models resources cleanly, knows the RBAC/ABAC distinction, knows idempotency keys and `ETag`/`If-Match` exist, knows to rate limit and version.
- **Staff**: treats frontend permission checks as UX-only and never a security boundary; picks RBAC vs. ABAC vs. PBAC based on how the org's access rules actually grow, not sophistication for its own sake; assumes an adversarial client for tenant isolation and closes the IDOR path at the data layer, not just the API layer; recognizes that a webhook-based async API inherits a message producer's delivery obligations; knows rate limiting is meaningless without shared state across a fleet and can name the algorithm tradeoff (fixed window vs. token bucket); and treats API governance at multi-team scale as an enforcement problem (linting, contract diffing) rather than a documentation problem.

---

## 12. Questions to expect, and what a strong answer sounds like

- **"A user changes a `tenantId` field in the request body to another tenant's ID — what happens?"** → Nothing, because tenant is derived from the authenticated session server-side and never trusted from client input — this is the question that reveals whether tenant scoping is real or cosmetic.
- **"How do you rate limit correctly when you have 50 stateless API servers behind a load balancer?"** → Shared counter store (Redis), not per-server in-memory counting, or the effective limit is 50x intended.
- **"Two admins `PATCH` the same resource within a second of each other — what's the actual UX?"** → Name the real choice (reject-and-retry vs. last-write-wins vs. field merge) and justify it based on whether the fields represent independent or atomic decisions.
- **"You're notifying clients via webhook instead of having them poll — what's different about your reliability obligations?"** → You're now the producer in an at-least-once delivery system; need retries with backoff on send, and the client needs a delivery-attempt ID to dedupe.
- **"Five teams consume this API — how do you keep it consistent without a full-time committee reviewing every PR?"** → Automated linting against a shared style guide and contract-diffing in CI, with an explicit list of what's centrally mandated vs. team discretion.
