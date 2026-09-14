# NoSQL Database Design

## Why this topic is a filter question

Naming key-value/document/wide-column/graph and giving one example of each is table stakes. The filter is whether you can actually *design* in the access-pattern-first mindset these systems require — which, done well, culminates in single-table design ([§4](#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one "Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one")), the single highest-signal NoSQL topic in a staff interview and the one most candidates have only heard of, not practiced. The rest of the filter is knowing precisely what you give up (joins, ad hoc queries, sometimes transactions) and being honest about when that trade isn't worth it.

---

## 1. The mindset shift: design backward from queries, not forward from entities

Relational modeling starts with "what are the entities and relationships" and lets query flexibility follow from that — indexes and joins are added later, against whatever the access pattern turns out to be. Most NoSQL systems (especially the ones with genuinely low, predictable per-request latency at scale — DynamoDB, Cassandra) invert this: **you enumerate every access pattern the application needs *before* designing the schema**, because the schema is going to be shaped specifically to make each of those patterns a single, local operation (one partition, one key lookup), and patterns you didn't design for can become expensive or outright unsupported later.

Concretely, before writing a single table definition, you should have a numbered list like:
1. Get a user's profile by user ID.
2. Get all orders for a user, most recent first.
3. Get an order and all its line items in one read.
4. Get all orders in a given status, across all users (for an ops dashboard).

Pattern 4 is the one that breaks a naive single-key design — it's a query that doesn't align with any single partition — and catching that *before* committing to a partition key is the entire point of doing this exercise upfront rather than discovering it in production.

---

## 2. The categories, compressed to what actually differs

| Category | Optimized for | Give up | Reach for it when |
|---|---|---|---|
| **Key-value** (Redis, DynamoDB used simply) | Lowest possible latency on an exact-key lookup | Any query the app didn't already know the key for | The app always knows the key, and speed matters more than flexibility |
| **Document** (MongoDB, Firestore) | Storing one naturally nested object as one read/write unit | Cross-document joins, some schema discipline unless enforced at the app layer | Records are naturally hierarchical and mostly read/written whole |
| **Wide-column** (Cassandra, Bigtable, ScyllaDB) | Very high, horizontally-scaled write throughput with query-driven table design | Ad hoc queries; cross-partition queries are expensive-to-unsupported | Write volume is extreme and access is genuinely partition-key-shaped (telemetry, activity streams) |
| **Graph** (Neo4j, Neptune) | Multi-hop relationship traversal | Most of the tooling/ergonomics of the above for non-traversal queries | The core value of the product *is* the traversal (fraud rings, recommendation paths, permission inheritance) |
| **Search** (Elasticsearch, OpenSearch) | Full-text relevance ranking, faceting | Transactional correctness — it's a derived index, not a source of truth | Users search by free text/ranking rather than exact keys |
| **Time-series** (TimescaleDB, ClickHouse, InfluxDB) | Append-heavy writes, time-windowed aggregation, downsampling/retention | General-purpose flexible querying | The dominant access dimension is genuinely time |

The common failure mode this table is meant to prevent: picking a category because of its reputation ("Mongo is flexible," "Cassandra scales") rather than because the *specific* access-pattern list from [§1](#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities") actually matches what that category is good at.

---

## 3. Partition keys, and the numbers that make a "bad key" concrete

A partition key routes data to a physical location and — this is the part worth stating precisely rather than just "avoid hot keys" — **every partitioned system has a real, finite per-partition throughput ceiling**, independent of how big the overall cluster is. DynamoDB, for a concrete number: a single partition supports up to 3,000 read capacity units or 1,000 write capacity units per second (numbers evolve, but the *shape* — a hard per-partition ceiling regardless of total provisioned capacity — is the durable fact). That means "partition by `country`" isn't just "probably uneven" — if one country legitimately drives more than that ceiling's worth of traffic, requests to that partition throttle *no matter how much total capacity the table has*, because the other partitions' spare capacity is architecturally unreachable from a hot single key.

Mitigations, concretely:
- **Composite/synthetic keys**: `country#bucket` where `bucket` is a hash of user ID or a rotating suffix — spreads one logical hot entity across N physical partitions, at the cost of needing to fan out reads across those N buckets when you need the full picture for that country.
- **Write sharding with a known fan-in**: same idea, explicitly sized to the entity's known scale (a large customer's data pre-split into a fixed number of buckets from day one, rather than reactively split after an incident).
- **Re-checking the key choice against the *actual* traffic distribution**, not the theoretical one — this is why the access-pattern exercise in [§1](#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities") should include rough traffic estimates per pattern, not just the pattern shape.

---

## 4. Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one

This is worth its own section because it's the concrete embodiment of [§1](#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities")'s mindset, and it's the part interviewers most want to see you actually reason through rather than recite.

**The idea**: instead of one table per entity type (users, orders, order_items — the relational instinct), store multiple related entity types in **one table**, using a generic partition key (`pk`) and sort key (`sk`) whose *meaning* is overloaded per item type, chosen so that every access pattern from your list becomes a single query against a contiguous key range.

Worked example — an order system with the four access patterns from [§1](#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities"):

| pk | sk | Item type / attributes |
|---|---|---|
| `USER#123` | `PROFILE` | User profile attributes |
| `USER#123` | `ORDER#2026-04-01#o1` | Order summary (denormalized: total, status) |
| `USER#123` | `ORDER#2026-04-15#o2` | Order summary |
| `ORDER#o2` | `ITEM#sku_1` | Line item |
| `ORDER#o2` | `ITEM#sku_2` | Line item |

How each access pattern resolves:
- **Pattern 1** (get profile): `pk = USER#123, sk = PROFILE` — a direct key lookup.
- **Pattern 2** (get a user's orders, most recent first): `pk = USER#123, sk begins_with ORDER#` — one query, and because the sort key is prefixed with an ISO timestamp, results come back already in chronological order for free (this is why timestamp-prefixed sort keys are the standard trick for "recent N" access patterns).
- **Pattern 3** (get an order and its line items): `pk = ORDER#o2` — one query returns the order and every line item under it, because they share a partition key.
- **Pattern 4** (all orders in a given status, across users): **this is the pattern that doesn't fit the base table** — it cuts across every user's partition. This is exactly the case a **Global Secondary Index (GSI)** exists for: project `status` and a subset of attributes into a separate index with its own partition key (e.g., `status` as the GSI's `pk`), so pattern 4 becomes a query against the *index* instead of a scan of the base table. The cost, worth naming explicitly: a GSI is a second, asynchronously-maintained copy of (a projection of) your data, consuming its own write capacity and storage, and it's eventually consistent with the base table by default — a status-change write and a GSI-based read of that status can briefly disagree.

The reason this is worth practicing, not just knowing exists: it inverts nearly every relational instinct (one table per entity, normalize, add indexes reactively) into its opposite (one table for related entities, denormalize deliberately, decide indexes *before* writing data) — and interviewers use it specifically to check whether you can actually think in this model or only describe it from the outside.

---

## 5. Denormalization and repair — the standing cost of the tradeoff [§4](#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one "Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one") makes

Storing an order summary denormalized under `USER#123` ([§4](#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one "Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one")'s table) means the order's total or status now exists in more than one place if it's also referenced elsewhere (e.g., a GSI projection, or a separate analytics record) — every design that denormalizes needs an explicit answer to **who updates the copies, and what happens when a copy update fails partway through**:

- **Own the update path explicitly**: a single write path (application code or a stream-triggered function) responsible for updating every copy, ideally inside one atomic operation where the database supports it (DynamoDB transactions, MongoDB multi-document transactions within a single replica set) — but see [§6](#6-transactions-and-correctness-vary-a-lot-and-nosql-is-not-one-answer 'Transactions and correctness — vary a lot, and "NoSQL" is not one answer') for why you should still prefer avoiding the need for this over relying on it.
- **Idempotent reconciliation jobs**: a background job that periodically re-derives the denormalized copies from the source of truth and corrects drift — the "repair" half of eventual consistency that's often mentioned in passing but rarely designed concretely. A strong answer names what triggers it (schedule? a detected mismatch?), how it detects drift (checksums, version numbers, a last-updated comparison), and that it must be safe to run repeatedly without side effects (idempotent), since a repair job that isn't idempotent is just a second source of bugs.
- **Event-driven propagation** (a change stream — DynamoDB Streams, MongoDB Change Streams, Cassandra CDC — triggering downstream updates): decouples "the source of truth changed" from "every copy is updated," same shape as the outbox/CDC pattern in [Message Queues and PubSub §3](Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#3-delivery-semantics-the-part-interviewers-actually-probe "Delivery semantics — the part interviewers actually probe"), and worth naming as the same underlying pattern rather than a NoSQL-specific trick.

---

## 6. Transactions and correctness — vary a lot, and "NoSQL" is not one answer

"NoSQL doesn't have transactions" is wrong and dated; the accurate statement is that **guarantees vary widely by system and usually come with a real cost that pushes you toward avoiding the need for them**:

- DynamoDB supports transactional writes across multiple items, but they cost roughly double the write capacity and are explicitly positioned as an escape hatch — the recommended default is still modeling so the *common* write path is a single-item (or single-partition) operation, and reserving transactions for the genuinely rare multi-item invariant.
- MongoDB supports multi-document transactions, but leaning on them heavily is a signal the document model (one object, one read/write unit) has been fought against rather than used — if most of your writes need a cross-document transaction, that's often evidence the data should be one document, or that you've picked the wrong tool.
- Cassandra is explicitly modeled to avoid needing cross-partition transactional guarantees at all — the query-driven table design philosophy ([§1](#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities")) is partly *in service of* this: if every important write is naturally single-partition, you never needed a distributed transaction to begin with.

The pattern across all three: **the tool isn't "add transactions," it's "model the write path so you rarely need one," and reach for the transactional API as the exception, not the default.**

---

## 7. Consistency — apply the general framework, don't re-derive it here

The concepts (strong vs. eventual, read-your-writes, quorum reads/writes) are covered in full in [Consistency and CAP](Building%20blocks/Data%20and%20Consistency/Consistency%20and%20CAP.md) — that doc's quorum arithmetic (`W + R > N`) and PACELC framing apply directly to Cassandra/DynamoDB-style systems, since they're exactly the systems that made those concepts mainstream. The NoSQL-specific point worth adding: **many of these systems make you choose consistency *per operation*, at call time** (e.g., DynamoDB's `ConsistentRead` flag on a `GetItem` call, Cassandra's per-query consistency level) rather than as a single database-wide setting — which means the design conversation should be "which specific reads need strong consistency" ([§6 in the Consistency doc](Building%20blocks/Data%20and%20Consistency/Consistency%20and%20CAP.md#6-worked-example-checkout-inventory-vs-product-page-views "Worked example: checkout inventory vs. product page views")'s worked example is the right shape of answer), not "is this database strongly or eventually consistent," because the honest answer is usually "both, depending which call you're making."

---

## 8. Schema evolution in schema-flexible systems

Schema flexibility is a capability, not an excuse to skip discipline — the actual risks and mitigations:

- **Field-naming drift** (`userId` in one service's writes, `user_id` in another's) turns querying and indexing into a mess; enforce a shared schema/validation layer (JSON Schema validation, a shared struct/type in a common library) even though the database itself won't stop you from writing whatever shape you want.
- **Versioned documents**: include a `schemaVersion` field, and handle multiple versions in application read code (a "migrate-on-read" pattern: old-shape documents get transformed to the new shape at read time, and optionally rewritten on next write) rather than requiring a synchronized big-bang migration across every existing document — this is the schema-flexible-store equivalent of relational's additive-first migration discipline ([RelationDBDesign §7](RelationDBDesign.md#7-schema-migrations-what-actually-locks-and-for-how-long "Schema migrations — what actually locks, and for how long")), just enforced in application code instead of DDL.
- **Large document growth** is a specific, concrete MongoDB gotcha worth knowing: a document that grows past its originally allocated storage space has to be *moved* on disk (historically a real performance cost, mitigated but not eliminated by modern storage engines) — a data model that appends unbounded arrays to a single document (e.g., "all of a user's activity, ever, in one document") can degrade over time in a way that isn't obvious from the data model alone; capping array size or moving high-growth data to its own document/collection is the standard fix.

---

## 9. Secondary indexes — what they cost, per system

- **DynamoDB GSIs**: a separate, asynchronously-maintained projection with its own partition key and its own provisioned capacity — real additional write cost and storage, and eventually consistent by default ([§4](#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one "Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one")). Local Secondary Indexes (LSIs) share the base table's partition key (same partition, different sort key) and can be strongly consistent, but must be created at table creation time and are capped in number — a real constraint worth knowing exists even if the exact cap isn't memorized.
- **Cassandra**: secondary indexes exist but are explicitly discouraged for most access patterns in favor of designing a purpose-built table per query (a materialized view or a second table you write to alongside the first) — the query-driven design philosophy again.
- **MongoDB**: secondary indexes work much like relational ones (B-tree-based, support the same left-prefix compound-index reasoning as [RelationDBDesign §3](RelationDBDesign.md#3-indexing-beyond-add-an-index-to-what-it-costs-and-doesnt-help "Indexing — beyond add-an-index, to what it costs")) — closer to relational indexing than DynamoDB/Cassandra's more constrained models, which is part of why MongoDB reads as the "easiest to get started with" NoSQL option for teams coming from a relational background.

---

## 10. When NOT to use NoSQL — including the mid-project version of this question

The standard answer ("relational is better when you need joins, strong multi-row transactions, unpredictable reporting queries") is correct but easy to state abstractly. The version worth being ready for is the **mid-project** one: a team picks a document store early because the schema was genuinely fluid in the prototype phase, ships, and eighteen months later the product has grown real cross-entity relationships and ad hoc internal reporting needs the original access-pattern list never anticipated. Recognizing this as a plausible trajectory — not a hypothetical failure of "bad NoSQL choice," but a normal way requirements evolve — and naming the mitigation (a search/analytics layer fed by CDC for the reporting need, rather than forcing joins into the primary store; or accepting a genuine, costly migration if the core access pattern itself has fundamentally changed) is a more mature answer than treating the initial choice as simply right or wrong in hindsight.

---

## 11. What separates senior from staff here

- **Senior**: names the NoSQL categories correctly, knows partition keys should be chosen for even distribution, knows eventual consistency exists and picks a database per category's reputation.
- **Staff**: designs backward from an explicit, numbered access-pattern list before touching the schema; can actually execute single-table design with GSI overloading for the one pattern that doesn't fit the base table, not just describe it; reasons about hot partitions using real per-partition throughput ceilings, not just "spread load evenly"; treats denormalization as requiring a named repair/propagation mechanism, not a one-time convenience; knows the transactional-API cost in each major system well enough to explain why you model around needing it rather than reaching for it by default; and recognizes that "wrong database choice" is often really "requirements outgrew an appropriate early choice," with a mitigation path rather than a verdict.

---

## 12. Questions to expect, and what a strong answer sounds like

- **"Design the DynamoDB schema for this order system."** → Start from the numbered access-pattern list ([§1](#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities "The mindset shift: design backward from queries, not forward from entities")), then walk through single-table design with a GSI for the one cross-cutting pattern ([§4](#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one "Single-table design — the technique that separates a real DynamoDB/Cassandra answer from a surface-level one")) — this is the answer shape interviewers are fishing for, not a generic "DynamoDB is good for scale."
- **"One customer is far larger than the rest and their partition is throttling — why, exactly, and what do you do?"** → Name the real per-partition throughput ceiling (not just "hot partition, bad"), and propose a composite/synthetic key to spread that customer across multiple physical partitions.
- **"You denormalized order totals into three places — a write fails halfway through updating them. What happens?"** → Name the specific mechanism you'd use (transactional write where cheap enough, or event-driven propagation plus an idempotent reconciliation job) rather than saying "eventual consistency will fix it."
- **"Why not just use MongoDB transactions everywhere and stop worrying about document boundaries?"** → Explain that heavy reliance on cross-document transactions is usually a sign the document model is being fought rather than used, and that the better fix is often reshaping which data lives in which document.
- **"Eighteen months in, product now needs ad hoc reporting your NoSQL access patterns never anticipated — what do you do?"** → Don't force joins into the primary store; add a search/analytics layer fed by change-data-capture, and treat a full migration as the last resort, reserved for when the *core* access pattern (not just reporting) has genuinely changed.
