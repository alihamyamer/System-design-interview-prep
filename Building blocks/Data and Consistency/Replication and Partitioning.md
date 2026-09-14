# Replication and Partitioning

## Why this topic is a filter question

Everyone knows replication copies data and partitioning splits it. The filter is whether you can reason about the *mechanics* that make each one hard: how replication actually propagates writes (and what that implies for lag and failover), how a partition key choice ripples through every future query and resharding effort, and how the two techniques compose — because almost every real large-scale datastore uses both simultaneously, not one or the other.

---

## 1. Replication: why, and by what mechanism

Replication exists for four distinct reasons, worth naming separately because they lead to different design choices:
- **Durability** — losing a node shouldn't lose data.
- **Availability** — a node failing shouldn't take down reads (or writes, depending on the model).
- **Read scale** — spread read load across multiple copies.
- **Geographic latency** — put a copy near the reader.

### Primary-replica (the default mental model)

Writes go to one primary; replicas apply the same writes, typically by shipping the primary's write-ahead log (WAL) or a logical change stream. Two propagation modes matter operationally:

- **Synchronous replication**: the primary waits for (some number of) replicas to confirm before acknowledging the write to the client. Stronger durability guarantee (a replica has the data before the client is told it succeeded), at the cost of write latency bounded by the slowest replica you wait for.
- **Asynchronous replication**: the primary acknowledges immediately, replicas catch up afterward. Lower write latency, but a primary failure before a replica applies the write means **that write is lost** even though the client was told it succeeded — this is the concrete failure mode behind "replication lag," not an abstract inconvenience.

Most real systems use a hybrid: synchronous to one nearby replica (for durability on failover) and asynchronous to the rest (for read scale and geographic distribution) — naming this hybrid explicitly, rather than treating replication as binary sync-or-async, is a good signal.

### Failover — the part that's usually hand-waved

"Replicas take over if the primary fails" hides real complexity:
- **Detection**: how does the system know the primary is actually down versus just slow or partitioned from the coordinator (a false positive here causes a split-brain — see below)?
- **Promotion**: which replica becomes the new primary? Usually the one with the most up-to-date log position, but if replication was asynchronous, the most-caught-up replica can still be behind — meaning **failover can lose recently-acknowledged writes**, a fact worth stating rather than glossing over.
- **Split-brain**: if the old primary isn't actually dead — just network-partitioned from the health-check system — and a replica gets promoted, you can end up with two nodes both accepting writes as "the primary." Fencing mechanisms (the old primary must be confirmed stopped/unreachable before promotion completes, e.g., via a lease or STONITH-style mechanism) exist specifically to prevent this.

### Multi-leader and leaderless

- **Multi-leader**: more than one node accepts writes (common in multi-region setups where each region writes locally for latency). Buys write availability during a partition and lower write latency per region, at the direct cost of needing conflict resolution (see [Consistency and CAP §4](Consistency%20and%20CAP.md#4-conflict-resolution-when-replicas-diverge "Conflict resolution when replicas diverge")) since two leaders can accept conflicting writes to the same record concurrently.
- **Leaderless (Dynamo-style)**: any replica can accept a write; consistency is tuned via quorums (`N`, `R`, `W` — see [Consistency and CAP §3](Consistency%20and%20CAP.md#3-quorums-the-mechanism-behind-tunable-consistency 'Quorums — the mechanism behind "tunable consistency"')). Buys very high write availability, at the cost of the quorum/conflict-resolution machinery becoming a first-class part of the system rather than an edge case.

---

## 2. Partitioning: the key choice is the whole design

Partitioning (sharding) splits data across nodes so no single node holds — or has to process writes for — the entire dataset. The partition key decision is the highest-leverage decision in this space because **it determines which queries are cheap (single-partition) and which are expensive (cross-partition) for the rest of the system's life.**

### Partitioning strategies

- **Hash-based**: `partition = hash(key) % N`. Spreads load evenly if the key has good cardinality and low skew. Downside: range queries (`get all orders between two dates`) become a scatter-gather across every partition, since hashing destroys locality.
- **Range-based**: partitions own contiguous key ranges (e.g., partition 1 = users A–F). Great for range queries and ordered scans, but concentrates load if writes cluster at one end of the range — the classic failure is auto-incrementing IDs or timestamp-prefixed keys, where all *new* writes land on the *same* "latest" partition regardless of total partition count.
- **Directory-based**: a lookup service maps key → partition explicitly, rather than a formula. Most flexible (arbitrary rebalancing without a formula change), but the directory itself becomes a critical, must-scale, must-be-highly-available component.

### Choosing the key: work backward from access patterns, not forward from the schema

The question to actually ask is: **what's the dominant query, and does this key make that query single-partition?** A partition key that's "correct" in isolation (evenly distributed) can still be wrong if the dominant query pattern needs data that key doesn't co-locate. Concrete example: partitioning a multi-tenant SaaS system by `user_id` distributes load evenly, but if the dominant query is "give me all activity for tenant X" and users of the same tenant land on different partitions, every such query becomes a scatter-gather. Partitioning by `tenant_id` instead fixes that query at the cost of creating hot partitions for unusually large tenants — there usually isn't a key that's free on every axis; the job is picking which axis you're willing to pay on.

---

## 3. Hot partitions — beyond "a celebrity gets a lot of traffic"

The celebrity-account example is the standard illustration, but the general mechanism is: **any partitioning scheme has some possible key-value distribution that breaks it**, and the design question is how you detect and mitigate it rather than assume it away:

- **Detection**: per-partition metrics (QPS, latency, CPU) segmented by partition, not just fleet-wide averages — a fleet-wide p50 can look perfectly healthy while one partition is on fire.
- **Mitigation without repartitioning everything**:
  - **Key salting**: append a suffix/bucket number to a known-hot key (`hot_user_123#0` .. `hot_user_123#9`) to spread its writes across multiple partitions, at the cost of needing to fan out reads across those buckets when reading that key's full data.
  - **Splitting a single hot partition further** (if your system supports dynamic range splitting, e.g., HBase/Bigtable-style) without touching the rest of the cluster's layout.
  - **Caching in front of the hot partition** for read-heavy hot keys (see [Caching §10](../Scaling%20and%20Performance/Caching.md#10-what-separates-senior-from-staff-here "What separates senior from staff here")'s hot-key section) so the partition only has to absorb writes, not the full read volume too.

---

## 4. Resharding — the operation everyone underestimates

Growth eventually forces adding partitions, and that means **moving data that already lives somewhere** — this is qualitatively different and harder than adding a stateless server behind a load balancer, because you can't just spin up a new empty partition and start routing to it; existing keys' current locations have to change without breaking in-flight traffic.

- **Naive `hash % N` re-sharding is the trap**: changing `N` (partition count) changes the target partition for nearly every key simultaneously, forcing almost all data to move at once.
- **Consistent hashing** is the standard fix: keys and nodes are placed on a hash ring, and adding a node only remaps the keys between it and its neighbor on the ring — roughly `1/N` of keys move instead of nearly all of them. Worth being able to explain *why* it helps in exactly this terms, not just name-drop it.
- **Live migration mechanics**: dual-write or shadow-write to both old and new locations during migration, backfill historical data, verify, then cut over reads — this is the same "expand-contract" discipline that shows up in [REST §5](../Communication/REST.md#5-versioning-what-actually-breaks-clients "Versioning — what actually breaks clients") versioning and [RPC and gRPC §5](../Communication/RPC%20and%20gRPC.md#5-schema-evolution-where-strong-typing-cuts-both-ways "Schema evolution — where strong typing cuts both ways") schema evolution: change happens gradually and reversibly, not as a single atomic flag-day cutover, because a flag-day cutover on live data at scale is exactly the kind of operation that turns a routine migration into an incident.

---

## 5. Cross-partition operations — the cost partitioning doesn't advertise

Splitting data across partitions makes anything that spans partitions harder, and this is worth naming as a direct, permanent cost of the decision rather than an occasional inconvenience:

- **Cross-partition transactions**: maintaining atomicity across partitions typically needs a distributed transaction protocol (two-phase commit or a saga-style compensating-transaction pattern), both of which add real latency and failure-mode complexity compared to a single-node transaction.
- **Cross-partition queries/joins**: a query that needs data from multiple partitions becomes a scatter-gather — fan out to every relevant partition, merge results — which is slower and more failure-prone (now you depend on *every* partition responding, not just one) than a single-partition query.
- **Secondary indexes across partitions**: an index on a field other than the partition key either has to be maintained per-partition (and queries scatter-gather across all of them) or maintained as a separate globally-partitioned index structure (which itself needs to stay consistent with the primary data) — this is a real design decision, not a checkbox feature.

---

## 6. Replication and partitioning together

Almost every large-scale system combines both: data is **partitioned** across many nodes for write/storage scale, and **each partition is separately replicated** for durability and availability. This means the actual unit of failure/failover isn't "the database" — it's "one partition's primary," and a real design should describe both dimensions: "we partition by `tenant_id` into 128 shards for write scale, and each shard has a primary plus two replicas for durability and read scale" is a complete answer; naming only one dimension usually means the interviewer will ask about the other.

---

## 7. What separates senior from staff here

- **Senior**: knows primary-replica vs. multi-leader vs. leaderless, knows partitioning is for write/storage scale, can name a hot-partition example.
- **Staff**: reasons about failover concretely (detection, promotion, split-brain risk, and that async replication means failover can lose recent writes); picks a partition key by working backward from the dominant query pattern rather than only checking distribution evenness; knows why naive `hash % N` resharding is dangerous and explains consistent hashing's actual mechanism; treats cross-partition transactions/queries as a standing architectural cost that shapes the rest of the design, not an edge case; describes replication and partitioning as two independent, composed dimensions rather than picking one.

---

## 8. Questions to expect, and what a strong answer sounds like

- **"The primary dies — walk me through what happens next."** → Detection (health check timeout), promotion (most-caught-up replica, note that async replication means potential data loss here), fencing against split-brain if the old primary isn't confirmed dead.
- **"You need to add partitions because you're running out of write capacity — what's the actual process?"** → Not a `hash % N` change; consistent hashing or a directory-based remap so only a fraction of keys move; dual-write/backfill/cutover, not a flag-day migration.
- **"One tenant is 100x the size of a typical tenant on your `tenant_id`-partitioned system — what happens, and what do you do?"** → Hot partition; detect via per-partition metrics; mitigate via key salting or dedicated capacity for that tenant rather than repartitioning the whole cluster.
- **"How do you run a transaction that touches two shards?"** → Name the real options (2PC, sagas with compensation) and their cost, and ideally explain why you'd first try to avoid needing one by revisiting the partition key or denormalizing.
- **"Users complain they don't see their own update immediately after making it."** → Diagnose as replication lag if reads can land on a replica; connect to read-your-writes consistency (see [Consistency and CAP §2](Consistency%20and%20CAP.md#2-consistency-models-precisely-not-just-by-name "Consistency models, precisely, not just by name")) as the specific guarantee to add, e.g., routing that user's read to the primary or a replica known to be caught up.
