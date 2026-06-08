Good — the board fills in the gaps from the clarifying questions. Reading it back, the confirmed scope is: **1:1 + group messaging, real-time, offline messages delivered on reconnect, delivery/read visibility, hard delete (data is gone), text only ≤30 KB.** NFRs: **10M users, ~30 KB max message, 100 ms latency target, ordering is crucial, durability, consistency.** I'll design to exactly those.

## Capacity sizing (back-of-envelope)

10M registered users doesn't mean 10M sockets. Assume ~10–15% peak concurrency → roughly **1–1.5M concurrent WebSocket connections**. A tuned connection server (epoll, ~20–30 KB/conn) comfortably holds ~100K live sockets, so you need **~10–15 gateway nodes**, doubled for headroom/failover → ~25–30. If each active user sends ~1 msg / 10s at peak, that's ~**100K msgs/sec** ingest; with group fan-out, reads dominate writes by 10–100×. At ~1 KB average payload that's tens of MB/s — easily handled by a partitioned log plus a sharded store. These numbers justify the component choices below.

## Architecture## The WebSocket layer (the heart of it)

A WebSocket gives you one full-duplex TCP connection per client that stays open, so the server can *push* without the client polling — essential for real-time chat and presence.

**Establishing the connection.** The client opens `wss://…` through an L4 load balancer (AWS NLB, not an L7 ALB-only setup) because you want the TCP/WS upgrade passed through and the connection pinned to one gateway for its lifetime. The handshake carries an auth token; the gateway validates it, resolves `userId`, creates a `Connection` object, and writes `userId → {gatewayNodeId, connId}` into the **session registry** (Redis) with a TTL. A user with phone + laptop + web simply has three entries.

**Heartbeats do double duty.** The client sends a WS ping (or an app-level `heartbeat`) every ~30s. The gateway refreshes the Redis TTL (liveness) and stamps `lastActiveAt` for the **presence service**. A background sweep flips users whose heartbeat lapsed to `AWAY`; a closed socket flips them to `OFFLINE`. Presence deltas are pushed only to users who share a conversation, not broadcast globally.

**Reconnect + resume.** Mobile networks drop constantly, so the client reconnects with exponential backoff and sends its `lastAckedSeq` per conversation. The server replays everything after that watermark from the store — this is the same mechanism that handles **offline delivery**: a message for a disconnected user is durably stored and streamed on the next connect.

**The routing problem.** The sender's socket lives on gateway A, the recipient's on gateway B. So the gateway never delivers directly — it forwards inbound messages to the message service, and a separate delivery service routes the result back to whichever node owns the recipient (looked up in the session registry, last hop over Redis pub/sub or gRPC to that node). This decoupling is what lets you add gateway nodes linearly as connections grow.

## Message lifecycle (send → durable → deliver)## How each NFR is satisfied

**Ordering is crucial → partition by conversation.** All writes for a given `conversationId` are keyed to a single Kafka partition, so they're committed and consumed in a strict total order; the offset *is* the order. The message service derives a contiguous per-conversation `seq` at append time. Clients render by `seq` and dedup by `clientMsgId`, so even out-of-order socket arrival sorts correctly and a retried send never duplicates.

**Durability + consistency → log-before-ack.** The service does not ack the sender until Kafka has *committed* the append (replicated across brokers). The log is the source of truth; two consumers fan off it — one persists to the message store (Cassandra/DynamoDB, replicated), one delivers. Because the ack is gated on a durable commit, the sender gets read-your-writes consistency, and a crash mid-flight loses nothing.

**100 ms latency → lean hot path.** The only synchronous step before the sender's ack is the Kafka append (single-digit ms) plus a registry lookup; persistence to the store happens asynchronously off the log. Same-region routing, in-memory session lookups in Redis, and the persistent socket (no connection setup per message) keep p99 within budget.

**Visibility → two receipt signals.** When gateway B pushes the message and the client confirms, a `delivery_receipt` flows back to the sender (status `SENT → DELIVERED`). When the user actually views it, the client sends `mark_read{seq}`, which advances that member's `lastReadSeq` and emits a `read_receipt` to the others. Storing one read watermark per member (not a row per user per message) is what makes receipts cheap at 10M scale.

**Hard delete → "data is gone."** Deletes are a separate operation against the materialized message store (physical row removal, not a tombstone), plus a `message_deleted{messageId}` event fanned out so connected clients drop it from view. One nuance worth raising in the interview: an append-only log can't truly erase a record, so the Kafka log is treated as a short-retention transport (TTL'd / compacted) while the *store* is the deletable system of record — that split is how you honor a hard-delete requirement without contradicting the durable log.

## APIs

**WebSocket events** carry everything latency-sensitive. Client → server: `send_message`, `mark_read`, `typing`, `heartbeat`. Server → client: `message`, `delivery_receipt`, `read_receipt`, `presence_changed`, `message_deleted`.

**REST** handles the control plane and history: `POST /conversations` and `POST /conversations/{id}/members` (create/manage), `GET /conversations/{id}/messages?afterSeq=…` (paginated history and reconnect backfill), `DELETE /messages/{id}` (hard delete), `GET /users/{id}/presence`. The split is the rule of thumb: persistent socket for the real-time data plane, request/response REST for everything that's a one-off query or mutation.

For broadcast channels (in the original prompt though not on the board), the only delta is the fan-out set — a `BroadcastChannel` resolves recipients to its full subscriber list and restricts posting to publisher IDs; the log/durability/ordering machinery is identical.

If you want, I can go deeper on the trickiest follow-up an AWS interviewer tends to push on here: exactly how the per-conversation sequencer stays correct under concurrent senders (Kafka partition ordering vs. an atomic DynamoDB counter vs. a dedicated sequencer service), and the failure modes of each.
