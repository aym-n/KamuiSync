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

<img width="745" height="650" alt="image" src="https://github.com/user-attachments/assets/eb0819ba-1f02-4084-9d91-da68cefdcc24" />


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

## Folder Structure 
```
pubsub/
├── Cargo.toml                  # workspace root
├── Cargo.lock
├── README.md
├── LICENSE
├── .env.example
├── docker-compose.yml
│
├── crates/
│   ├── broker/                 # core broker engine (the HLD we designed)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── router/         # topic routing (HashMap fan-out)
│   │       │   ├── mod.rs
│   │       │   └── topic.rs
│   │       ├── buffer/         # ring buffer + retry buffer
│   │       │   ├── mod.rs
│   │       │   ├── replay.rs
│   │       │   └── retry.rs
│   │       ├── subscriber/     # subscriber state + registry
│   │       │   ├── mod.rs
│   │       │   └── registry.rs
│   │       ├── publisher/
│   │       │   ├── mod.rs
│   │       │   └── handle.rs
│   │       ├── ack/            # ACK processor
│   │       │   └── mod.rs
│   │       ├── eviction/       # TTL + size reaper task
│   │       │   └── mod.rs
│   │       └── error.rs
│   │
│   ├── proto/                  # shared message types + serialization
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── message.rs      # Message, TopicName, MsgId, Offset
│   │       ├── command.rs      # Publish, Subscribe, ACK, Replay enums
│   │       └── codec.rs        # encode/decode (bincode / protobuf)
│   │
│   ├── transport/              # pluggable network layer
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── traits.rs       # Transport trait
│   │       ├── tcp/
│   │       │   ├── mod.rs
│   │       │   └── listener.rs
│   │       ├── ws/             # WebSocket (future)
│   │       │   └── mod.rs
│   │       └── grpc/           # gRPC (future)
│   │           └── mod.rs
│   │
│   ├── auth/                   # authn/authz (future)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── traits.rs       # Authenticator trait
│   │       ├── token.rs        # JWT / HMAC
│   │       └── acl.rs          # topic-level ACL rules
│   │
│   ├── persistence/            # optional WAL / snapshot (future)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── traits.rs       # Storage trait
│   │       ├── wal.rs          # Write-Ahead Log
│   │       └── snapshot.rs
│   │
│   ├── metrics/                # observability (future)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── counters.rs     # publish rate, ACK latency, buffer depth
│   │       └── exporter.rs     # Prometheus / OpenTelemetry
│   │
│   └── cli/                    # admin CLI (future)
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs
│           └── commands/
│               ├── inspect.rs  # list topics, subscribers
│               └── replay.rs   # manual replay trigger
│
├── server/                     # binary entrypoint — wires everything together
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       └── config.rs           # config file parsing (TOML / env)
│
├── client/                     # official Rust client SDK (future)
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       ├── connection.rs
│       ├── publisher.rs
│       └── subscriber.rs
│
├── tests/
│   ├── integration/
│   │   ├── publish_subscribe.rs
│   │   ├── replay.rs
│   │   └── at_least_once.rs
│   └── load/
│       └── throughput_bench.rs
│
└── benches/
    └── broker_bench.rs         # criterion benchmarks

```

MIT — see [LICENSE](LICENSE) for details.
