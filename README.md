# 🦀 PubSub — High-Performance Pub/Sub Broker in Rust

A blazing-fast, in-memory publish/subscribe message broker built in Rust, designed for microsecond-level latency with at-least-once delivery guarantees and bounded message replay.

---

## ✨ Features

| Feature | Strategy | Detail |
|---|---|---|
| **Message Replay** | Bounded Ring Buffer | New/recovering consumers can replay recent messages from in-memory history |
| **Topic Routing** | Exact Match (`HashMap`) | O(1) lookup latency |
| **Delivery Guarantee** | At-Least-Once | Messages held in retry buffer until explicit ACK |
| **Persistence** | In-Memory | All state stored in RAM for microsecond throughput |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                    PubSub Broker                    │
│                                                     │
│  Publisher ──▶  [Topic Router (HashMap)]            │
│                        │                            │
│               ┌────────┴────────┐                   │
│               ▼                 ▼                   │
│        [Subscriber A]    [Subscriber B]             │
│               │                 │                   │
│        [Retry Buffer]    [Retry Buffer]             │
│               │                                     │
│        [Replay Ring Buffer]  (shared per topic)     │
└─────────────────────────────────────────────────────┘
```

### Messaging Model — Message Replay

The broker retains a **bounded history** of messages per topic using a **ring buffer** in RAM. When a new consumer connects or a consumer recovers from a crash, it can request past events from the replay buffer.

- **Rationale:** Enables state recovery and event sourcing patterns without a persistent store.
- **Eviction Policy:** Supports both size-based limits and TTL-based expiry to prevent OOM conditions.
- **Constraint:** Consumers that fall too far behind may miss messages that have been evicted.

### Routing — Exact Match

Topic routing is handled via `HashMap<TopicName, Vec<Subscriber>>` lookups.

- **Complexity:** O(1) per publish operation.
- **Rationale:** Minimizes routing overhead to keep end-to-end latency in the microsecond range.

### Delivery — At-Least-Once

Messages are held in a **per-consumer retry buffer** and are only removed once an explicit **ACK** is received from the consumer.

- **Rationale:** Prevents message loss during network partitions or consumer crashes.
- **Important:** Consumers **must be idempotent** — duplicate delivery is possible and expected.

### Persistence — In-Memory Only

All state (routing tables, replay history, retry buffers) lives exclusively in RAM.

- **Benefit:** Achieves microsecond-level latency, bottlenecked only by CPU and network.
- **Risk:** A broker crash or restart results in **total data loss** of the active queue and replay history. This is a deliberate trade-off for performance.



MIT — see [LICENSE](LICENSE) for details.
