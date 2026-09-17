# Q1 — Design a Live Customer Support Platform

A synthesis question. It's built specifically so that every "Building blocks" concept in this repo has a *real, non-forced* reason to show up: [Load Balancing](../Building%20blocks/Scaling%20and%20Performance/Load%20Balancing.md), [Caching](../Building%20blocks/Scaling%20and%20Performance/Caching.md), [Long Polling](../Building%20blocks/Communication/Long%20Polling.md), [Websockets](../Building%20blocks/Communication/Websockets.md), [RelationDBDesign](../RelationDBDesign.md), [noSqlDesign](../noSqlDesign.md), and [Message Queues and PubSub](../Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md). If you can solve this cleanly and *justify each choice from the requirement that forces it* — not recite it because it's expected — you're solving it the right way.

---

## The prompt (as an interviewer would give it to you)

> Design the backend for an embeddable customer support / live chat platform — think Intercom or Zendesk Chat. Other companies drop a `<script>` widget on their own websites. A visitor on `some-random-company.com` can open the widget, wait in a queue, and get connected to a live support agent for a real-time text chat. Support agents work from a separate web dashboard where they handle multiple concurrent chats, see ticket history, and use canned responses.

---

## Functional requirements

1. A visitor opens the embedded widget on a third-party site and joins a support queue for that company.
2. While waiting, the visitor sees their live queue position and an estimated wait time.
3. Once an agent is available, the visitor is connected and both sides exchange real-time text messages, including "agent is typing…" indicators.
4. Agents work from a dashboard: they see their active chats, can pull up a visitor's full conversation history, and use canned response templates.
5. Every conversation becomes a **ticket** with a status (`open`, `assigned`, `resolved`) and is tracked against an SLA (e.g., "first response within 2 minutes").
6. When a ticket is created, assigned, or resolved, the customer's own systems (e.g., their CRM or a Slack channel) can optionally receive a webhook.
7. Admins need reporting: message volume per company, SLA breach rate, agent load over time.

## Non-functional requirements / scale

| Dimension | Number |
|---|---|
| Companies embedding the widget | 50,000 |
| Concurrent website visitors with the widget open (across all companies) | ~2,000,000 at peak |
| Concurrent *active* chats (visitor matched to an agent) | ~20,000 at peak |
| Online agents at peak | ~5,000 |
| Chat messages/sec at peak, system-wide | ~5,000 |
| Target first-message latency once matched | < 300 ms |
| SLA-breach detection latency | < 30 s of the actual breach |
| Widget must work on | Arbitrary third-party sites — unknown corporate networks, proxies, ad blockers, and CSP policies you don't control |

---

## Clarifying questions worth asking (and the answers, so you can just start designing)

- **"Does the visitor need to keep the widget open the whole time, or can they leave and come back?"** → They can close the tab and reopen later; the conversation must resume where it left off (this is what forces a durable transcript store, not just an in-memory chat buffer).
- **"Is queue position update frequency high or low?"** → Low — a visitor's position changes maybe once every 15–30s as the queue drains, not multiple times a second.
- **"Do visitors ever need to send a message *before* being matched to an agent?"** → No — pre-match, it's read-only status (position, wait estimate). This matters for picking a transport.
- **"Can one agent handle multiple chats at once?"** → Yes, typically 3–6 concurrently, which affects how you size the agent-side real-time connection.
- **"What happens if the customer's webhook endpoint is slow or down?"** → It must never block or delay the visitor/agent experience — treat it as a best-effort, retryable, isolated side effect.
- **"Do we need full-text search over old transcripts?"** → Yes, for admins/agents pulling up history, but it's explicitly a secondary, less latency-sensitive access pattern.

---

# Reference solution

## 1. High-level shape

```
Visitor (widget, 3rd-party site)          Agent (dashboard, your own site)
        |                                          |
   [Global LB / CDN]                          [Global LB / CDN]
        |                                          |
   [L7 LB] ----------------+---------------- [L7 LB]
        |                  |                       |
  Long-poll queue API   WebSocket gateway (chat)    REST API (tickets, history, search)
        |                  |                       |
        +--------[ Redis: queue state, agent availability cache ]
        |                  |
        +------------[ Kafka: ticket_events, message_events ]-------------+
                           |                    |                |        |
                     Analytics consumer   SLA monitor   Webhook dispatcher  Search indexer (Elasticsearch)
                                                                            |
   [PostgreSQL: companies, agents, tickets, SLA]   [DynamoDB/Cassandra: chat transcripts]
```

Two separate real-time transports, two separate databases, one event backbone tying them together — each for a specific, defensible reason, worked through below.

---

## 2. Communication: why long polling *and* WebSockets, not one or the other

This system has two genuinely different real-time needs, and picking one transport for both would be wrong in both directions.

### Pre-match queue status → Long polling

The visitor hasn't been matched yet. What they need is: "what's my position now, and has an agent become available?" That's a **low-frequency**, **one-way**, **read-mostly** need — but it runs inside a widget embedded on 50,000 *other people's* websites, over networks and corporate proxies you don't control and can't assume are WebSocket-friendly.

This is exactly the scenario [Long Polling §6](../Building%20blocks/Communication/Long%20Polling.md#6-long-polling-vs-the-alternatives-the-actual-decision) calls out as the *honest* reason to reach for it in 2026: not "it's simpler," but "we need this to work through infrastructure known to kill long-lived streaming connections, for a client population we don't control." A support widget on an arbitrary company's site, sitting behind an unknown enterprise proxy, is precisely that case.

Design specifics:
- Client holds a request open against `GET /queue/status`; server responds when position changes or after a ~20s timeout.
- Each response carries a monotonic `event_id`. The next request says "give me anything newer than X," so a client that reconnects mid-gap (flaky wifi, laptop sleep) self-heals instead of silently missing the "an agent is now available" event — the exact fix described in [Long Polling §3](../Building%20blocks/Communication/Long%20Polling.md#3-the-gap-on-reconnect-problem).
- Because each request is independent, this tier needs **no sticky sessions** — any instance can serve any request, as long as queue state lives in Redis, not server-local memory. This is the one place in the whole design where statelessness is nearly free.
- **Capacity math worth stating out loud**: 2,000,000 concurrent visitors × up to a ~20s held request is a sustained held-connection/memory problem, not a requests/sec problem (per [Long Polling §2](../Building%20blocks/Communication/Long%20Polling.md#2-the-real-cost-held-request-capacity-not-bandwidth)) — this tier is sized by concurrency headroom, and it's the largest connection count in the whole system by two orders of magnitude, which is worth flagging even though each individual connection is cheap and short-lived relative to a chat session.

### Live chat → WebSockets

Once matched, both the visitor *and* the agent need to **send**, frequently, on the same low-latency channel — messages, typing indicators, read receipts. Per [Websockets §2](../Building%20blocks/Communication/Websockets.md#2-the-actual-decision-does-the-client-need-to-speak-frequently-on-the-same-low-latency-channel), this is the one condition that actually justifies WebSockets over a one-way push mechanism: this is not "push a notification," it's a live bidirectional conversation.

Design specifics:
- Handshake carries a short-lived auth token; on connect, the client sends a `join` for its `conversation_id`, and the gateway sets up a broker subscription on the visitor's/agent's behalf (per [Websockets §4](../Building%20blocks/Communication/Websockets.md#4-connection-lifecycle-and-heartbeats)).
- A WebSocket gives you a byte channel, nothing more ([Websockets §3](../Building%20blocks/Communication/Websockets.md#3-what-a-websocket-does-not-give-you)) — delivery, ordering across reconnects, and dedup are built on top: every message gets a server-assigned sequence number within its conversation, the client sends `Last-Seen-Seq` on reconnect, and the server replays from the durable transcript store (§5 below), not from memory.
- 15–30s ping/pong heartbeats detect a dead connection (laptop sleep, network drop without a clean close) — without this, the gateway silently holds resources for connections that are actually gone, and "agent is typing…" / online-status features drift out of sync with reality.
- The gateway is kept **stateless**: per-connection routing state lives in the broker subscription (backed by the same event backbone as §6), not gateway-local memory, so any gateway instance can serve any client and a rolling deploy doesn't require pinning.
- Agents connect once and multiplex several concurrent conversations (3–6) over that single socket, tagged by `conversation_id` per message — this keeps agent-side connection count an order of magnitude smaller than visitor-side.

### Why not one transport for both

Using WebSockets for the pre-match queue would mean holding ~2,000,000 persistent, mostly-idle connections purely to deliver an event that fires once every 15–30s per visitor — expensive for no latency benefit, and it fails outright on networks that kill long-lived streams, which is exactly the population this widget has to support. Using long polling for live chat would mean a full request/response cycle (auth check, routing, possibly a new TCP setup) *per message*, which falls over at chat frequency per [Long Polling §2](../Building%20blocks/Communication/Long%20Polling.md#2-the-real-cost-held-request-capacity-not-bandwidth)'s churn argument, and gives the visitor no way to push a message the moment they type it. Naming this contrast — not just using both because the assignment said to — is the actual point of this section.

---

## 3. Load balancing

- **Global/edge**: the widget's JS bundle and static assets are CDN-cached (long TTL, versioned URLs); API traffic uses GeoDNS to route a visitor to the nearest region, accepting the DNS-TTL-bounded failover delay described in [Load Balancing §5](../Building%20blocks/Scaling%20and%20Performance/Load%20Balancing.md#5-global-load-balancing-and-failover).
- **L7, path-based routing** in front of three independently scaled tiers, since they have completely different load shapes: the long-poll queue-status tier (huge connection count, trivial CPU/request), the WebSocket chat gateway (moderate connection count, needs connection draining on deploy), and the REST API (normal request/response, scales on requests/sec like any stateless HTTP tier).
- **The WebSocket gateway specifically** cannot use a naive least-connections signal — per [Load Balancing §6](../Building%20blocks/Scaling%20and%20Performance/Load%20Balancing.md#6-long-lived-connections-break-the-request-rate-mental-model), an idle agent-side socket holding 5 quiet conversations looks identical to one handling 5 very active ones by raw connection count. Route new connections by a resource-aware signal (active message rate or current conversation count per instance) instead, and use explicit **connection draining** during deploys — stop accepting new connections on an instance, let existing ones finish or force-close after a grace window — rather than the "just stop routing new requests" story that's sufficient for the stateless REST tier.
- **Health checks**: the REST/long-poll tiers use deep readiness checks (can this instance reach Postgres, Redis, Kafka), while being aware — per [Load Balancing §4](../Building%20blocks/Scaling%20and%20Performance/Load%20Balancing.md#4-health-checks-the-part-that-quietly-determines-your-real-availability) — that if every instance's readiness check depends on the same downstream (e.g., Redis), a Redis blip can flip the entire fleet unhealthy simultaneously, turning a partial degradation into a total outage. Worth explicitly deciding which dependencies are "must be healthy" vs. "degrade gracefully without."

---

## 4. Caching — per field, not per entity

Following the product-page pattern in [Caching §9](../Building%20blocks/Scaling%20and%20Performance/Caching.md#9-worked-example-same-entity-different-guarantees-per-field), this system has at least four caching decisions that are genuinely different:

| What | Strategy | Why |
|---|---|---|
| **Agent availability / current load** (used for routing a new visitor to an agent) | Cache-aside in Redis, short TTL **and** event-driven invalidation on agent status change | Staleness here directly causes a real bug — routing a visitor to an agent who just went offline — so this is the field that can least tolerate a long TTL |
| **Canned response templates** | App-layer cache, long TTL (minutes–hours) | Changes rarely, read constantly by every agent; staleness cost is essentially zero |
| **Visitor/customer profile** (name, plan tier, prior ticket count — shown in the widget/dashboard for context) | Read-through cache, moderate TTL | Read far more often than it changes; a few minutes of staleness is harmless |
| **Widget config/branding per company** (colors, greeting text, business hours) | CDN edge cache, long TTL, versioned/cache-busted URL on change | Practically static; served to millions of anonymous visitors, so pushing it to the edge matters most here |

The routing-relevant agent-availability cache is also where the [cache-aside concurrency race](../Building%20blocks/Scaling%20and%20Performance/Caching.md#5-the-concurrency-race-that-makes-cache-aside-genuinely-hard-not-just-sometimes-stale) is worth naming unprompted: if an agent's status flips to "busy" via one path while a stale in-flight read repopulates the cache with "available," you can route a second visitor to an already-full agent. A version/timestamp compare-and-set on the cache write (not just a short TTL) closes that race.

---

## 5. SQL database — tickets, agents, companies, SLAs

This half of the data model needs joins across normalized entities (which agent, which company, which SLA policy), and — critically — a **transactional correctness guarantee at ticket-assignment time**. That combination is exactly [RelationDBDesign §1](../RelationDBDesign.md#1-when-relational-is-the-right-default-precisely)'s case for relational.

**Core tables**: `companies`, `agents`, `customers`, `tickets`, `sla_policies`, `ticket_events_outbox` (see §6).

`tickets(id, company_id, customer_id, assigned_agent_id, status, sla_policy_id, created_at, first_responded_at, resolved_at)`

**Indexes, and why each one earns its cost**:
- `(company_id, status, created_at)` composite — the unassigned-queue scan ("next open ticket for this company") is the hottest read in the system and needs to be a range scan on a narrow index, not a table scan.
- `(assigned_agent_id, status)` — an agent's dashboard ("my active chats") is a per-agent equality lookup.
- A **partial index** on `status = 'open'` is worth naming explicitly: the unassigned-queue query only ever touches open tickets, and most tickets in the table are eventually resolved — indexing only the open subset keeps that index small and fast indefinitely as history grows, instead of degrading as the resolved-ticket count climbs.

**Primary key choice**: time-ordered (ULID/UUIDv7) rather than random UUIDv4 — tickets are created continuously at high volume, and a monotonic-ish key avoids the random B-tree scatter/page-split cost described in [RelationDBDesign §2](../RelationDBDesign.md#2-keys-the-choice-with-the-longest-lasting-consequences), while still allowing distributed ID generation without a central counter.

### The concurrency bug this design has to prevent

**Scenario**: two idle agents' dashboards both poll for "the next unassigned ticket" at nearly the same instant. Both read the same open ticket. Both try to claim it. Without protection, both succeed, and the visitor ends up double-assigned (or one assignment silently overwrites the other).

This is the queue-claiming version of [RelationDBDesign §8](../RelationDBDesign.md#8-worked-example-preventing-overselling-and-double-spending-completely)'s overselling problem, and it needs the same two distinct mechanisms:

```sql
BEGIN;

-- Row lock + SKIP LOCKED: don't just lock the row, skip past rows
-- another concurrent claimer already has locked, so two agents polling
-- at once land on two different tickets instead of queuing behind each other.
SELECT id FROM tickets
WHERE company_id = $1 AND status = 'open'
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;

UPDATE tickets SET status = 'assigned', assigned_agent_id = $2 WHERE id = $3;

-- Idempotency: a retried "claim" request from the same agent action is a no-op,
-- not a second assignment.
INSERT INTO ticket_assignment_log (idempotency_key, ticket_id, agent_id)
VALUES ($4, $3, $2) ON CONFLICT (idempotency_key) DO NOTHING;

COMMIT;
```

`FOR UPDATE` alone would be correct but would serialize every agent behind a single lock while claiming from a shared queue; `SKIP LOCKED` is the detail that makes concurrent claiming from the same pool actually scale, and it's worth naming as a refinement on the base double-spend pattern rather than a different mechanism.

---

## 6. NoSQL database — chat transcripts

Message volume (~5,000/sec at peak, append-only, one conversation read back as a contiguous ordered block) is the opposite shape from the ticket data above — high write throughput, no joins, no cross-message transactions, and exactly one dominant access pattern. Forcing this into the same relational store both wastes the relational engine's strengths and fights its constraints, which is the case [noSqlDesign §1](../noSqlDesign.md#1-the-mindset-shift-design-backward-from-queries-not-forward-from-entities)'s access-pattern-first framing is built for.

**Access patterns, enumerated first, per the doc's own discipline**:
1. Get all messages in a conversation, in order (the hot path — rendering a chat window, and reloading history when a visitor reopens the widget).
2. Get a customer's past conversations, most recent first (agent pulling up "what has this person asked before").
3. (Secondary, explicitly lower-priority) Full-text search across transcripts.

**Schema** (DynamoDB/Cassandra-shaped):

| pk | sk | Item |
|---|---|---|
| `CONV#c1` | `MSG#2026-09-17T10:02:01#m1` | Message body, sender, seq number |
| `CONV#c1` | `MSG#2026-09-17T10:02:04#m2` | Message body, sender, seq number |
| `CUST#u9` | `CONV#2026-09-17#c1` | Conversation summary (denormalized: last message preview, status) |

- **Pattern 1** resolves as `pk = CONV#c1, sk begins_with MSG#` — one query, already time-ordered for free because the sort key is timestamp-prefixed, same trick as [noSqlDesign §4](../noSqlDesign.md#4-single-table-design-the-technique-that-separates-a-real-dynamodbcassandra-answer-from-a-surface-level-one)'s order-history example.
- **Pattern 2** resolves against the `CUST#u9` partition directly — no GSI needed, because the access pattern was designed for up front.
- **Pattern 3** is the one that doesn't fit either partition — exactly the case for a **derived search layer** (Elasticsearch/OpenSearch), fed asynchronously off the same event stream in §7, rather than forcing full-text search into the primary transcript store. This is a direct application of [noSqlDesign §10](../noSqlDesign.md#10-when-not-to-use-nosql-including-the-mid-project-version-of-this-question): don't force an unsupported access pattern into the primary store, add a purpose-built layer fed by CDC/events instead.

**Partition key risk**: an unusually large enterprise customer running one extremely active, long-lived support conversation could push a single `conversation_id` partition past its per-partition throughput ceiling regardless of total cluster capacity (the concrete numeric point in [noSqlDesign §3](../noSqlDesign.md#3-partition-keys-and-the-numbers-that-make-a-bad-key-concrete)). Mitigation: bucket suffix on the sort key for exceptionally long conversations, or simply cap and roll conversations into time-boxed segments — worth naming even though it's a rare case, since it's exactly the kind of hot-key question a follow-up will probe.

---

## 7. Event backbone — Kafka

Four independent consumers need to react to the same underlying facts (a ticket was created/assigned/resolved, a message was sent) without the core request path knowing anything about any of them: **analytics**, **SLA breach monitoring**, **webhook delivery to the customer's own systems**, and the **search indexer**. That's the textbook fan-out case from [Message Queues and PubSub §1](../Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#1-the-core-mechanism) — one event, many independent interested parties — and it's also the case that specifically wants **replay** (a new consumer, like a future "add a second webhook provider" or a backfill of the search index, should be addable without touching the producer), which is the property that tips this toward Kafka over a plain queue, per the doc's own one-line framing in [§8](../Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#8-technology-comparison).

**Topics**:
- `ticket_events` — `ticket_created`, `ticket_assigned`, `ticket_resolved`, `sla_breach_warning`.
- `message_events` — one event per chat message sent (much higher volume than `ticket_events`, kept separate so a slow consumer of one doesn't create backpressure on the other).

**Partition key = `conversation_id`** (equivalently `ticket_id`, since each ticket maps to one conversation): every event about a given conversation needs to stay ordered relative to the others (you can't process `ticket_resolved` before `ticket_assigned`), but there's no ordering requirement *across* different conversations — so partitioning by `conversation_id` maximizes parallelism while preserving exactly the ordering guarantee that matters, the same reasoning as the doc's own order-processing worked example in [§9](../Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#9-worked-example-order-processing-pipeline).

**Consumer groups (independent fan-out)**:
- **Analytics** — aggregates volume/SLA metrics for the admin dashboard; can tolerate lag, doesn't need low latency.
- **SLA monitor** — watches `ticket_assigned` events and fires `sla_breach_warning` if `first_responded_at` isn't set within the policy window; this is the consumer with the tightest latency SLO (<30s) in the whole pipeline.
- **Webhook dispatcher** — calls each company's configured webhook URL. This consumer talks to *arbitrary third-party endpoints you don't control*, which can be slow, rate-limiting, or entirely down — it gets **its own DLQ**, isolated from the other three consumers, so a broken customer webhook can never back up SLA monitoring or analytics (the same isolation argument as the notifications consumer in the doc's worked example, [§9 point 5](../Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#9-worked-example-order-processing-pipeline)).
- **Search indexer** — writes into Elasticsearch for the transcript-search access pattern noted in §6.

**Dual-write problem, solved explicitly**: a ticket-assignment write and its `ticket_assigned` event must not disagree — if the process crashed between committing the SQL transaction and publishing to Kafka, the event would silently vanish even though the assignment is durable. Fix: a **transactional outbox** — the ticket-assignment SQL transaction (§5) also inserts a row into `ticket_events_outbox` in the *same* transaction, and a relay process (or Debezium CDC tailing the outbox table) publishes to Kafka and marks it sent. This is the direct application of [Message Queues and PubSub §3](../Building%20blocks/Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#the-dual-write-problem-a-favorite-follow-up).

**Delivery semantics**: design for at-least-once and make every consumer idempotent, not for a blanket exactly-once — the webhook dispatcher keys its calls to the customer's endpoint on the event's own ID so a redelivered `ticket_assigned` event doesn't fire the webhook twice; the search indexer's writes are naturally idempotent (an upsert keyed by message ID).

---

## 8. Consistency summary — what's strong, what's eventual, and why

| Data | Consistency | Why this is the right choice, not a compromise |
|---|---|---|
| Ticket claim/assignment (SQL) | Strong (row lock + transaction) | Two agents claiming the same ticket is a real, user-visible correctness bug — this is the one place in the design that cannot be eventually consistent |
| Chat message delivery (WebSocket + transcript store) | Ordered per-conversation, at-least-once, deduped by sequence number | The channel itself gives no guarantee by default; correctness is explicitly built on top, per [Websockets §3](../Building%20blocks/Communication/Websockets.md#3-what-a-websocket-does-not-give-you) |
| Agent availability cache | Short-TTL + event-driven invalidation, not strong | A few seconds of staleness costs a slightly suboptimal routing decision, not a correctness violation — cheap to tolerate |
| Analytics / SLA dashboards | Eventually consistent (Kafka consumer lag) | Nobody needs sub-second-fresh aggregate metrics; SLA *breach detection* itself has a real latency SLO (<30s) but the reporting dashboard doesn't |
| Search index | Eventually consistent (event-driven from CDC) | Explicitly a derived, secondary access pattern — never the source of truth |

---

## 9. Senior vs. staff bar for this question

- **Senior**: correctly proposes WebSockets for chat and *some* form of polling for the queue; knows a load balancer and a cache belong somewhere; picks a SQL database for tickets and "probably NoSQL" for messages without a sharp reason why; knows Kafka/SQS exist and mentions one.
- **Staff**: justifies long polling specifically from the "arbitrary third-party network" constraint rather than defaulting to it; names that the WebSocket gateway needs a resource-aware load-balancing signal and explicit connection draining, not naive least-connections; makes the caching decision per field with an explicit staleness-cost argument for each; catches the ticket-double-claim race unprompted and reaches for `FOR UPDATE SKIP LOCKED` specifically, not just a generic lock; enumerates NoSQL access patterns before designing the schema and explains *why this data* (not just "chat apps use NoSQL"); partitions Kafka by `conversation_id` with the ordering-vs-parallelism tradeoff stated explicitly; and names the dual-write problem and isolates the flaky third-party webhook consumer with its own DLQ without being prompted.

---

## 10. Follow-up questions to expect

- "Your webhook dispatcher is stuck retrying against a customer's dead endpoint — what happens to the rest of the pipeline?" → Nothing, if the DLQ and consumer-group isolation are actually separate per consumer; walk through why a shared queue here would have been wrong.
- "A visitor's laptop goes to sleep mid-chat and wakes up 40 seconds later — what do they see?" → Depends entirely on whether the resume protocol (sequence numbers against the durable transcript store) was actually built — a raw WebSocket reconnect gives nothing back by default.
- "One enterprise customer is 100x your median company's message volume — what breaks first?" → Walk through the Kafka partition-key hot-key risk and the DynamoDB/Cassandra partition throughput ceiling, and name the same family of fix (sub-bucketing) in both places.
- "Why not just use one database for everything?" → Contrast the transactional, join-heavy, low-volume ticket data against the append-only, single-partition-shaped, extreme-volume message data — the point isn't "NoSQL is faster," it's that the two datasets have genuinely different access-pattern shapes.
- "Why long polling instead of just using WebSockets everywhere for consistency?" → Cost (millions of mostly-idle persistent connections for an infrequent event) and reliability (WebSocket-hostile networks on arbitrary third-party sites) — say the cost of *not* unifying transports too: two tiers to operate instead of one.
