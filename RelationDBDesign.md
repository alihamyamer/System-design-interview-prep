# Relational Database Design

## Why this topic is a filter question

Everyone can explain normalization and name a B-tree index. The filter is whether you understand the mechanisms underneath the vocabulary: what "repeatable read" actually fails to prevent (write skew, specifically), why a random UUID primary key quietly degrades write performance at scale, what a schema migration actually locks and for how long, and how you'd design money-movement logic that's correct under concurrent access and safe to retry. Those are the questions that separate "has read about databases" from "has operated one under real load."

---

## 1. When relational is the right default, precisely

Relational is the right default when the domain has real structure, correctness constraints the database itself should enforce (not just the application), and query patterns that aren't fully known upfront — the ability to `JOIN` and add an index later, without re-modeling the whole dataset, is the actual value being bought. It stops being the right default when write throughput needs to exceed what one write-serializing primary can sustain even after standard scaling tools (see [§11](#11-replication-backup-and-the-two-numbers-that-define-disaster-recovery "Replication, backup, and the two numbers that define disaster recovery")), or when the access pattern is fixed, simple, and key-based enough that a purpose-built store's tradeoffs (see [noSqlDesign](noSqlDesign.md)) pay off. Neither of those is common in a typical product's first few years — reaching for a distributed NoSQL store before you've actually measured a relational bottleneck is a common overcorrection, and naming that explicitly is a good staff-level instinct.

---

## 2. Keys — the choice with the longest-lasting consequences

Primary key choice affects indexing behavior, replication, and sharding strategy for the table's entire lifetime, so it's worth reasoning about precisely rather than picking UUID by default:

- **Auto-increment integer**: small, cache-friendly, and — critically — **monotonically increasing**, which means every insert appends to the end of the B-tree's key range rather than landing at a random position. This locality is a real performance property, not just a size convenience.
- **Random UUID (v4)**: globally unique with no coordination needed across distributed writers, but random values scatter inserts across the *entire* key space of the index — every insert can touch a different, possibly-cold B-tree page, causing more page splits, worse buffer-cache hit rates, and larger index write amplification than a monotonic key at high insert volume. This is a genuine, measurable production cost, not a theoretical one.
- **ULID / UUIDv7 (time-ordered UUIDs)**: globally unique *and* roughly monotonic (a timestamp prefix, random suffix) — gets distributed-generation safety without the random-scatter cost. This is why most systems that need distributed ID generation today reach for a time-ordered UUID variant rather than UUIDv4, and it's worth knowing why, not just that it's "trendier."
- **Natural keys** (email, external account number): tempting, but the meaning "this value uniquely and permanently identifies the row" rarely survives contact with reality (emails change, account numbers get reissued) — treat them as unique constraints on a surrogate-keyed table, not as the primary key itself.

Also worth knowing precisely: **whether your engine clusters the table by primary key.** MySQL/InnoDB physically orders row storage by primary key (a clustered index) — so PK choice directly determines the physical layout and range-scan locality of the *whole row*, not just the index. PostgreSQL stores rows in an unordered heap with indexes as separate structures pointing into it — PK monotonicity still matters for the *index's* write pattern, but doesn't determine row physical layout the same way. This is a genuine cross-engine difference worth having ready if asked "does this apply the same way in Postgres and MySQL."

---

## 3. Indexing — beyond "add an index," to what it costs and doesn't help

The mechanics (B-tree for equality/range/sort, GIN for full-text/JSONB/arrays, the left-prefix rule for composite indexes) are standard; the higher-signal points:

- **Every index is a second data structure that every write has to update** — an insert/update doesn't just write the row, it writes every index that covers a changed column. A table with eight indexes for read convenience can have meaningfully worse write latency than the same table with two well-chosen ones — indexing is a read/write tradeoff, not a free read optimization.
- **A covering index can eliminate the row lookup entirely** (an "index-only scan" in Postgres terms) if every column the query needs is in the index itself — worth naming as the difference between "the index found the right rows" and "the index answered the query without touching the table at all," which is a materially faster path for hot, narrow queries.
- **`EXPLAIN`/query plan literacy is the actual skill, not index trivia.** Knowing index *types* by name is table stakes; being able to say "this query plan shows a sequential scan on a 40M-row table because the filter column isn't indexed, or because the planner decided the index wouldn't be selective enough — here's how I'd verify which" is the staff-level version of the same knowledge.

---

## 4. Normalization vs. denormalization — as a measured decision, not a default

Start normalized; it's the safer default because duplicated data drifts out of sync exactly when you're not looking. Denormalize when you have **specific, measured** evidence a read path needs it — a join that `EXPLAIN` shows is scanning millions of rows on a latency-sensitive endpoint, not a guess that "joins are probably slow." When you do denormalize, the design question that actually matters is **how the duplicate stays in sync**: on every write (harder to get wrong, adds write latency — akin to cache write-through), via an async job/event (eventual consistency window — same shape as [cache invalidation §3](Building%20blocks/Scaling%20and%20Performance/Caching.md#3-invalidation-the-actually-hard-part-in-mechanism-not-just-in-name "Invalidation — the actually-hard part, in mechanism, not just in name")'s event-driven approach), or accepted as a periodically-refreshed materialized view (bounded staleness by design, simplest to reason about). Naming which sync mechanism you're choosing, and what staleness window it implies, is what turns "I'd denormalize this" into a real answer.

---

## 5. Transactions, isolation, and the anomaly that actually bites people

ACID and the standard isolation levels (read uncommitted/committed, repeatable read, serializable) are table stakes to define. The part worth going deeper on is **write skew**, because it's the anomaly that survives "repeatable read" and is genuinely counterintuitive:

> Two doctors are both on-call. The on-call policy requires at least one doctor on call at all times. Doctor A checks "is anyone else on call besides me?" — sees Doctor B, decides it's safe to go off call. Concurrently, Doctor B runs the identical check, sees Doctor A, and also goes off call. Both transactions individually read consistent, correct data and both individually satisfy the invariant *as they observed it* — but the two transactions never saw each other's write, and the result violates the invariant: zero doctors on call.

This passes under **repeatable read / snapshot isolation** (the level Postgres actually gives you when you ask for "repeatable read" — each transaction sees a consistent snapshot, so neither read is "dirty" or "non-repeatable" in the classic sense) precisely because the anomaly isn't about one row being read inconsistently — it's about two transactions each reading a *different, disjoint* set of rows and each writing based on that read, with no overlap for a lock to catch. Only **serializable** isolation (or an explicit application-level constraint — e.g., a `CHECK` enforced via a covering exclusion constraint, or re-validating the invariant at commit time) prevents it. This is worth being able to describe unprompted if asked "what could go wrong with repeatable read" — it's the single most common real gap between what people assume isolation levels guarantee and what they actually guarantee.

**Operational note specific to Postgres**: a long-running transaction holds back vacuum's ability to reclaim dead tuples (it needs to keep old row versions visible for as long as any transaction's snapshot might still need them), which causes table/index bloat under sustained write load — a "why is this table 5x its logical size and getting slower" incident is very often traced back to one long-lived transaction (a stuck migration, a debugging session left open, a poorly-bounded batch job) rather than a data volume problem. This is a good concrete example of transactions having an operational cost beyond lock contention.

---

## 6. Concurrency control patterns, matched to their actual failure mode

- **Row-level locking / `SELECT ... FOR UPDATE`**: the right tool when you need to read-then-write a specific row and can't tolerate another writer sneaking in between — the classic "reserve the last unit of inventory" pattern ([§8](#8-worked-example-preventing-overselling-and-double-spending-completely "Worked example: preventing overselling and double-spending, completely")). Cost: lock contention under high concurrency on the same row/hot key.
- **Optimistic locking (version column / `WHERE version = ?`)**: better when conflicts are expected to be rare — no lock held during the "think" time between read and write, just a check-and-reject at commit, with the caller retrying on conflict. Worse under high contention (many wasted retries), better when contention is genuinely occasional.
- **Idempotency keys**: solve a *different* problem than the other two — not "two different actors racing," but "the same logical request arriving more than once" (a client retry, a network blip). See [REST §2](Building%20blocks/Communication/REST.md#2-idempotency-keys-how-they-actually-work "Idempotency keys — how they actually work") for the full mechanism; the relational-specific implementation detail is a unique constraint on the idempotency key column enforced at the database, so a concurrent duplicate insert fails atomically rather than depending on an application-level check-then-insert race.

A well-designed money-movement or inventory path typically needs **both** a concurrency-control mechanism (for concurrent competing writers) **and** idempotency (for retries of the same writer) — they're solving different problems and one doesn't substitute for the other.

---

## 7. Schema migrations — what actually locks, and for how long

Beyond the additive-first, backfill-then-enforce sequence (which is table stakes — see also the same expand-contract discipline in [REST §5](Building%20blocks/Communication/REST.md#5-versioning-what-actually-breaks-clients "Versioning — what actually breaks clients") and [RPC and gRPC §5](Building%20blocks/Communication/RPC%20and%20gRPC.md#5-schema-evolution-where-strong-typing-cuts-both-ways "Schema evolution — where strong typing cuts both ways")), the detail that separates real operational experience from a memorized checklist is **which specific operations lock the table, and for how long**:

- **Adding a nullable column with no default**: cheap in modern Postgres/MySQL — effectively metadata-only.
- **Adding a column with a non-null default**: historically required rewriting every row (a full table lock for the duration on large tables); Postgres 11+ made this metadata-only for a *constant* default, but MySQL's behavior here is version/storage-engine dependent — worth explicitly saying "I'd verify this on the specific engine and version," which is truer and more senior than confidently asserting a universal rule.
- **Adding a `NOT NULL` constraint** or a `CHECK` constraint on an existing large table can require a full table scan to validate — Postgres lets you add it as `NOT VALID` first (no scan, immediately enforced for new writes) and validate the existing data separately with `VALIDATE CONSTRAINT`, which takes a much lighter lock than validating inline. Knowing this two-step exists is a genuinely strong signal.
- **Adding an index**: use `CREATE INDEX CONCURRENTLY` (Postgres) or an online DDL tool (`pt-online-schema-change`, `gh-ost` for MySQL) on any table large enough that a blocking index build would cause a real outage window — plain `CREATE INDEX` takes a lock that blocks writes for the build's duration, which is fine on a small table and a real incident on a large one.

The general principle underneath all of these: **assume the table is too large to lock for more than a moment, and design the migration around that assumption from the start**, rather than writing the "obvious" DDL and discovering the lock duration in production.

---

## 8. Worked example: preventing overselling and double-spending, completely

This shows up constantly as a follow-up, and a complete answer combines [§6](#6-concurrency-control-patterns-matched-to-their-actual-failure-mode "Concurrency control patterns, matched to their actual failure mode")'s two mechanisms explicitly:

```sql
BEGIN;

-- Row-level lock: no other transaction can concurrently modify this row
-- until this transaction commits or rolls back.
SELECT quantity FROM inventory WHERE sku = 'sku_123' FOR UPDATE;

-- Application checks quantity > 0 here, using the locked, current value.

UPDATE inventory SET quantity = quantity - 1 WHERE sku = 'sku_123';

INSERT INTO reservations (id, sku, order_id)
VALUES ('idempotency-key-abc', 'sku_123', 'order_456')
ON CONFLICT (id) DO NOTHING;  -- idempotency: a retried request is a no-op, not a second decrement

COMMIT;
```

- `FOR UPDATE` closes the concurrent-writer race (two checkouts can't both see `quantity = 1` and both succeed).
- The unique constraint on `reservations.id` (the idempotency key) closes the retry race (the same logical request, retried, doesn't decrement twice) — and `ON CONFLICT DO NOTHING` makes the retry a safe no-op instead of an error the caller has to special-case.
- For actual money movement (not just a counter), the equivalent pattern is a **double-entry ledger**: every movement writes two balanced rows (a debit and a credit) inside one transaction, rather than mutating a single `balance` column — this makes the invariant ("debits always equal credits") independently auditable and checkable after the fact, which a single mutable balance column can't offer, and it's the detail that signals you've thought about how a financial system gets *audited*, not just how it avoids a race.

---

## 9. Scaling tools, in the order you should actually reach for them

Vertical scaling → read replicas (for read-heavy load, accepting replication lag — see [Replication and Partitioning §1](Building%20blocks/Data%20and%20Consistency/Replication%20and%20Partitioning.md#1-replication-why-and-by-what-mechanism "Replication: why, and by what mechanism") for the failover and lag mechanics) → table partitioning (splits one logical table into physical partitions on the *same* database, mainly to make maintenance/retention/scans manageable, e.g. monthly range partitions on an events table with an old-partition-drop retention policy) → caching (offload repeated reads entirely — [Caching](Building%20blocks/Scaling%20and%20Performance/Caching.md)) → sharding (splits data across independent database instances, the most operationally expensive step, since cross-shard joins/transactions become genuinely hard — see [Replication and Partitioning §2–5](Building%20blocks/Data%20and%20Consistency/Replication%20and%20Partitioning.md#2-partitioning-the-key-choice-is-the-whole-design "Partitioning through cross-partition operations") for the partition-key and resharding mechanics, which apply directly here). The staff-level framing: **sharding is what you reach for after the cheaper tools are exhausted and you have a natural partition key already implied by your access pattern (usually `tenant_id`)** — not a default scaling posture, and not something to design around speculatively before there's a measured need.

---

## 10. Multi-tenancy at the schema level

Shared schema with a `tenant_id` column (simplest to operate centrally, requires airtight query discipline — see [ApiDesign §3](ApiDesign.md#3-multi-tenancy-and-isolation-assume-an-adversarial-client-not-just-a-well-behaved-one "Multi-tenancy and isolation — assume an adversarial client, not just a well-behaved one") for the enforcement mechanisms, including row-level security as a database-enforced backstop against a missing `WHERE` clause) vs. schema-per-tenant (better logical separation, schema migrations now have to run against every tenant schema, which gets operationally heavy past some tenant count) vs. database-per-tenant (strongest isolation and easiest per-customer compliance story, highest cost and operational overhead). Most systems start shared-schema and move a specific large or compliance-sensitive tenant to its own database only when there's a concrete reason (contractual isolation requirement, or that tenant's load pattern is disruptive to others) — not as a blanket policy from day one.

---

## 11. Replication, backup, and the two numbers that define disaster recovery

Beyond naming that replicas exist: the two concrete numbers a real design commits to are **RPO** (recovery point objective — how much data can we afford to lose, i.e., how far back does our last durable copy go) and **RTO** (recovery time objective — how long can we be down while recovering). These directly determine the backup/replication strategy: async replication + periodic backups implies a nonzero RPO (recent writes since the last replica sync or backup can be lost on a hard failure); synchronous replication to at least one standby shrinks RPO toward zero at the cost of write latency. A strong answer states these two numbers for the specific system being designed, rather than listing backup types generically.

---

## 12. What separates senior from staff here

- **Senior**: models entities/relationships correctly, knows normalization vs. denormalization, can name isolation levels and index types, knows `FOR UPDATE` and idempotency keys exist.
- **Staff**: explains write skew as the specific anomaly repeatable-read doesn't prevent, and when serializable or an explicit invariant check is actually needed; reasons about primary key choice through its effect on index/page locality (random UUID vs. time-ordered ID), not just uniqueness; knows which specific migration operations lock a table and for how long, and reaches for the safe variant (`NOT VALID` + `VALIDATE`, `CONCURRENTLY`, online DDL tools) by default on large tables; combines concurrency control and idempotency as two distinct mechanisms solving two distinct problems rather than treating them as interchangeable; and states RPO/RTO as concrete numbers rather than describing backup strategy in the abstract.

---

## 13. Questions to expect, and what a strong answer sounds like

- **"Your app uses repeatable read and you still got a data integrity violation under concurrent access — how?"** → Write skew: two transactions each read a disjoint set of rows and each write based on that read, with no overlapping lock to catch the conflict; needs serializable isolation or an explicit invariant check, not just a stronger read guarantee.
- **"Why would a random UUID primary key hurt performance at scale, if it's just as unique as an integer?"** → Random values scatter inserts across the entire B-tree key range instead of appending, causing more page splits and worse cache locality — propose a time-ordered UUID (ULID/UUIDv7) instead.
- **"You need to add a `NOT NULL` column to a 200-million-row table with zero downtime — what's the actual sequence?"** → Add nullable, backfill in batches, add the constraint `NOT VALID` then `VALIDATE CONSTRAINT` separately (Postgres) rather than one blocking `ALTER TABLE`, or use an online DDL tool on engines without that two-step.
- **"How do you prevent overselling the last item in stock, including if the client retries the request?"** → Two distinct mechanisms: `SELECT ... FOR UPDATE` (or optimistic version check) for the concurrent-writer race, plus an idempotency key with a unique constraint for the retry race — walk through both, and why one doesn't substitute for the other.
- **"How would you design a money-transfer table so an auditor could verify correctness after the fact?"** → Double-entry ledger (paired debit/credit rows per movement, in one transaction) instead of a single mutable balance column, so the "debits equal credits" invariant is independently checkable.
