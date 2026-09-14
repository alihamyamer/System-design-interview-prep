# Caching

## Why this topic is a filter question

"Add Redis" is the single most reflexive answer in system design interviews, and it's exactly why this topic filters hard. The real skill isn't knowing caching exists — it's knowing precisely what staleness a specific field can tolerate, what happens when a hot key evicts everything around it, how a cache and its source of truth can drift out of sync under perfectly normal concurrent access (not just failure), and when caching is actually masking a correctness problem instead of solving a performance one.

---

## 1. The core tradeoff, stated precisely

A cache trades **correctness risk (staleness)** for **latency and load reduction**. Every caching decision is really answering one question: *how stale can this specific piece of data be before that staleness causes real harm* — and the answer differs field by field within the same entity, not just system by system (see [§9](#9-worked-example-same-entity-different-guarantees-per-field "Worked example: same entity, different guarantees per field")'s worked example). Treating "we'll cache this" as one decision per entity, rather than per field per access pattern, is the most common shallow answer here.

---

## 2. Caching patterns — mechanics and the failure mode each one actually has

- **Cache-aside (lazy loading)**: app checks cache; on miss, reads the source, populates the cache, returns. Simplest, most common. **Its real failure mode isn't the well-known "first request is slow"** — it's a race condition under concurrent writes (see [§5](#5-the-concurrency-race-that-makes-cache-aside-genuinely-hard-not-just-sometimes-stale 'The concurrency race that makes cache-aside genuinely hard, not just "sometimes stale"')).
- **Write-through**: writes go to the cache and the backing store together (typically: write store, then update cache, or vice versa, within the write path). Keeps the cache warm and correct immediately after writes, at the cost of extra write latency and wasted cache churn for data that's rarely read.
- **Write-behind (write-back)**: write lands in the cache first and is asynchronously flushed to the backing store later. Fast writes, but the cache is now **briefly the only copy of unflushed data** — if it fails before flushing, that write is gone. This is a durability tradeoff, not just a performance one, and should be named as such: write-behind is a reasonable choice for something like a view counter (losing a few increments on a crash is fine) and a dangerous choice for anything with business/financial consequence.
- **Read-through**: the cache library/layer itself owns fetching from the backing store on a miss, so application code just asks the cache — simpler call sites, but you lose fine-grained control over exactly when and how the fallback fetch happens (harder to add per-call logic like "only cache if this specific condition holds").

---

## 3. Invalidation — the actually-hard part, in mechanism, not just in name

Every "how do you keep the cache correct" answer boils down to one of these, each with a specific real cost:

- **TTL expiration**: simplest, needs no coordination — but staleness is bounded only by the TTL, and a short TTL to reduce staleness risk directly increases load on the source (more frequent re-fetches), so TTL choice is itself a latency/correctness/load three-way tradeoff, not a free knob.
- **Explicit invalidation on write**: delete or update the cache key when the underlying write happens. Better freshness, but now correctness depends on *every* code path that writes that data remembering to also invalidate the cache — a second write path added later (a backfill script, an admin tool, a different service writing to the same table) that forgets this is a very common real production bug, and it's worth naming that this pattern has a hidden coupling cost: every writer needs to know about the cache.
- **Event-driven invalidation**: a change event (from a message bus, or CDC off the database log — see [Message Queues and PubSub §3](../Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#3-delivery-semantics-the-part-interviewers-actually-probe "Delivery semantics — the part interviewers actually probe")'s outbox/CDC discussion) triggers invalidation, decoupling "something wrote to the DB" from "something remembered to clear the cache." More robust than scattering invalidation calls across every writer, more infrastructure to run.
- **Versioned/keyed-by-version cache keys**: include a version or content hash in the key itself (`product:123:v7`) so old versions simply become unreferenced and age out via normal eviction rather than needing active deletion — trades a small amount of cache-space waste (old versions linger until evicted) for never needing a correctness-critical "did the invalidation actually happen" step.

---

## 4. The cache stampede — and the difference between the two related-but-distinct problems people conflate

- **Cache stampede (thundering herd on a hot key)**: a popular key expires, and many concurrent requests all miss at once and all hit the source simultaneously — the source, which the cache exists specifically to protect, gets hit with a spike sized to your *entire* concurrent request volume in that instant, at exactly the moment it's least prepared for it.
  - **Request coalescing** (a.k.a. single-flight): the first miss triggers the actual fetch; concurrent requests for the *same* key block/wait on that in-flight fetch instead of independently hitting the source, then all get served from its result once it completes.
  - **Locking around refill**: similar idea via an explicit lock/mutex per key held during the refill.
  - **Jittered TTLs**: instead of every instance of a key expiring at exactly the same wall-clock moment (common when many keys are seeded at the same time, e.g., a deploy or a cache warm), add random jitter to each key's TTL so expirations spread out over time instead of clustering.
  - **Probabilistic early expiration**: refresh a key *before* it actually expires, with a probability that increases as it approaches its TTL — smooths refresh load over time instead of concentrating it at the exact expiry instant.
- **Cold cache / cache warm-up**: after a full cache flush or a fresh deploy, the *entire* dataset is cold simultaneously — every key, not just a hot one, misses at once. This is a different problem needing a different fix: pre-warming the cache before cutting traffic to it, or rate-limiting/circuit-breaking the source so a full-cold-cache event degrades gracefully instead of taking the source down entirely.

---

## 5. The concurrency race that makes cache-aside genuinely hard, not just "sometimes stale"

This is the detail that separates a candidate who's actually debugged a caching bug from one who's read about caching:

1. Request A reads key `X`, misses, starts fetching from the DB.
2. Before A's fetch returns, a write updates `X` in the DB and invalidates the cache (or the cache had nothing to invalidate yet).
3. A's fetch — which read the DB *before* the write — completes and writes the **old** value into the cache.
4. The cache now holds a stale value, and (with TTL-only expiration) it will keep serving it until TTL expiry, even though the DB has the correct newer value the whole time.

This isn't a rare edge case — it's a structural race in the cache-aside pattern under any concurrent read+write traffic on the same key. Mitigations:
- **Short TTLs as a backstop** (bounds how long the bad state can persist, doesn't prevent it).
- **Versioned writes**: only let a cache write succeed if it's writing a version at least as new as what's already there (compare-and-set against a version/timestamp stored alongside the cached value) — this directly closes the race, because A's stale write gets rejected once it carries an older version than what the invalidation-driven write already installed.
- **Delete instead of populate, on invalidation** (and let the next reader repopulate) reduces the race window somewhat but doesn't eliminate it if a fetch is already in flight when the delete happens.

Being able to describe this race unprompted — not just "caching can be stale" — is one of the highest-signal moves in this topic.

---

## 6. Eviction policies — not just "LRU"

When a cache is full, what gets evicted determines effective hit rate as much as cache size does:
- **LRU (least recently used)**: evict the item not accessed longest. Simple, generally solid, but vulnerable to a single large scan (e.g., a batch job reading a huge range once) evicting your entire actually-hot working set, since a one-time scan touches everything "recently."
- **LFU (least frequently used)**: evict by access count instead of recency — resistant to the scan problem above, but slow to adapt when access patterns shift (an item that was frequently accessed yesterday can crowd out today's genuinely hot items if the count doesn't decay).
- **TinyLFU / ARC and similar adaptive policies**: used by modern caching libraries (e.g., Caffeine) specifically to get LFU's scan-resistance without its stale-popularity problem, by combining frequency estimation with recency and using it as an *admission* filter (should this new item even be let into the cache) rather than only an eviction rule.

Knowing that eviction policy is a real, differentiated design lever — not an implementation detail you accept from whatever library default ships — is worth a sentence even if you don't go implement TinyLFU by hand.

---

## 7. Hot keys and cache penetration

- **Hot key**: one key gets disproportionate traffic and overloads the single cache node/shard hosting it, even though the cluster overall has capacity — this is a partitioning problem inside your cache layer, structurally identical to the [hot partition §3](../Data%20and%20Consistency/Replication%20and%20Partitioning.md#3-hot-partitions-beyond-a-celebrity-gets-a-lot-of-traffic 'Hot partitions — beyond "a celebrity gets a lot of traffic"') problem in a sharded database, and the same fix families apply: replicate the hot key across multiple nodes and read from any of them, or shard the key itself (`hot_key#0`..`hot_key#9`).
- **Cache penetration**: repeated requests for a key that **doesn't exist** in the source bypass the cache every time (there's nothing to cache — a miss is a miss) and hit the source directly, every single request — this is effectively an easy vector for accidental or deliberate load amplification against your source. **Negative caching** (explicitly cache the fact "this key doesn't exist" for a short TTL) fixes the repeated-identical-miss case. For a wider or adversarial version of this (many *different*, mostly-nonexistent keys probing your system), a **Bloom filter** in front of the cache can cheaply answer "definitely not present" without a source round-trip at all, at the cost of a small, tunable false-positive rate (it can say "maybe present" for a key that isn't, but never the reverse).

---

## 8. Multi-layer caching and cross-layer coherence

A real request path often has multiple cache layers stacked — browser cache → CDN/edge → application cache → database buffer cache — and the hard part isn't any one layer, it's that **an invalidation at the source doesn't automatically propagate to every layer above it.** Purging a CDN cache doesn't affect an already-served browser cache entry with its own TTL; invalidating an application-level Redis entry doesn't purge a CDN edge that cached the rendered response upstream of it. A design that needs strict freshness has to either explicitly propagate invalidation to every layer (CDN purge APIs, cache-busting query params/versioned URLs) or accept that end-to-end staleness is bounded by the **longest** TTL in the stack, not the shortest — worth stating explicitly rather than reasoning about only the layer you're actively designing.

---

## 9. Worked example: same entity, different guarantees per field

Reusing the product page from [Consistency and CAP §6](../Data%20and%20Consistency/Consistency%20and%20CAP.md#6-worked-example-checkout-inventory-vs-product-page-views "Worked example: checkout inventory vs. product page views") to make the "per-field, not per-entity" point concrete:

- **Product title/description/images**: cache aggressively at the CDN with a long TTL (minutes to hours) — staleness here has essentially zero cost.
- **Price**: cache at the application layer with a short TTL or explicit invalidation on price-change events — showing a stale price for even a minute has real business/legal cost (charging the wrong amount).
- **"X in stock" badge**: negative/short-TTL caching, explicitly *not* used as the number actually decremented at checkout — the badge is a UX hint, the checkout write path ([§6 in the Consistency doc](../Data%20and%20Consistency/Consistency%20and%20CAP.md#6-worked-example-checkout-inventory-vs-product-page-views "Worked example: checkout inventory vs. product page views")) goes straight to a strongly consistent source, never through this cache.
- **User's own cart contents**: read-your-writes matters here specifically — if this is cached, it must be invalidated (or bypassed) on the user's own request path immediately after their own write, even if other users' views of similar aggregate data stay on a longer-TTL cache.

One entity, one page render, four different caching decisions — this is the answer shape that demonstrates real judgment.

---

## 10. What separates senior from staff here

- **Senior**: knows cache-aside/write-through/write-behind, knows TTL and hit rate, can name Redis and CDN use cases.
- **Staff**: identifies the concurrent-write race inherent to cache-aside and names a concrete fix (versioned writes), not just "add a short TTL"; distinguishes stampede-on-a-hot-key from cold-full-cache-warm-up as different problems with different fixes; treats eviction policy as a real lever (scan resistance, adaptive admission); reasons about multi-layer cache coherence and bounds staleness by the *longest* TTL in the stack; and — most importantly — makes the caching decision per field/access-pattern rather than per entity, explicitly stating the harm of staleness for each one instead of defaulting to a single caching strategy for "the product."

---

## 11. Questions to expect, and what a strong answer sounds like

- **"Your cache-aside cache occasionally serves a stale value even though invalidation-on-write is implemented correctly — why?"** → The read-during-concurrent-write race ([§5](#5-the-concurrency-race-that-makes-cache-aside-genuinely-hard-not-just-sometimes-stale 'The concurrency race that makes cache-aside genuinely hard, not just "sometimes stale"')); walk through the exact sequence and propose versioned/compare-and-set cache writes as the fix.
- **"A popular key just expired and your database is suddenly getting hammered — what happened, and what do you do?"** → Cache stampede; request coalescing / single-flight, jittered TTLs, or probabilistic early refresh — and distinguish this from a full cold-cache scenario, which needs pre-warming instead.
- **"Someone's hitting your API with random nonexistent IDs and your database load is climbing — why does your cache not help here?"** → Cache penetration — nothing to cache on a genuine miss; negative caching for repeated identical misses, a Bloom filter if the IDs are varied/adversarial.
- **"You purge your CDN cache but users still see the old price for a few minutes — why?"** → Browser-level caching downstream of the CDN with its own TTL; end-to-end staleness is bounded by the longest TTL in the stack, not the layer you just purged.
- **"Design caching for a product page."** → Give the per-field answer from [§9](#9-worked-example-same-entity-different-guarantees-per-field "Worked example: same entity, different guarantees per field"), not one caching strategy for the whole page.
