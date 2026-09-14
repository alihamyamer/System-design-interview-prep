# Server-Sent Events (SSE)

## Why this topic is a filter question

SSE is the "boring, correct" answer for one-way server push, and interviewers use it to check whether you reach for WebSockets reflexively (over-engineering) or dismiss SSE as a toy (under-informed). The filter is whether you can articulate precisely why one-way push doesn't need a bidirectional protocol, and what SSE actually guarantees — and doesn't — around delivery and reconnection.

For shared real-time-transport concepts (connection scaling, fan-out via a broker, reconnect storms), see [Websockets](Websockets.md) and [Long Polling](Long%20Polling.md) — this doc focuses on what's specific to SSE.

---

## 1. The mechanism

The client opens a normal HTTP request (`GET /notifications/stream`) via the browser's `EventSource` API. The server keeps the response open and writes a sequence of text-formatted events to it over time, instead of closing after one response body:

```
event: low_stock
data: {"sku":"sku_123","available":4}
id: 8842

event: low_stock
data: {"sku":"sku_777","available":1}
id: 8843
```

- `data`: the payload (required)
- `event`: an optional named event type, so the client can dispatch to different handlers
- `id`: an event identifier — this is what makes reconnection self-healing ([§3](#3-reconnection-and-last-event-id-sses-actual-killer-feature "Reconnection and `Last-Event-ID` — SSE's actual killer feature"))
- `retry`: a server-supplied hint for how long the client should wait before reconnecting

It's plain HTTP the whole way — no protocol upgrade, no special infrastructure requirement beyond "don't time out or buffer a long-lived response," which is the main reason it's the simplest of the three real-time options operationally.

---

## 2. Why one-way is a feature, not just a limitation

The reflexive framing is "SSE is WebSockets without the send direction." The better framing: **most real-time UI features are actually one-way** — notifications, live dashboards, price tickers, job progress, activity feeds. The client's "writes" in these systems (marking a notification read, adjusting a dashboard filter) are almost always fine as ordinary REST calls, because they're infrequent and don't need to ride the same low-latency channel as the push stream.

Choosing SSE over WebSockets for a one-way feature isn't a compromise — it buys you real HTTP tooling for free: standard proxies, standard auth middleware, standard load balancer health checks, and (per §6) automatic browser-level reconnection that a raw WebSocket doesn't give you out of the box. The cases that genuinely need WebSockets are the ones where the *client* also needs to emit frequent, low-latency messages on the same channel — chat, multiplayer state, collaborative editing operations (see [Websockets §2](Websockets.md#2-the-actual-decision-does-the-client-need-to-speak-frequently-on-the-same-low-latency-channel "The actual decision: does the client need to speak, frequently, on the same low-latency channel?")).

---

## 3. Reconnection and `Last-Event-ID` — SSE's actual killer feature

This is the detail that most differentiates a strong SSE answer from a shallow one. The `EventSource` browser API has **built-in automatic reconnection**: if the connection drops, the browser reconnects on its own (after the `retry` hint, or a default backoff) and sends a `Last-Event-ID` header set to the most recent `id` it saw.

That means the *server* gets to decide what "resume" means, entirely through the `id` field:
- If events are held in a short-lived buffer or a durable log keyed by ID, the server can replay everything after `Last-Event-ID` on reconnect — no gap, no duplicate suppression logic needed on the client.
- If the server ignores `Last-Event-ID` and just resumes the live feed, reconnects silently drop whatever happened during the gap.

Whether that gap is acceptable is a business decision, not a technical one — worth stating explicitly: "a missed live notification is fine because the client can also refetch unread counts on demand" is a different answer than "a missed job-progress event means the UI shows the wrong state until the job finishes," and only the second one requires investing in a backing event log with replay.

---

## 4. Scaling: connection count is the constraint, and the fan-out problem is universal

Like WebSockets, an SSE server holds one open connection per client for the lifetime of that client's session — the scaling ceiling is concurrent connections per instance, not requests per second. Where does the *data* for those connections come from across a horizontally scaled fleet? The same shape as every other push mechanism in this document set:

```
producer --publish event--> broker (Redis Pub/Sub / Kafka / internal bus) --fan out--> N app servers --push--> connected clients
```

Each app server subscribes to the topics its currently-connected clients care about and forwards matching events down their open SSE connections. The two follow-up questions worth pre-empting:
- **Does a client need to stay pinned to the same server (sticky sessions)?** Only if server-local state matters beyond the open connection itself; if all routing state lives in the broker subscription, any server can serve any client, and a reconnect after a deploy can land anywhere.
- **What's the per-instance connection ceiling?** Bounded by open file descriptors and per-connection memory more than CPU — worth sizing explicitly (e.g., "we expect 50k concurrent connections at peak, our instances handle ~20k comfortably, so we need at least 3 instances just for connection headroom, independent of event throughput").

---

## 5. Reliability: SSE gives you a channel, not a delivery guarantee

By default, SSE is **fire-and-forget from the server's perspective** — if nobody's listening when an event is published, it's gone unless you've built the replay mechanism in §3. This is worth contrasting explicitly with a durable message queue: SSE is the *last-mile delivery* mechanism to a connected browser, not a substitute for the durable event system that should sit behind it if the underlying facts matter (see [Message Queues and PubSub](../Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md)). A common, correct architecture is: durable topic → SSE gateway subscribes and forwards to connected clients → client also has a REST endpoint to fetch current state on load/reconnect, so SSE is treated as a freshness optimization on top of a source of truth the client can always re-derive from, not the only path data travels.

---

## 6. Security specifics

- **Auth at connection time is not enough for long-lived streams.** A stream opened with a short-lived token can outlive that token; either re-validate periodically server-side, or design the stream to be cleanly closeable so the client is forced to reconnect (and re-auth) on a bounded interval.
- **Per-tenant/per-user channel isolation must be enforced server-side**, not by obscurity of a channel name — a user should never be able to open `GET /stream?channel=other_org_id` and receive another tenant's events because the channel ID is guessable. The subscription the server sets up on the broker should be derived from the authenticated identity, not from a client-supplied parameter.

---

## 7. What separates senior from staff here

- **Senior**: knows SSE is one-way server push over plain HTTP, knows it's simpler than WebSockets, can name the `EventSource` API.
- **Staff**: treats `Last-Event-ID` + a backing buffer/log as the actual mechanism for a reliable resume, not an afterthought; sizes connection capacity explicitly as the scaling constraint; separates "is a missed event acceptable" as a product decision with different answers per feature; designs SSE as the last-mile layer on top of a durable event source rather than the source of truth itself; derives channel subscriptions from authenticated identity, never from client-supplied parameters.

---

## 8. Questions to expect, and what a strong answer sounds like

- **"A user's SSE connection drops for 10 seconds during a network blip — what do they see when it reconnects?"** → Depends entirely on whether the server implements `Last-Event-ID` replay; walk through both the "acceptable gap" and "needs replay from a backing log" cases and which features fall into each.
- **"How do you push an event to a specific user when they might be connected to any of 50 app servers?"** → Broker-based fan-out (Redis Pub/Sub/Kafka) — each server subscribes on behalf of its currently connected clients; no sticky routing needed if that's the only shared state.
- **"Why SSE instead of WebSockets here?"** → Name that the feature is one-way, and that SSE buys plain-HTTP infra compatibility and free browser-level reconnection in exchange for not needing client→server push on the same channel.
- **"Could a user see another tenant's live events?"** → Only if channel subscription is derived from a client-supplied value instead of the authenticated session — describe deriving it server-side instead.
