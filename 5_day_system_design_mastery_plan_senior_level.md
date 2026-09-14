# 5-Day System Design Mastery Plan (Senior / Staff Level)

## Objective

This plan assumes you already know the fundamentals — it isn't where you learn what a load balancer is. It's a schedule for converting the depth in this repo's reference docs into something you can actually produce live, under time pressure, while being interrupted and pushed on your assumptions. The reference docs (linked throughout) hold the *why*; this plan holds the *how to rehearse it* and *how to grade yourself*.

What actually separates senior from staff in these interviews, stated once so it doesn't need repeating in every section below: a senior candidate can describe the right components and name the right tradeoffs when asked. A staff candidate volunteers the tradeoffs *before* being asked, quantifies the decision with a rough number instead of an adjective, and can be pushed two levels deeper on any part of the design without running out of real content — because they've actually operated something like it, or have thought about it as if they had.

---

## The interview framework — with an actual time budget

Every design answer should move through these phases. The time budget below is for a standard 45-minute round; adjust proportionally for 30 or 60.

1. **Requirements (5 min)** — functional scope (what the system does) and non-functional scope (scale, latency, consistency needs). Do not skip asking about scale — "how many users, how many requests per second, read-heavy or write-heavy" changes almost every downstream decision, and asking it unprompted is itself a signal.
2. **Capacity estimation (3–5 min)** — see below. Skipping this is one of the most common ways an otherwise-good candidate reads as mid-level rather than senior.
3. **High-level design (8–10 min)** — major components and data flow, at the level of a box diagram. Resist the pull to go deep on any one box yet.
4. **Data model / API (5–8 min)** — schema or access patterns, and the core endpoints/contracts. This is where [ApiDesign](ApiDesign.md), [RelationDBDesign](RelationDBDesign.md), and [noSqlDesign](noSqlDesign.md) get applied.
5. **Deep dives (15–20 min)** — this is where the interview is actually won or lost, and where the interviewer steers. Scaling, failure handling, consistency tradeoffs, concurrency correctness. Everything in this repo's Building Blocks docs is deep-dive material.
6. **Wrap-up (2–3 min)** — proactively name what you'd do with more time, and any assumption you made that you'd want to validate with the product team.

### Capacity estimation, done for real, not skipped

This step is small in time but disproportionate in signal — it's the fastest way to demonstrate you think quantitatively instead of just architecturally. The mechanical version, worked for a concrete example (a URL shortener, since the arithmetic is simple enough to do live):

- **Traffic**: 100M new short URLs/month → 100,000,000 / (30 × 24 × 3600) ≈ 40 writes/sec average. If reads are 100x writes (typical for a shortener), that's ~4,000 reads/sec average — and you should explicitly ask or state whether traffic is roughly uniform or peaky, since peak provisioning is usually 3–10x the average, not the average itself.
- **Storage**: 100M URLs/month × 12 months × ~500 bytes/record (short code + long URL + metadata) ≈ 600GB/year — small enough that this system is not a storage-scale problem, which is itself a useful thing to say out loud (it tells the interviewer where NOT to spend design effort).
- **Bandwidth**: reads/sec × average response size gives you a rough network throughput number, worth a sentence if the system has a large payload (video, images) rather than a redirect.

The point isn't precision — it's demonstrating that a design decision ("do we need to shard," "is a single Postgres primary enough," "do we need a CDN") is being driven by an actual number instead of a reflex. Getting the arithmetic slightly wrong out loud is fine; not attempting it at all is the actual mistake.

---

## Day 1 — API and Data Modeling

**Goal**: translate requirements into clean, defensible APIs and schemas, and pick the right storage category deliberately.

**Study**: [ApiDesign](ApiDesign.md) in full — particularly [§2](ApiDesign.md#2-authentication-vs-authorization-and-where-each-actually-gets-enforced "Authentication vs. authorization — and where each actually gets enforced") (auth model choice), [§3](ApiDesign.md#3-multi-tenancy-and-isolation-assume-an-adversarial-client-not-just-a-well-behaved-one "Multi-tenancy and isolation — assume an adversarial client, not just a well-behaved one") (multi-tenancy/isolation), [§4](ApiDesign.md#4-idempotency-the-mechanism-and-where-it-actually-gets-designed-wrong "Idempotency — the mechanism, and where it actually gets designed wrong") (idempotency mechanics), [§7](ApiDesign.md#7-rate-limiting-the-algorithm-and-the-distributed-state-problem "Rate limiting — the algorithm and the distributed-state problem") (rate limiting algorithms). [RelationDBDesign §2](RelationDBDesign.md#2-keys-the-choice-with-the-longest-lasting-consequences "Keys — the choice with the longest-lasting consequences") (key choice), [§5](RelationDBDesign.md#5-transactions-isolation-and-the-anomaly-that-actually-bites-people "Transactions, isolation, and the anomaly that actually bites people") (write skew), [§8](RelationDBDesign.md#8-worked-example-preventing-overselling-and-double-spending-completely "Worked example: preventing overselling and double-spending, completely") (worked concurrency+idempotency example). [noSqlDesign §1](noSqlDesign.md#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities") and [§4](noSqlDesign.md#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one "Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one") (access-pattern-first design and single-table design — this is the single highest-leverage NoSQL topic to actually practice, not just read).

**Practice — design the schema and core API for each of these, out loud, timed at 15 minutes each:**
- A chat system (get [RelationDBDesign §8](RelationDBDesign.md#8-worked-example-preventing-overselling-and-double-spending-completely "Worked example: preventing overselling and double-spending, completely")'s concurrency+idempotency pattern into your reflexes, since message delivery has the same double-send risk as the inventory example)
- A notification system (forces the polling-vs-webhook-vs-push decision from [ApiDesign §6](ApiDesign.md#6-async-apis-polling-vs-webhooks-and-the-idempotency-problem-this-creates-for-the-client "Async APIs — polling vs. webhooks, and the idempotency problem this creates for the *client*") and the real-time transport docs)
- A rate limiter (forces the token-bucket-vs-sliding-window and distributed-shared-state decision from [ApiDesign §7](ApiDesign.md#7-rate-limiting-the-algorithm-and-the-distributed-state-problem "Rate limiting — the algorithm and the distributed-state problem"))

**Self-check**: did you enumerate access patterns *before* proposing a schema ([noSqlDesign §1](noSqlDesign.md#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities"))? If you designed tables first and checked queries against them afterward, you defaulted to the relational instinct even where it wasn't asked for — worth noticing which mode you reach for under pressure.

---

## Day 2 — Concurrency, Idempotency, and Ordering

**Goal**: handle the edge cases that separate "the happy path works" from "this survives real, concurrent, retrying traffic."

**Study**: [RelationDBDesign §5](RelationDBDesign.md#5-transactions-isolation-and-the-anomaly-that-actually-bites-people "Transactions, isolation, and the anomaly that actually bites people") (write skew — know this cold, it's the anomaly most candidates can't name), [§6](RelationDBDesign.md#6-concurrency-control-patterns-matched-to-their-actual-failure-mode "Concurrency control patterns, matched to their actual failure mode") (concurrency patterns matched to their failure mode). [REST §2–3](Building%20blocks/Communication/REST.md#2-idempotency-keys-how-they-actually-work "Idempotency keys through partial updates and concurrency control") (idempotency key mechanics and the concurrent-request race, optimistic concurrency via `ETag`/`If-Match`). [Message Queues and PubSub §3–4](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#3-delivery-semantics-the-part-interviewers-actually-probe "Delivery semantics through ordering, partitioning, and scaling consumption") (the dual-write problem, ordering via partition keys) and [§7](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#7-idempotency-in-practice "Idempotency in practice") (idempotent consumer patterns).

**Practice — reason through each of these out loud:**
- Two admins `PATCH` the same record within the same second — walk through last-write-wins vs. optimistic locking vs. field merge, and justify your pick.
- A payment webhook is delivered twice — design the receiver so this is provably safe, not "probably fine."
- Reordering items in a list (a todo list, a kanban column) under concurrent drag-and-drop — compare gap-based integer positions, fractional/lexicographic keys, and a linked-list model, and state which one you'd actually ship and why (this is a real, common follow-up precisely because every option has a genuine, statable downside — no answer is free).

**Self-check**: for every "handle retries safely" claim, can you say specifically *what store* holds the dedup state and *what makes the check atomic* (a unique constraint, not just an in-memory check)? If not, you have the concept but not the mechanism — go re-read [REST §2](Building%20blocks/Communication/REST.md#2-idempotency-keys-how-they-actually-work "Idempotency keys — how they actually work").

---

## Day 3 — Storage, Scaling, and Caching

**Goal**: design systems that degrade gracefully under load instead of falling over, and can actually reason about *why* a given scaling lever works.

**Study**: [Caching §4–5](Building%20blocks/Scaling%20and%20Performance/Caching.md#4-the-cache-stampede-and-the-difference-between-the-two-related-but-distinct-problems-people-conflate "The cache stampede through the cache-aside concurrency race") (stampede vs. cold-cache-warmup as distinct problems, and the cache-aside race condition), [§9](Building%20blocks/Scaling%20and%20Performance/Caching.md#9-worked-example-same-entity-different-guarantees-per-field "Worked example: same entity, different guarantees per field") (per-field caching decisions, not per-entity). [Replication and Partitioning §1–4](Building%20blocks/Data%20and%20Consistency/Replication%20and%20Partitioning.md#1-replication-why-and-by-what-mechanism "Replication through resharding") (failover mechanics, partition key selection, consistent hashing and why naive resharding is dangerous). [Load Balancing §3–4](Building%20blocks/Scaling%20and%20Performance/Load%20Balancing.md#3-routing-strategies-and-where-the-obvious-choice-breaks 'Routing strategies through health checks') (routing strategy vs. workload shape, health check depth tradeoff).

**Practice:**
- "Sharding by `user_id` improves locality but risks celebrity hot spots" — don't stop at that sentence. Name the actual mitigation (key salting, sub-partitioning) and the cost it introduces (fan-out reads).
- Design the cache layer for a product page, field by field, using the [Caching §9](Building%20blocks/Scaling%20and%20Performance/Caching.md#9-worked-example-same-entity-different-guarantees-per-field "Worked example: same entity, different guarantees per field") pattern — different TTL/consistency per field, not one blanket cache strategy.
- Walk through what happens, step by step, when a popular cache key expires under sustained load — this should produce the stampede mechanism and a named fix (request coalescing, jittered TTL), not just the word "stampede."

**Self-check**: can you state, for your design, which specific number (per-partition throughput ceiling, cache hit rate, connection count per instance) would actually tell you the design is starting to fail — not just "it doesn't scale"?

---

## Day 4 — Messaging and Async Systems

**Goal**: design event-driven systems where delivery semantics, ordering, and failure handling are explicit decisions, not assumptions.

**Study**: [Message Queues and PubSub](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md) in full — [§3](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#3-delivery-semantics-the-part-interviewers-actually-probe "Delivery semantics — the part interviewers actually probe") (delivery semantics and the dual-write problem), [§4](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#4-ordering-partitioning-and-scaling-consumption "Ordering, partitioning, and scaling consumption") (ordering/partitioning/rebalancing), [§6](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#6-failure-handling-retries-poison-messages-and-dlqs "Failure handling: retries, poison messages, and DLQs") (retries, poison messages, DLQ as an operational surface), [§9](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#9-worked-example-order-processing-pipeline "Worked example: order processing pipeline") (the worked order-processing example, which is the template for how to answer "design an event-driven system" end to end).

**Practice:**
- Design the notification fan-out from Day 1 as a pub/sub system this time — name why fan-out (not a queue) fits, and design the transactional outbox so the triggering write and the event can't disagree.
- "Your DLQ has 50,000 messages from three days ago" — walk through triage, not just "we'd look into it."
- Compare Kafka, SQS, and SNS for a specific stated requirement (replay needed vs. not; ordering needed vs. not) rather than reciting their feature lists.

**Self-check**: for any "at-least-once, idempotent consumer" claim, can you name the specific idempotency mechanism for *that* consumer's side effect (naturally idempotent write vs. a dedup table vs. a provider-native idempotency key)? Generic idempotency talk without a mechanism is the most common shallow answer in this section.

---

## Day 5 — Full Mock Interviews

**Goal**: put all four days together under real time pressure, including being interrupted and redirected.

**Practice systems** (45 minutes each, ideally with someone else interrupting you the way an interviewer would): a chat system, a news feed, a rate limiter, a file upload system, a search system. For each, run the full framework above — requirements, capacity estimate, high-level design, data model, then let the deep dive go wherever the "interviewer" pushes, rather than a topic you pre-selected.

### Self-grading rubric

After each mock, grade yourself honestly against these, since this is the checklist an interviewer is implicitly using:

| Signal | Weak version | Strong version |
|---|---|---|
| Scale | Never asked, or asked and ignored | Drove at least one concrete design decision (see [Capacity estimation](#capacity-estimation-done-for-real-not-skipped) above) |
| Tradeoffs | Named after being asked "what are the downsides" | Volunteered unprompted, with the specific cost stated (not just "it's more complex") |
| Failure modes | "We'd add monitoring" | A specific failure walked through step by step (a node dies, a cache stampedes, a retry storm) with the actual mechanism and mitigation |
| Depth on push | Repeats the same point with different words when pushed | Goes to a genuinely new layer (mechanism → operational cost → how you'd detect it in production) |
| Structure | Jumps to implementation details before requirements are settled | Follows the phase order, but adapts pacing live rather than rigidly |

**Weak signals to watch for in your own answers**: jumping into implementation before requirements are settled; stating a tradeoff exists without stating what it actually costs; overengineering (introducing Kafka, a graph database, or a service mesh for a problem that doesn't need one — and not being able to say why you'd hold off).

**Strong signals to aim for**: proactive tradeoffs, quantified decisions, concrete failure walkthroughs, and comfort saying "I'd need to check the real numbers, but here's how I'd reason about it" instead of bluffing precision you don't have.

---

## Rapid-fire topic index

One line each, pointing to where the real answer lives — use this as a pre-interview refresher, not a substitute for having read the docs:

- **Idempotency mechanics and the concurrent-retry race** → [REST §2](Building%20blocks/Communication/REST.md#2-idempotency-keys-how-they-actually-work "Idempotency keys — how they actually work")
- **Write skew (what repeatable read doesn't catch)** → [RelationDBDesign §5](RelationDBDesign.md#5-transactions-isolation-and-the-anomaly-that-actually-bites-people "Transactions, isolation, and the anomaly that actually bites people")
- **Cursor vs. offset pagination, precisely why** → [REST §4](Building%20blocks/Communication/REST.md#4-pagination-where-the-just-use-cursors-answer-needs-to-go-deeper 'Pagination — where the "just use cursors" answer needs to go deeper')
- **Cache stampede vs. cold-cache warmup** → [Caching §4](Building%20blocks/Scaling%20and%20Performance/Caching.md#4-the-cache-stampede-and-the-difference-between-the-two-related-but-distinct-problems-people-conflate "The cache stampede — and the difference between the two related-but-distinct problems people conflate")
- **Hot partitions, with real throughput numbers** → [noSqlDesign §3](noSqlDesign.md#3-partition-keys-and-the-numbers-that-make-a-bad-key-concrete 'Partition keys, and the numbers that make a "bad key" concrete')
- **Eventual consistency, precisely defined, plus PACELC** → [Consistency and CAP §1–2](Building%20blocks/Data%20and%20Consistency/Consistency%20and%20CAP.md#1-what-cap-actually-says-precisely "What CAP actually says, plus consistency models")
- **WebSockets vs. SSE vs. long polling — the actual decision** → [Websockets §2](Building%20blocks/Communication/Websockets.md#2-the-actual-decision-does-the-client-need-to-speak-frequently-on-the-same-low-latency-channel "The actual decision: does the client need to speak, frequently, on the same low-latency channel?")
- **Rate limiting, and why it breaks across a fleet** → [ApiDesign §7](ApiDesign.md#7-rate-limiting-the-algorithm-and-the-distributed-state-problem "Rate limiting — the algorithm and the distributed-state problem")
- **The dual-write problem and transactional outbox** → [Message Queues and PubSub §3](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#3-delivery-semantics-the-part-interviewers-actually-probe "Delivery semantics — the part interviewers actually probe")
- **Single-table design** → [noSqlDesign §4](noSqlDesign.md#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one "Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one")

---

## Outcome

By the end of five days, you should be able to lead a system design discussion without needing to recall facts — because the reasoning (why this tradeoff, what breaks, what number would tell you) is now something you produce live, not retrieve. If a section above still feels like recall rather than reasoning, that's the signal to spend another pass on that doc before your interview, not a sign to add more topics.
