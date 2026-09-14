# REST

## Why this topic is a filter question

Everyone can say "REST uses HTTP verbs and nouns." The filter is whether you can defend the *hard* design decisions that real REST APIs force on you: how you handle partial failure in a multi-resource operation, how you version without breaking every client at once, how pagination behaves under concurrent writes, and when REST's resource model actively fights the problem you're modeling. That's where this doc spends its time.

---

## 1. The core model

REST maps a domain onto **resources** (nouns, identified by URLs) manipulated through a small, uniform set of HTTP methods. The contract clients rely on isn't just "JSON over HTTP" — it's that method semantics are predictable across every resource in the API without reading extra documentation:

| Method | Semantics | Idempotent? | Safe (no side effects)? |
|---|---|---|---|
| `GET` | Read | Yes | Yes |
| `PUT` | Full replace | Yes | No |
| `PATCH` | Partial update | Not by default | No |
| `DELETE` | Remove | Yes | No |
| `POST` | Create, or a non-idempotent action | No, unless you add a mechanism | No |

"Idempotent" here has a precise meaning that's worth being able to state exactly: **sending the same request N times has the same effect on server state as sending it once.** This is a statement about server state, not about the response — a `DELETE` that returns `404` on the second call is still idempotent, because the resource is gone either way.

The gap that matters: `POST` is not idempotent by default, and `POST` is exactly the method you use for the operation most vulnerable to duplication — "create." A client that times out waiting for a `POST /orders` response and retries can create two orders. This is the single most common REST correctness bug in production systems, and it's why idempotency keys exist ([§2](#2-idempotency-keys-how-they-actually-work "Idempotency keys — how they actually work")).

---

## 2. Idempotency keys — how they actually work

An `Idempotency-Key` header lets a client mark a `POST` as safe to retry. The mechanics matter more than the concept:

1. Client generates a unique key (usually a UUID) **once**, before the first attempt — not regenerated on retry.
2. Server, on receiving a request with that key, checks a store (Redis, a dedicated table) for a prior result under that key.
3. If found: return the **stored response**, don't re-execute the operation. This is the part people get wrong — it's not enough to just avoid a duplicate row; the retry needs to get back the *same response* the original call would have produced (same order ID, same status), or the client can't reconcile state.
4. If not found: execute the operation, atomically store `(key → response)` before or alongside committing the result, then return it.
5. Keys expire after a bounded window (commonly 24h) — long enough to outlast realistic client retry behavior, short enough to bound storage.

The subtle failure mode: what if two requests with the same key arrive **concurrently** (client fired a retry before the first attempt's response came back)? Without a lock or a unique constraint on the key at the database level, both can pass the "not found" check and both execute. The fix is a unique constraint on the idempotency key column combined with `INSERT ... ON CONFLICT`, not just an application-level cache check — this is a good concrete detail to volunteer if asked to design this.

---

## 3. Partial updates and concurrency control

`PATCH` raises two questions REST doesn't answer for you:

**What does "partial" mean?** Two real conventions, and they behave differently:
- **JSON Merge Patch** (RFC 7396): send only the fields you want to change; `null` means "delete this field." Simple, but can't express "remove this array element" cleanly and can't distinguish "don't touch this field" from "the client's language serializes absent fields as null."
- **JSON Patch** (RFC 6902): a list of explicit operations (`add`, `remove`, `replace`, `test` at a path). More expressive and precise, more verbose, less commonly implemented by client tooling.

Most APIs use an ad hoc "send the fields that changed" convention that's really merge-patch without saying so — worth naming this explicitly if you're describing a design, since it has the same null-vs-absent ambiguity.

**What happens under concurrent writes?** Two clients `PATCH` the same resource seconds apart — last-write-wins silently discards one of them, which is often wrong ("two support agents both resolve conflicting fields on the same ticket"). The standard fix is **optimistic concurrency control** via `ETag` / `If-Match`:
1. `GET` returns the current representation plus an `ETag` (a hash or version of the resource).
2. Client sends `PATCH` with `If-Match: <etag>`.
3. Server rejects with `412 Precondition Failed` if the resource's current ETag doesn't match — the client refetches, reapplies its intended change, retries.

This is a detail that separates candidates who've operated a real multi-writer API from candidates who haven't: they don't wait to be asked "what if two people edit at once," they raise it while describing the write path.

---

## 4. Pagination — where the "just use cursors" answer needs to go deeper

The standard senior-level answer is "cursor pagination over offset because offset is unstable at scale." That's correct but incomplete — the interesting part is *why*, precisely:

- **Offset pagination** (`?offset=100&limit=20`) asks the database "skip 100 rows, give me 20" on every request. Two problems: (1) the database still has to scan and discard the skipped rows, so cost grows with offset — page 500 is meaningfully slower than page 1 on a large table; (2) if rows are inserted or deleted ahead of the current offset while the client is paging, rows shift, and the client can see duplicates or skip rows entirely mid-scroll. This is a *correctness* bug, not just a performance one, and it's the detail worth stating unprompted.
- **Cursor pagination** (`?cursor=<opaque token>&limit=20`) encodes a position — typically the sort key's value of the last item seen (e.g., `(created_at, id)` to break ties) — rather than a row count. The next query becomes `WHERE (created_at, id) > (last_created_at, last_id) ORDER BY created_at, id LIMIT 20`, which is a direct index seek regardless of how deep into the collection you are, and it isn't perturbed by inserts/deletes elsewhere in the collection.

The tradeoff you give up with cursors: no "jump to page 14," and the cursor must be opaque and stable, which means you can't freely change the underlying sort order without invalidating outstanding cursors. Worth naming that tradeoff rather than presenting cursors as a free win.

---

## 5. Versioning — what actually breaks clients

"Use `/v1/`" is the surface answer. The real design question is: **which changes are breaking, and how do you ship a breaking change without a coordinated flag-day migration for every client?**

Non-breaking (safe to ship without a version bump):
- Adding a new optional field to a response
- Adding a new endpoint
- Adding a new optional request parameter with a sensible default

Breaking (needs a version boundary or a deprecation path):
- Removing or renaming a field
- Changing a field's type or meaning
- Changing validation to reject previously-accepted input
- Changing default behavior

For breaking changes, the pattern that scales past "just bump `/v1/` to `/v2/`" is **expand-contract**: add the new field/behavior alongside the old one (expand), migrate clients at their own pace while both are live, monitor usage of the old field, then remove it once usage hits zero (contract). This avoids forcing every consumer — including ones you don't control, like a partner integration — onto a hard cutover date. A full `/v2/` is usually reserved for a critical mass of breaking changes, not every individual field tweak, because standing up and maintaining two parallel API surfaces is expensive.

---

## 6. Over-fetching, under-fetching, and chattiness

This is REST's most legitimate weakness relative to alternatives, and it's worth being precise about the actual failure mode rather than reciting "REST can be chatty":

- A mobile client rendering an order summary screen needs the order, the customer name, and the first product image. A resource-oriented API gives you `GET /orders/{id}`, then `GET /customers/{id}`, then `GET /products/{id}` — three round trips, each with its own latency and TLS/auth overhead, dominated by whichever is slowest (or, on a high-latency mobile network, by the sum if done serially).
- Fixes, each with a real cost:
  - **Compound/expansion endpoints**: `GET /orders/{id}?expand=customer,items.product` — pragmatic, but the server now has to support an increasingly wide matrix of expansion combinations.
  - **BFF (Backend-for-Frontend)**: a purpose-built aggregation layer per client type that calls the underlying resource APIs server-side (low-latency internal network) and returns exactly what that client screen needs. Common answer at companies with multiple client platforms with different data needs.
  - **GraphQL**: lets the client specify the exact shape it needs in one request. Solves over/under-fetching directly, at the cost of a more complex server (resolver N+1 query risk, harder HTTP-level caching, more work to secure/rate-limit meaningfully since every query is different).

A strong answer names which of these fits the actual client population — BFF when you control a handful of client apps with stable, different needs; GraphQL when you have many heterogeneous consumers with unpredictable data needs (e.g., an internal platform serving many product teams).

---

## 7. Status codes and error contracts

Getting codes technically right (`404` vs `403` vs `409`) is table stakes. The staff-level layer on top: **`403` vs `404` is a security decision, not just a semantic one.** Returning `404` for a resource that exists but the caller can't access (instead of `403`) is a deliberate choice to avoid leaking existence of resources to unauthorized callers — e.g., you generally don't want `GET /orders/12345` to reveal via `403` that order 12345 exists at all if the caller has no relationship to it. Worth naming as a conscious tradeoff (usability of error messages vs. information leakage) rather than an arbitrary pick.

Beyond the numeric code, error responses need a **structured, machine-parseable body** consistently shaped across the whole API (a stable `code`/`message`/`details` envelope, e.g. following RFC 7807 Problem Details), so client error handling doesn't need per-endpoint special-casing. Rate limiting (`429`) should include `Retry-After` so well-behaved clients back off correctly instead of hammering immediately.

---

## 8. REST vs. the alternatives

| | REST | gRPC | GraphQL | WebSockets/SSE |
|---|---|---|---|---|
| Best for | Public/partner APIs, resource CRUD | Internal service-to-service, high QPS, typed contracts | Flexible client-driven queries over many resource types | Real-time push/bidirectional |
| Typing | Loose (OpenAPI optional) | Strong (Protobuf, generated code) | Strong (schema, but resolved at query time) | None inherent |
| Caching | Native HTTP caching (`ETag`, CDN) | None built-in | Hard (single endpoint, varying queries) | N/A |
| Debuggability | High — curl, browser devtools | Lower — needs proto-aware tooling | Medium | Medium |

See [RPC and gRPC](RPC%20and%20gRPC.md) for the deeper internal-service comparison, and [Websockets](Websockets.md) / [Server-sent events (SSE)](Server-sent%20events%20%28SSE%29.md) for when push replaces pull entirely.

---

## 9. What separates senior from staff here

- **Senior**: correct verbs, correct status codes, knows cursor > offset pagination, knows REST vs gRPC at a high level.
- **Staff**: treats idempotency keys as a concurrency-control mechanism, not just a header to mention; explains *why* offset pagination corrupts under concurrent writes, not just that it's "unstable"; designs versioning as an ongoing migration process (expand-contract) rather than a URL prefix; makes the `403` vs `404` call deliberately; picks BFF vs GraphQL vs expansion params based on actual client diversity instead of defaulting to whichever is trendy.

---

## 10. Questions to expect, and what a strong answer sounds like

- **"A client says they created two orders from one click — how?"** → Non-idempotent `POST` plus a client-side retry or double-submit; walk through the idempotency key mechanism, including the concurrent-request race and the unique-constraint fix.
- **"Your API returns page 40 of a list and the user complains about seeing duplicate rows."** → Offset pagination drifting under concurrent inserts; move to cursor-based pagination keyed on a stable, unique sort tuple.
- **"How do you add a required field to a request without breaking existing clients?"** → You don't, directly — expand-contract: ship it optional with a default first, migrate/monitor callers, only make it required (or cut a new version) once old callers are gone.
- **"Two agents edit the same support ticket at once — what happens?"** → Optimistic concurrency via `ETag`/`If-Match`; explain the `412` retry loop.
- **"This mobile screen needs data from four resources — do you make four calls?"** → Discuss BFF/expansion/GraphQL tradeoffs tied to how many client types exist and how much their data needs vary.
