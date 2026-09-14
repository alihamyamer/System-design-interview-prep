# Consistency and CAP

## Why this topic is a filter question

"CAP theorem says pick two of three" is the most-repeated, least-precisely-understood line in system design interviews. It's also usually recited wrong — CAP doesn't offer a menu of three properties to freely combine; it's a statement about one specific moment (a network partition) and a forced binary choice at that moment. Staff-level candidates use CAP correctly, in one sentence, and then move on to the conversation that actually matters: what consistency guarantee does *this* product need, at *this* boundary, and what's the real cost of getting it. That's where this doc spends its time.

---

## 1. What CAP actually says, precisely

CAP theorem, stated correctly: **in the presence of a network partition, a distributed system must choose between consistency and availability; it cannot have both.**

The parts people get wrong:
- **It's conditional on a partition existing.** Outside of a partition, you can absolutely have a consistent, available system — CAP says nothing about that normal-operation state. Citing CAP to justify a general "we chose availability" design decision, without a partition in the picture, is a category error interviewers will catch.
- **Partition tolerance isn't optional in a real distributed system.** Networks partition; you don't get to "choose" P the way you choose C or A — any system with more than one node across a network that can fail is implicitly partition-tolerant by necessity, or it isn't really distributed. So in practice CAP reduces to: **when a partition happens, do you choose C or A?** — not a three-way menu.
- **Consistency here means linearizability** (every read sees the most recent write, and operations appear to happen atomically at some point between invocation and response) — a much stronger and more specific claim than the colloquial "the data is correct."

### PACELC — the extension worth knowing

CAP only describes behavior *during a partition*, but partitions are rare; the tradeoff you actually make on every single request is between latency and consistency, partition or not. **PACELC** captures this: **if Partition, choose Availability or Consistency; Else (normal operation), choose Latency or Consistency.** This is the more operationally relevant framing for most interviews, because most consistency decisions you'll actually defend aren't "what do we do during a partition" — they're "do we wait for a quorum of replicas to confirm this write before returning, accepting higher latency, or return as soon as one node has it, accepting a wider staleness window." Naming PACELC unprompted is a strong signal that you understand this isn't purely a partition-time conversation.

---

## 2. Consistency models, precisely, not just by name

Vague usage ("we want eventual consistency") is a red flag. Precise definitions:

- **Linearizability (strong consistency)**: once a write is acknowledged, every subsequent read, by any client, sees that write or a later one. The strongest, most expensive guarantee — every replica effectively needs to agree on a single global order of operations.
- **Sequential consistency**: all clients see operations in *some* consistent global order, but that order doesn't have to match real-time — weaker and cheaper than linearizability, rarely the thing product requirements actually ask for by name, but useful to know exists as a middle ground.
- **Eventual consistency**: replicas converge to the same value *given no further writes and enough time* — but offers no bound on how stale a read can be in the meantime, and no ordering guarantee across reads. This is the weakest commonly-cited guarantee, and "eventual" is doing a lot of unspecified work — a good follow-up question to ask an interviewer (or ask yourself) is "eventual by how long, typically?"
- **Read-your-writes**: a specific client sees its *own* writes immediately, even if the underlying store is otherwise eventually consistent for other clients. This is usually the actual guarantee a product needs where "eventual consistency" gets casually proposed — a user who updates their profile should see the update on their own next page load, even if other users' feeds lag.
- **Monotonic reads**: once a client has observed a value, it never observes an older one afterward — prevents the "I refreshed and my comment disappeared" bug caused by a later read landing on a more-lagged replica than an earlier one.

The senior-to-staff jump here: senior engineers can define these terms; staff engineers map a specific product requirement to the *cheapest* guarantee that satisfies it, rather than defaulting to strong consistency everywhere out of caution, or eventual consistency everywhere for simplicity.

---

## 3. Quorums — the mechanism behind "tunable consistency"

Systems like Dynamo-style stores (Cassandra, DynamoDB, Riak) let you tune the consistency/availability/latency tradeoff per-operation via quorum parameters instead of picking one global mode:

- **N** = number of replicas holding a piece of data.
- **W** = number of replicas that must acknowledge a write before it's considered successful.
- **R** = number of replicas read from before returning a result to the client.

The guarantee: **if W + R > N, every read quorum overlaps with every prior write quorum by at least one replica**, guaranteeing the read sees the latest write (strong consistency for that operation) — because it's mathematically impossible to pick R replicas and W replicas out of N total without at least one node being in both sets.

Concrete example: N=3, W=2, R=2 → W+R=4 > 3, so every read is guaranteed to overlap a majority write — this is the standard "majority quorum" setup and tolerates one node being down or slow without sacrificing consistency. Compare to N=3, W=1, R=1: fast (only one replica needs to respond on either side) but reads and writes can easily miss each other entirely — no overlap guarantee, so this is an eventually-consistent, low-latency configuration by construction, not a bug. Being able to do this arithmetic live is one of the highest-signal moves available in this part of an interview.

---

## 4. Conflict resolution when replicas diverge

If more than one node can accept writes independently (multi-leader or leaderless replication — see [Replication and Partitioning §2](Replication%20and%20Partitioning.md#2-partitioning-the-key-choice-is-the-whole-design "Partitioning: the key choice is the whole design")), concurrent writes to the same key on different replicas can conflict. Resolution strategies, with their real cost:

- **Last-write-wins (LWW)**: attach a timestamp, keep the newest. Simple, but silently discards a concurrent write — fine for something like "last viewed timestamp," actively wrong for something like "items in a shopping cart" (a concurrent add can vanish).
- **Vector clocks / version vectors**: track causal history per replica so the system can detect *true* concurrency (neither write causally preceded the other) versus a clean overwrite, and surface real conflicts for resolution instead of silently picking one. More correct, meaningfully more complex to reason about and implement.
- **Application-level / semantic merge**: the application understands the data well enough to merge instead of picking a winner — e.g., a shopping cart merges as a union of items rather than overwriting.
- **CRDTs (Conflict-free Replicated Data Types)**: data structures (counters, sets, sequences) specifically designed so concurrent updates merge deterministically without coordination — used in collaborative editing and some multi-region systems, at the cost of constraining your data model to CRDT-compatible structures.

---

## 5. Consistency is a per-operation, end-to-end decision — not a single database setting

The most common mistake this topic invites: treating "our database" as consistent or eventually-consistent as one global fact. Real systems make this choice **per operation**, and the guarantee has to hold **end-to-end**, not just at the database:

- The same database can serve a linearizable read for a balance check (read from primary, or with a quorum read) and an eventually-consistent read for a recommendation feed (read from the nearest replica, accept staleness) — same storage engine, two different guarantees chosen deliberately per code path.
- A cache in front of a strongly-consistent database can silently turn the *effective* end-to-end guarantee into an eventually-consistent one (see [Caching §11](../Scaling%20and%20Performance/Caching.md#11-questions-to-expect-and-what-a-strong-answer-sounds-like "Questions to expect, and what a strong answer sounds like")) — worth stating explicitly that consistency analysis has to trace the full request path, not stop at "the database is strongly consistent."

---

## 6. Worked example: checkout inventory vs. product page views

A single e-commerce request flow illustrates why the right answer isn't one guarantee for "the app":

- **Product description/images on the page**: eventually consistent, served from a CDN/cache, is completely fine — a stale description for a few minutes has no real cost.
- **"Only 2 left in stock" badge**: read-your-writes is nice to have (so a user who just bought the last one doesn't get told it's still available) but slight staleness across *other* users is tolerable — this is a UX nicety, not a correctness requirement.
- **The actual checkout decrement**: needs linearizable, quorum-backed writes with real concurrency control — two concurrent checkouts must not both successfully reserve the last unit. This is usually enforced with a conditional/compare-and-swap write (`UPDATE inventory SET qty = qty - 1 WHERE sku = ? AND qty > 0`) at the primary, not eventual convergence.

One product, three different consistency requirements on three fields that happen to be rendered on the same page — this is the shape of answer that demonstrates real judgment rather than a single "we'll use eventual consistency for scale" blanket statement.

---

## 7. What separates senior from staff here

- **Senior**: states CAP correctly, knows strong vs. eventual consistency, can name a case for each.
- **Staff**: knows CAP is conditional on a partition and reaches for PACELC when discussing normal-operation tradeoffs; can do the quorum arithmetic (`W + R > N`) live and explain what a specific `(N, R, W)` choice actually buys or costs; distinguishes read-your-writes/monotonic-reads from blanket "eventual," because that's usually the guarantee the product actually needs; treats consistency as a per-operation, end-to-end decision (tracing through caches and replicas) rather than a single property of "the database."

---

## 8. Questions to expect, and what a strong answer sounds like

- **"Explain CAP theorem."** → State it precisely as a partition-time forced choice, then immediately pivot to PACELC as the more relevant everyday framing — this alone separates you from most candidates.
- **"Your replica read returns stale data — is that a CAP violation?"** → No — CAP is about partition-time behavior; this is ordinary replication lag during normal operation, a latency/consistency tradeoff (PACELC), not a partition.
- **"Design the consistency model for a ticket-booking checkout."** → Walk through [§6](#6-worked-example-checkout-inventory-vs-product-page-views "Worked example: checkout inventory vs. product page views")-style reasoning: browse/search pages can be eventually consistent, the actual seat reservation needs a linearizable conditional write (or a quorum write with W+R>N) to prevent double-booking.
- **"You have N=5 replicas — what W and R would you pick, and why?"** → Demonstrate the arithmetic: W=3, R=3 gives overlap (6>5) and tolerates 2 node failures on either side; W=1, R=1 is fast but offers no consistency guarantee and is really an eventually-consistent configuration.
- **"How do you handle two replicas that accepted conflicting concurrent writes?"** → Name LWW vs. vector clocks vs. application-level merge vs. CRDTs, and pick based on whether silently discarding one write is an acceptable product outcome.
