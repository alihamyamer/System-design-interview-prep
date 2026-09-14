# Long Polling

## Why this topic is a filter question

Long polling looks like a historical curiosity next to SSE and WebSockets, but it's still the right answer in specific environments (corporate proxies that kill long-lived streams, clients that can't use `EventSource` or WebSocket APIs, infrastructure that only understands request/response). The filter question is whether you understand *why* it behaves the way it does under load — connection churn, held-request capacity, and the gap-on-reconnect problem — rather than treating it as "SSE's worse cousin."

For the shared concepts across all three real-time transports (connection scaling, fan-out architecture, reconnection), see [Server-sent events (SSE)](Server-sent%20events%20%28SSE%29.md) and [Websockets](Websockets.md) — this doc focuses on what's specific to long polling.

---

## 1. The mechanism

Regular polling asks "anything new?" on a fixed interval regardless of whether anything happened — freshness is bounded by the interval, and most requests return "no." Long polling instead **holds the request open** until there's something to return or a timeout elapses:

1. Client sends a request.
2. Server checks for new data immediately; if present, responds right away.
3. If not, the server holds the connection (doesn't respond) until an event occurs or a timeout (typically 15–30s) is reached.
4. Client processes the response (real data, or an empty "no updates yet" timeout response) and immediately opens a new request.

The result: latency approaches "as fast as the server can respond to an event" rather than "as fast as the next poll interval," without needing a persistent bidirectional protocol.

---

## 2. The real cost: held-request capacity, not bandwidth

The naive framing is "long polling saves requests compared to short polling." The actual operational constraint that shows up at scale is different: **every waiting client is holding open a server-side request/thread/socket for up to the full timeout window.** With N concurrent clients and a T-second timeout, your servers need to sustain roughly N held connections at any instant, not N/T requests per second — this is a concurrency/memory problem more than a throughput problem, and it's the number worth doing back-of-envelope math on in an interview: 100k concurrent users each holding a 25s request is 100k concurrent held connections your fleet needs headroom for, continuously, even if most of them resolve to "nothing happened."

This is also where long polling loses to SSE/WebSockets at scale: a long-poll client still tears down and re-establishes a request after **every single response**, timeout or not — so a busy topic (frequent events) turns into request churn at roughly one full request lifecycle (TCP if not reused, TLS handshake amortized via keep-alive, auth check, routing) per event, whereas SSE/WebSockets pay that setup cost once and then stream. Long polling is most attractive precisely when events are infrequent enough that this churn stays low, and least attractive for chat-like frequencies.

---

## 3. The gap-on-reconnect problem

Between "client received response N" and "client's next request is registered and waiting," there is a real window during which an event can occur and have nowhere to land. A naive implementation drops it silently. This is the concrete failure scenario worth naming unprompted:

- If losing an occasional event is tolerable (e.g., a live "someone is typing" indicator), ignore it.
- If it isn't (e.g., a notification the user must see), the fix is **not** trying to make the gap zero-width — it's making the response self-healing: each poll response includes an event ID or timestamp, and the *next* request asks "give me everything since ID X" rather than "give me whatever's next," so a gap just means slightly delayed delivery instead of loss. This requires the server side to retain a short backlog (a small ring buffer per topic/user, or a durable event log) rather than only tracking "the next event," which is a real design decision, not a free property of the pattern.

---

## 4. Distributed architecture: where does the "wait" actually happen?

A long-poll endpoint on a single server is easy to picture — hold the HTTP thread until local state changes. In a horizontally scaled deployment it's not local state; the event that satisfies the held request might be produced by a completely different node handling a different request. The standard shape:

```
client --held request--> API server --subscribes--> pub/sub / broker (Redis Pub/Sub, Kafka, in-memory bus)
                                            ^
                                            |
                                     producer publishes event
```

The API server holding the client's request subscribes to the relevant channel/topic on the broker; when a matching event arrives, it completes the held response. This means long polling isn't actually simpler to operate at scale than SSE or WebSockets — it needs the same fan-out infrastructure behind it, it just doesn't need sticky sessions in front of it the way a stateful streaming connection often does, since each request is independent and can land on any server.

---

## 5. Failure modes and mitigations

| Failure | Effect | Mitigation |
|---|---|---|
| Load balancer/proxy enforces a shorter idle timeout than your held-request timeout | Held requests get killed early, client sees spurious disconnects | Set your server-side timeout comfortably below the infra's idle timeout; don't assume you control the full path |
| Reconnect storm (many clients disconnect and retry simultaneously — e.g., after a deploy or a broker blip) | Sudden spike of new requests all landing at once | Jittered client-side reconnect delay, same rationale as retry jitter in [Message Queues and PubSub §6](../Async%20and%20Messaging/Message%20Queues%20and%20PubSub.md#6-failure-handling-retries-poison-messages-and-dlqs "Failure handling: retries, poison messages, and DLQs") |
| Auth token expires while a request is held for 25s | Stale-auth requests get served or rejected inconsistently | Re-validate auth freshness at response time, not just at request start, for long-held requests |
| Server restart/deploy | All currently-held requests on that instance are dropped at once | Graceful drain: stop accepting new held requests, let in-flight ones resolve or hit a short forced-timeout response, so clients reconnect against a still-healthy instance rather than erroring |

---

## 6. Long polling vs. the alternatives — the actual decision

| | Long polling | SSE | WebSockets |
|---|---|---|---|
| Direction | Server→client (via repeated requests) | Server→client only | Bidirectional |
| Transport | Plain HTTP request/response | Long-lived HTTP response stream | Upgraded persistent connection |
| Request overhead | One full request per event/timeout | One connection setup, then free | One connection setup, then free |
| Proxy/infra friendliness | Highest — looks like normal HTTP | Good in modern browser stacks | Needs WebSocket-aware infra throughout the path |
| Best for | Infrequent updates, environments where streaming connections are unreliable (some corporate networks/proxies) | Frequent one-way push in modern browsers | Frequent bidirectional interaction |

The honest reason to pick long polling in 2026 is almost never "it's simpler than SSE" — SSE is simpler once you're targeting a normal modern browser and modern infra. It's "we need this to work through infrastructure that's known to kill long-lived streaming connections but tolerates normal request/response," or "we need a fallback for clients where `EventSource`/WebSocket isn't available." Naming that honestly, rather than defaulting to long polling out of habit, is the senior-vs-staff tell here.

---

## 7. What separates senior from staff here

- **Senior**: knows long polling holds a request open instead of fixed-interval polling, knows it's more efficient than short polling and less efficient than SSE/WebSockets.
- **Staff**: reasons about held-connection *capacity* as the real scaling constraint (not request rate); designs the reconnect gap to be self-healing via event IDs and a short backlog rather than accepting silent loss; knows the broker/pub-sub fan-out layer is required infrastructure, not an optional extra, in a multi-node deployment; picks long polling for an honest infra-compatibility reason rather than as a default.

---

## 8. Questions to expect, and what a strong answer sounds like

- **"100k users are long-polling with a 30s timeout — what does that mean for your servers?"** → Roughly 100k concurrent held connections/threads sustained continuously, which is a concurrency/memory sizing problem, independent of how often events actually fire.
- **"A user says they missed a notification right after reconnecting — why, and how do you fix it?"** → The gap-on-reconnect window between one held request ending and the next beginning; fix with event IDs/timestamps and a short server-side backlog so the next request can ask "since X" instead of "what's next."
- **"You deploy a new server version — what happens to clients with an open held request on the old instance?"** → Without a graceful drain, the request errors and the client has to detect and retry; with one, in-flight requests resolve or get a clean forced-timeout response first.
- **"Why not just use SSE here?"** → A good answer either picks SSE (if the client population is modern browsers with no proxy issues) or names the specific infra constraint that rules it out — not "long polling is what we've always used."
