# WebSockets

## Why this topic is a filter question

WebSockets are the most over-prescribed answer in real-time system design interviews — candidates reach for them whenever "real-time" is mentioned, without asking whether the feature actually needs bidirectional, low-latency, high-frequency messaging. The filter is whether you can justify the cost (stateful connections, harder scaling, no free HTTP tooling) against a specific requirement, and whether you know what a WebSocket connection does *not* give you for free: ordering, delivery guarantees, or resumability.

For shared real-time-transport concepts, see [Server-sent events (SSE)](Server-sent%20events%20%28SSE%29.md) (one-way alternative) and [Long Polling](Long%20Polling.md) (HTTP-native fallback) — this doc focuses on what's specific to full-duplex connections.

---

## 1. The mechanism

A WebSocket connection starts as a normal HTTP request with an `Upgrade: websocket` header; after a successful handshake, the same TCP connection is repurposed for a **full-duplex** message channel — both sides can send frames at any time, independent of each other, without a request/response cycle per message. This is the property SSE and long polling fundamentally don't have: the client isn't just receiving, it's an equal participant on the same channel.

---

## 2. The actual decision: does the client need to speak, frequently, on the same low-latency channel?

This is the single question that should drive whether you reach for WebSockets at all. Break the "real-time" requirement into two independent axes instead of treating it as one blob:

| | Client → Server needed? | Frequency | Right tool |
|---|---|---|---|
| Live dashboard, notifications, price ticker | No (writes are separate REST calls) | Any | SSE |
| Chat | Yes | High | WebSockets |
| Multiplayer game state | Yes | Very high, latency-sensitive | WebSockets (often UDP-based outside the browser, but WebSockets in-browser) |
| Collaborative editing (cursors, live ops) | Yes | High | WebSockets |
| "Live" admin panel that occasionally lets you click a button | Sort of, but infrequent | Low | REST is fine; the button click doesn't need the push channel |

The trap: a feature that's 95% server-push and 5% occasional client action doesn't need WebSockets just because of that 5% — a REST endpoint alongside an SSE stream covers it with far less operational cost. Reserve WebSockets for when the client-to-server direction is itself frequent and latency-sensitive, not merely present.

---

## 3. What a WebSocket does *not* give you

This is the section that separates candidates who've operated one of these systems from candidates who've only read about them. A WebSocket is a **byte-stream channel**, nothing more — everything below is something you build on top of it, not something it hands you:

- **Delivery guarantee**: if the connection drops mid-message or the server crashes after sending but before the client processed it, that message may be lost. If losing a chat message is unacceptable, the client needs to reconcile against durable history on reconnect ([§5](#5-scaling-connection-pinning-is-the-real-complexity-not-throughput "Scaling: connection pinning is the real complexity, not throughput")), not trust the socket alone.
- **Ordering across reconnects**: within one open connection, TCP guarantees ordering. Across a disconnect/reconnect, there's no inherent guarantee the client hasn't missed messages or that a new connection routes to a server with the same in-memory state — this needs an explicit resume protocol, same idea as SSE's `Last-Event-ID` but nothing standardizes it for WebSockets, so you design it yourself (sequence numbers, a "since" parameter on reconnect).
- **Deduplication**: a reconnect-and-replay strategy can hand the client the same message twice; if replaying, the message needs a stable ID so the client (or server) can dedupe — the same idempotency discipline as [Message Queues and PubSub §7](../Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#7-idempotency-in-practice "Idempotency in practice"), just at the transport layer instead of the broker layer.
- **Liveness detection**: a dead connection (client's laptop went to sleep, network dropped without a clean close) doesn't announce itself — TCP can sit "open" for a long time with nothing flowing. This is what heartbeats are for ([§4](#4-connection-lifecycle-and-heartbeats "Connection lifecycle and heartbeats")).

A strong interview answer treats all of this as "the WebSocket is the transport; correctness is a protocol you design on top of it," rather than assuming the socket itself provides application-level guarantees.

---

## 4. Connection lifecycle and heartbeats

A production WebSocket system needs an explicit answer for each stage:

1. **Connect + authenticate** — the handshake itself should carry auth (token in the connection URL or first message), since there's no per-message `Authorization` header the way REST has.
2. **Subscribe / join** — client tells the server which rooms/channels/topics it cares about (chat room ID, document ID); server sets up the corresponding broker subscription.
3. **Heartbeat** — a periodic ping/pong (client or server initiated) to detect a dead connection that hasn't cleanly closed. Without this, a server can hold state and resources for a connection that's actually gone, and a client can believe it's connected when it isn't. Typical interval: 15–30s ping, with a small multiple of that as the dead-connection threshold.
4. **Disconnect detection and cleanup** — server needs to release any per-connection state (presence, subscriptions) promptly on detected disconnect, or "who's online" and similar features drift out of sync with reality.
5. **Reconnect + resync** — client reconnects (typically with jittered backoff, same rationale as retry jitter elsewhere in this document set) and needs to recover any state it may have missed — see [§5](#5-scaling-connection-pinning-is-the-real-complexity-not-throughput "Scaling: connection pinning is the real complexity, not throughput").

---

## 5. Scaling: connection pinning is the real complexity, not throughput

A WebSocket connection is stateful and long-lived, which breaks the assumption behind stateless horizontal scaling (any request can hit any server). The standard architecture:

```
client <--persistent conn--> gateway/edge server --subscribes--> broker (Redis Pub/Sub, Kafka, NATS)
                                                                        ^
                                                            backend services publish events
```

Two concrete design questions this forces:
- **Is the client pinned to the server it connected to, or can any server handle any client?** If per-connection state lives only in the broker subscription (not in server-local memory), any gateway instance can serve any client, which makes rolling deploys and autoscaling far simpler — worth explicitly designing for statelessness at the gateway layer rather than accepting sticky sessions as a given.
- **How does a message meant for user X reach whichever specific gateway instance user X is connected to?** The broker fan-out pattern above — the gateway holding X's connection is subscribed on X's behalf and forwards matching messages down that specific socket. This is the same shape as [SSE's fan-out](Server-sent%20events%20%28SSE%29.md#4-scaling-connection-count-is-the-constraint-and-the-fan-out-problem-is-universal "Scaling: connection count is the constraint, and the fan-out problem is universal") with the added wrinkle that the gateway might also need to accept inbound messages *from* the client and route them to backend services.

The number worth being ready to reason about: connection count per instance is typically bounded by memory and file descriptors well before CPU — sizing a fleet for "N concurrent connections" is a different calculation than sizing for "N requests/sec," and conflating the two is a common under-provisioning mistake.

---

## 6. Message format tradeoff

JSON is easy to debug (readable in devtools, easy to version informally) but has real overhead at high message volume. Binary formats (Protobuf, MessagePack) cut payload size and parse cost, which matters once you're pushing thousands of messages/sec/connection (e.g., multiplayer state deltas) but costs you debuggability and adds a schema-management burden similar to [gRPC's §5](RPC%20and%20gRPC.md#5-schema-evolution-where-strong-typing-cuts-both-ways "Schema evolution — where strong typing cuts both ways"). Pick JSON by default; move to binary only once message volume/size is measured to actually matter — this is a case where premature optimization is a real, common mistake in this space.

---

## 7. Security specifics beyond the general auth story

- **Channel/room authorization must be enforced at subscribe time, server-side** — the same principle as [SSE §6](Server-sent%20events%20%28SSE%29.md#6-security-specifics "Security specifics"): a client sending `{"action":"join","room":"other_tenant_room_id"}` must be rejected by the server based on the authenticated identity's actual permissions, not merely by the room ID being hard to guess.
- **Abuse and flooding**: because the client can push messages at will (unlike SSE), a WebSocket gateway needs per-connection rate limiting on inbound messages, or a single misbehaving or compromised client can flood a room/channel.
- **Origin checks on the handshake** — WebSocket connections aren't subject to the same-origin policy the way `fetch`/XHR are by default, so `Origin` header validation at connect time is a real, sometimes-missed control.

---

## 8. What separates senior from staff here

- **Senior**: knows WebSockets are full-duplex and persistent, knows they're for chat/games/collaboration, knows scaling needs a pub/sub layer.
- **Staff**: decomposes "real-time" into the two independent axes (needs push? needs frequent client→server on the same channel?) before reaching for WebSockets at all; explicitly names that the socket gives you a byte channel, not delivery/ordering/dedup guarantees, and designs those separately; treats heartbeats and disconnect cleanup as required infrastructure, not optional polish; designs the gateway layer to be stateless (state lives in the broker subscription) so scaling and deploys stay simple; picks JSON vs. binary based on measured volume, not by default.

---

## 9. Questions to expect, and what a strong answer sounds like

- **"Why WebSockets instead of SSE for this feature?"** → Name the specific requirement for frequent, low-latency client→server messages on the same channel — not "it's more real-time."
- **"A user's laptop goes to sleep mid-connection — how does the server find out?"** → It doesn't, automatically; heartbeats (ping/pong) are what surface a dead connection that never sent a clean close frame.
- **"How does a message from service X reach a specific connected user who might be on any of 200 gateway instances?"** → Broker-based fan-out; the gateway instance holding that user's connection is subscribed on their behalf.
- **"A user reconnects after a 30-second network drop — what messages do they see?"** → Depends on whether you built a resume protocol (sequence numbers / "since" cursor against durable history); a raw WebSocket gives you nothing here by default — this is a deliberate design decision, not automatic.
- **"How do you stop one client from flooding a chat room?"** → Per-connection inbound rate limiting at the gateway, enforced server-side, independent of client behavior.
