# Message Queues and Pub/Sub

## Why this topic is a filter question

Almost every candidate can say "we'll put it on a queue." What separates senior from staff is what happens in the next two minutes: can you explain *exactly* what breaks if a consumer crashes mid-message, why "exactly-once" is usually a lie, how you'd size partitions, and when messaging is actually the wrong call because it hides a correctness problem instead of solving it.

This doc is written around that bar. Skim the definitions section if you already have it; spend your time on delivery semantics, ordering, and the worked example — that's where interviews actually get won or lost.

---

## 1. The core mechanism

A **producer** writes a message to a **broker**. A **consumer** reads it later and does the work. The broker's job is to durably hold the message between those two events and hand it to the right consumer(s).

Two shapes of this:

- **Queue (point-to-point)**: each message is logically claimed and processed by *one* consumer (or one consumer per competing group). Good for distributing units of work.
- **Pub/sub (fan-out)**: each message is delivered to *every* independent subscriber. Good for broadcasting a fact that multiple, decoupled systems each need to react to.

The distinction that actually matters isn't the API shape, it's the **cardinality of interest**: does exactly one thing need to happen as a result of this message, or do many independent things need to happen? If you get this wrong, you either accidentally process the same work multiple times (queue used for something that needed fan-out) or you silently drop interested consumers (pub/sub topic without proper subscription tracking).

Most real brokers blur the line: Kafka topics can be consumed by multiple independent consumer groups (fan-out) while each group internally load-balances across partitions (queue-like). SNS+SQS is the AWS-native version of the same idea: SNS fans out, each subscribing SQS queue load-balances within itself.

---

## 2. Why you reach for asynchronous messaging at all

The honest reason isn't "it's a best practice" — it's a specific set of problems synchronous calls can't solve well:

| Problem | Why sync fails | What async buys you |
|---|---|---|
| Work is slower than the caller can wait | Caller times out or ties up a thread/connection | Caller returns immediately; work happens on its own schedule |
| Downstream is unreliable or occasionally down | Every caller failure cascades to the caller | Broker absorbs the outage; work resumes when downstream recovers |
| Traffic is bursty | You must provision downstream for peak load | The queue smooths bursts into a steady drain rate |
| One event, many interested parties | Caller has to know and call every consumer, and a new consumer requires caller changes | Producer stays ignorant of consumers; add a subscriber without touching the producer |
| A step needs to survive a crash | In-memory retry state is lost on crash | The message itself is the durable retry state |

The cost you're accepting in exchange: the caller no longer knows the outcome synchronously, you now have an extra piece of infrastructure to operate, and you've introduced eventual consistency between "the event happened" and "all consumers have reacted to it." A strong interview answer names this cost explicitly instead of treating messaging as free.

**When *not* to use it**: if the caller needs the result to render a response (e.g., "show me the calculated price"), or if you need a strong, immediate, read-your-writes guarantee, queuing just adds latency and a second failure mode without buying you anything. Staff-level candidates say this out loud rather than reflexively reaching for a queue.

---

## 3. Delivery semantics — the part interviewers actually probe

There are three theoretical guarantees. Only one of them is real in most systems.

**At-most-once**: the broker hands off the message and doesn't retry. If the consumer crashes before finishing, the message is gone. Cheap, but silently loses work. Rarely acceptable for anything with business consequence (fire-and-forget metrics/logging is the main legitimate use).

**At-least-once**: the broker keeps retrying until it gets an acknowledgment. This is the default posture of almost every real system (SQS, Kafka with manual offset commits, RabbitMQ with manual acks). The mechanism is usually: the consumer receives the message, it becomes invisible/locked for a **visibility timeout**, the consumer processes it and explicitly acks; if the ack doesn't arrive before the timeout, the message reappears for redelivery. That means **redelivery isn't a corner case — it's an expected, routine occurrence**, triggered by anything that makes the consumer slow (GC pause, a slow downstream call, a deploy that kills the pod mid-processing).

**Exactly-once**: not actually deliverable as a blanket property of "the message system." What you can get:
- *Exactly-once within a single system's transactional boundary* — Kafka's transactional/idempotent producer API guarantees a message is written to the log exactly once even under producer retries, and consumer offset commits can be tied to that same transaction. This solves duplication **at the broker layer**.
- What it does *not* solve: if your consumer's side effect is "call a payment API" or "write to a different database," you're back to at-least-once at that boundary, because the consumer can still crash between doing the side effect and committing the offset.

The correct staff-level framing: **design for at-least-once delivery and make the consumer idempotent.** Don't try to engineer exactly-once end-to-end — it's expensive, often impossible across system boundaries, and the idempotency work you'd need anyway makes it redundant.

### The dual-write problem (a favorite follow-up)

A very common bug pattern: a service writes a row to its database, *then* publishes an event about it. If the process crashes between the two, or the publish fails, the database and the event stream disagree — the event is lost even though the state change is durable. This is the **dual-write problem**, and interviewers who've operated real systems will ask about it directly.

Two standard fixes:
- **Transactional outbox**: write the business row and an "outbox" row describing the event *in the same database transaction*. A separate relay process (or CDC tool like Debezium) tails the outbox table and publishes to the broker, then marks it sent. This guarantees the event is published if and only if the transaction committed.
- **Change Data Capture (CDC)**: skip the outbox table and tail the database's replication/write-ahead log directly, turning row changes into events. More infrastructure, but no application code has to remember to write to an outbox.

Naming the dual-write problem unprompted is one of the highest-signal things you can do in this section of an interview — it shows you've actually operated a messaging system in production, not just read about one.

---

## 4. Ordering, partitioning, and scaling consumption

Global ordering across an entire topic is expensive and usually unnecessary — it requires a single writer and a single reader, which caps your throughput at one machine. Real systems instead offer **ordering within a partition** (Kafka) or **within a message group** (SQS FIFO's `MessageGroupId`), and no ordering guarantee across partitions.

This turns "does order matter?" into a **partition key design problem**:
- Pick a key that groups events that must stay ordered relative to each other (e.g., all events for a given `account_id`), and accept no ordering guarantee across different keys.
- A bad key choice creates **hot partitions**: if 30% of your traffic is one key (a celebrity account, a large enterprise customer), that partition's consumer becomes the throughput ceiling for the whole pipeline no matter how many partitions you add.
- Partition count is also your parallelism ceiling: a Kafka topic with 12 partitions can be consumed by at most 12 consumer instances in a group doing useful work simultaneously — the 13th instance sits idle. Repartitioning later is a real migration (existing per-partition ordering guarantees break across the rebalance), so this is a number worth reasoning about up front, not adjusting reactively.
- **Consumer group rebalancing**: when a consumer joins or leaves a group, partitions are reassigned. During a rebalance, consumption pauses for the affected partitions — this is a real, measurable latency spike under autoscaling or rolling deploys, and it's worth mentioning if the design involves frequent scale events.

A strong answer connects this back to the business requirement instead of stating it abstractly: "payment state transitions for a single order must be ordered, so I'll partition by `order_id`; I don't need ordering across different orders, so that's where I get my parallelism."

---

## 5. Backpressure

Async decoupling doesn't eliminate the limits of your downstream system — it just adds a buffer in front of them. If producers sustainedly outpace consumers, the queue grows without bound (cost, and eventually broker capacity limits or unbounded consumer lag), and the "smoothing" story quietly turns into "silent, growing latency."

Ways to actually handle this instead of hand-waving "the queue smooths spikes":
- **Consumer autoscaling** keyed on queue depth / consumer lag, not just CPU.
- **Load shedding or backpressure signals to the producer** — reject or degrade at the edge once queue depth crosses a threshold, rather than accepting unbounded backlog.
- **Rate limiting the producer side** for non-critical event types when the system is under strain.
- Monitoring **consumer lag** (Kafka) or **ApproximateAgeOfOldestMessage** (SQS) as a first-class SLO, not an afterthought — this is the metric that tells you whether the async boundary is actually keeping up.

---

## 6. Failure handling: retries, poison messages, and DLQs

Redelivery is the normal case (see §3), so retry policy is a first-class design decision, not error-handling boilerplate.

Questions a staff engineer answers explicitly:
- **Which failures are retryable?** A downstream 503 or timeout: retry. A malformed message that will *never* parse: retrying is pure waste — route it to a dead-letter queue (DLQ) immediately.
- **Backoff strategy**: fixed retries under load amplify an outage (thundering herd against a downstream that's already struggling). Exponential backoff with jitter is the standard answer, and jitter specifically matters because synchronized retries from many consumers otherwise re-create the spike they're backing off from.
- **DLQ as an operational surface, not a graveyard**: messages there need alerting, a runbook, and either automatic or manual reprocessing — a DLQ nobody looks at just delays data loss instead of preventing it.
- **Poison messages**: a message that crashes the consumer every time (bad schema, unexpected null) will loop through retries indefinitely without a max-retry-count → DLQ policy, potentially head-of-line-blocking everything behind it on an ordered queue/partition.

---

## 7. Idempotency in practice

Since at-least-once is the realistic default, the consumer — not the broker — is where correctness actually gets enforced.

Patterns, roughly in order of how commonly they show up:
- **Idempotency key**: the message carries a unique ID (often generated by the producer at the point the underlying action was decided, e.g., a client-generated request ID for "charge this order"). The consumer checks a dedup store before acting.
- **Dedup table / unique constraint**: `INSERT ... ON CONFLICT DO NOTHING` on `(idempotency_key)`, or a dedicated `processed_message_ids` table with a TTL long enough to outlast the broker's maximum redelivery window. This is usually cheaper and more reliable than trying to build dedup logic into the business tables directly.
- **Natural idempotency**: design the operation so replays are harmless by construction — `SET status = 'shipped'` is naturally idempotent; `balance += amount` is not. Preferring naturally idempotent operations avoids needing a dedup store at all, and it's worth explicitly considering before reaching for one.
- **Exactly-once producer + transactional offset commit** (Kafka-specific): solves duplication *within* the broker but, per §3, doesn't extend to external side effects.

---

## 8. Technology comparison

Don't memorize this as trivia — use it to justify a choice with the actual property that mattered for the requirement, e.g. "I need replay for a new consumer joining later, so I want a log, not a queue that deletes on ack."

| System | Model | Ordering | Replay | Fan-out | Operational profile |
|---|---|---|---|---|---|
| **Kafka** | Durable, partitioned append-only log | Per-partition | Yes — consumers control their own offset, can rewind | Native (multiple independent consumer groups read the same log) | High throughput, but you own cluster operation (or pay for a managed tier); higher setup complexity |
| **RabbitMQ** | Message broker with flexible routing (exchanges/bindings) | Per-queue | No (messages removed on ack, unless explicitly configured otherwise) | Via exchange fan-out to multiple queues | Rich routing patterns (topic, direct, fanout exchanges); simpler mental model than Kafka for pure task queues |
| **Amazon SQS** | Managed queue | FIFO queues: per-`MessageGroupId`; standard queues: best-effort | No | Not native — pair with SNS for fan-out | Fully managed, minimal ops; standard queues are effectively unlimited throughput, FIFO caps throughput per message group |
| **Amazon SNS** | Managed pub/sub | None | No | Native, typically fans out to SQS/Lambda/HTTP | Fully managed; pairs with SQS so each subscriber gets its own durable, independently-consumed buffer |
| **Google Pub/Sub** | Managed pub/sub with durable subscriptions | Best-effort (ordering keys available, with a throughput tradeoff) | Limited (subscription seek) | Native | Fully managed, global by default |
| **Redis Streams / Pub-Sub** | In-memory (Streams add persistence to disk via RDB/AOF) | Per-stream | Streams: yes, bounded by retention; Pub/Sub: no, fire-and-forget only | Yes | Lowest latency, but durability is weaker than a purpose-built broker — treat Redis pub/sub (not Streams) as at-most-once |

The one-line version an interviewer wants to hear: **Kafka when you need replay and high sustained throughput; SQS/RabbitMQ-style queues when you need simple, reliable work distribution; SNS/pub-sub when many independent consumers need the same event and you don't want the producer to know about them.**

---

## 9. Worked example: order processing pipeline

Putting the above together, since interviews rarely stop at "define pub/sub":

**Requirement**: when an order is placed, billing must charge the customer, inventory must reserve stock, and notifications must email a confirmation — and a new "fraud scoring" consumer needs to be added six months from now without changing the order service.

**Design decisions and why:**
1. **Pub/sub over a queue**: three-plus independent consumers need the same fact → fan-out, not point-to-point. Use an `order_created` topic (Kafka) or SNS topic fanning into per-consumer SQS queues.
2. **Transactional outbox** on the order service: the order row and the `order_created` outbox row commit in one transaction, so a crash right after order creation can't silently drop the event (§3).
3. **Partition key = `order_id`**: no cross-order ordering requirement, so this maximizes parallelism while keeping any per-order event sequence (e.g., `order_created` → `order_cancelled`) ordered.
4. **At-least-once + idempotent consumers**: billing keys its charge attempt on `order_id` as an idempotency key against the payment provider (most providers, e.g. Stripe, support this natively) so a redelivered event can't double-charge. Inventory's reservation is naturally idempotent (`SET reserved = true WHERE order_id = ...`).
5. **DLQ per consumer, not shared**: if notifications' email provider is down, that shouldn't block billing or inventory — each subscriber gets its own queue/DLQ so failures are isolated (this is also why SNS→SQS or independent Kafka consumer groups beat a single shared queue for this fan-out).
6. **Adding fraud scoring later** costs zero changes to the order service: it's a new consumer group subscribing to the existing topic. This is the concrete payoff of decoupling via pub/sub that's worth stating explicitly when asked "why not just have the order service call each downstream directly."

---

## 10. What actually separates senior from staff here

- **Senior**: correctly identifies queue vs. pub/sub, knows the common delivery guarantees exist, can name Kafka/SQS/RabbitMQ and roughly what they're for.
- **Staff**: proactively raises the dual-write problem before being asked; reasons about partition key choice in terms of a specific hot-key risk in *this* system; treats "at-least-once + idempotent consumer" as the default design stance rather than a fallback; can say when messaging is the wrong tool; ties DLQ and backpressure into operational ownership (alerting, runbooks, consumer lag as an SLO) rather than treating them as edge-case footnotes.

---

## 11. Questions to expect, and what a strong answer sounds like

- **"What happens if the consumer crashes right after processing but before acknowledging?"** → Expect redelivery; the design must already assume this (visibility timeout expires, message reappears, idempotent consumer handles the duplicate safely).
- **"How would you guarantee the event is never lost even if the service crashes right after the DB write?"** → Transactional outbox or CDC; explain why publish-after-commit without one is unsafe.
- **"One customer is 10x the traffic of everyone else — what happens to your partitioning?"** → Name the hot-partition problem and a mitigation (sub-partitioning the hot key, e.g. `order_id + hash bucket`, or accepting reduced parallelism for that key as a conscious tradeoff).
- **"Your DLQ has 50,000 messages sitting in it from three days ago — what does that tell you, and what do you do?"** → Something upstream broke systematically, not transiently (schema change, a bad deploy) — this needs alerting to have paged someone at message #500, not #50,000; walk through triage (sample messages, identify the common failure, decide replay vs. discard).
- **"Why not just call the three downstream services synchronously from the order service?"** → Slowest downstream sets your p99 latency and its outage becomes your outage; adding a fourth consumer means changing and redeploying the order service; you lose the ability to replay history for a new consumer. Say the cost too: added infra, eventual consistency between order creation and downstream reaction becoming visible.
