---
name: communication
description: Index of communication-style skills — REST, gRPC, GraphQL, WebSockets, SSE, long-polling, webhooks, message queues, pub/sub, Kafka patterns, RPC, API gateway, service discovery, idempotency, API versioning, backpressure, sync vs async. Use when picking how services talk (sync request/response vs async events vs streaming), debugging duplicate writes / lost messages, or designing inter-service contracts.
---

# Communication

How services and clients **talk**. Most cross-service incidents — duplicate writes, lost messages, retry storms, broken contracts — come from the wrong choice here or missing safety patterns on top of it.

## Skills

### Synchronous request/response

| Skill | Description |
|-------|-------------|
| [REST](rest/) | HTTP-verb-shaped resource APIs. The default; great browser/proxy/cache tooling; weak typing without OpenAPI. |
| [gRPC](grpc/) | HTTP/2 + Protobuf RPC. Strongly typed, fast, streaming-capable; not browser-native without grpc-web. |
| [GraphQL](graphql/) | Single endpoint, client-specified shape. Good for many-client BFF-shaped APIs; pays in caching/observability complexity. |
| [RPC](rpc/) | The general pattern. Protobuf, Thrift, Cap'n Proto, JSON-RPC. Trade-offs vs REST. |

### Server push / streaming

| Skill | Description |
|-------|-------------|
| [WebSockets](websockets/) | Full-duplex over TCP. Live chat, collaborative editing, gaming. Sticky sessions; reconnect strategy. |
| [Server-Sent Events (SSE)](sse/) | Server → client one-way streaming over HTTP. Simpler than WebSockets when you don't need client → server frames. |
| [Long Polling](long-polling/) | Hold the request open until something happens. Ancient but cheap; falls back when WebSockets don't. |

### Asynchronous

| Skill | Description |
|-------|-------------|
| [Sync vs Async](sync-vs-async/) | When async is worth its complexity tax. Decision tree. |
| [Webhooks](webhooks/) | "Call me back when X happens" over HTTP. Receiver-side reliability is on the receiver. |
| [Message Queues](message-queues/) | RabbitMQ, SQS, Azure Service Bus. Point-to-point work distribution; ack-based. |
| [Pub/Sub](pub-sub/) | One-to-many fanout. SNS, GCP Pub/Sub, Redis Pub/Sub. |
| [Kafka Patterns](kafka-patterns/) | Topics, partitions, consumer groups, exactly-once semantics, log compaction, transactional writes. |
| [Backpressure](backpressure/) | Stop fast producers from drowning slow consumers. Rx, queues with depth limits, credit-based flow control. |

### Cross-cutting

| Skill | Description |
|-------|-------------|
| [API Gateway](api-gateway/) | Edge tier — auth, rate limit, routing, request shaping. AWS API GW, Kong, Envoy, Nginx. |
| [Service Discovery](service-discovery/) | DNS, Consul, etcd, Kubernetes Services. Client-side vs server-side. |
| [Idempotency](idempotency/) | Make a retry safe. Idempotency keys, conditional writes, deduplication windows. |
| [API Versioning](api-versioning/) | URL-based, header-based, additive evolution. Rollouts, deprecation, contract testing. |

## Decision Trees

### Pick a sync style

| Constraint | Use |
|------------|-----|
| Public API for browsers and partners | REST + OpenAPI |
| Internal service-to-service, low latency, strict types | gRPC |
| Aggregating many backends for many clients | GraphQL (BFF) |
| One method, no dependencies | REST (don't over-engineer) |

### Pick async vs sync

| Need | Use |
|------|-----|
| User waits for the result | Sync (REST/gRPC) |
| Side-effect can be deferred | Async (queue/pub-sub) |
| Buffer load spikes | Async (queue) |
| Multiple consumers per event | Pub/Sub or Kafka |
| Strict ordering per key | Kafka with key-based partitioning |
| Work per item, retries, DLQ | Queue (SQS/RabbitMQ) |
| Event sourcing / CDC / log-as-truth | Kafka |
| Cross-org notifications | Webhooks |

### Pick push to client

| Need | Use |
|------|-----|
| Client → server frames + server → client frames | WebSocket |
| Server → client only, simple | SSE |
| Compatibility with old proxies | Long polling |

## Universal Safety Patterns

When two systems talk over a network, the network will fail. Add these to **every** call that mutates state:

1. **Idempotency** — make the second copy of the same request a no-op.
2. **Timeout** — every wait has a finite ceiling.
3. **Retry with exponential backoff + jitter** — and a bounded retry budget.
4. **Circuit breaker** — fail fast when downstream is unhealthy.
5. **Observability** — request ID, span, timestamp, status.
6. **Schema contract** — versioned, machine-checked.

These all live in `reliability/` and `code-design/` — communication choice picks the wire; reliability picks how it survives.

## Rules of Thumb

- **REST until you can articulate why not.** Most teams reach for GraphQL/gRPC prematurely.
- **Sync unless you can articulate why async.** Async = harder to debug, eventual consistency, ordering bugs. Use it when you have to.
- **Idempotency is non-negotiable for any mutating endpoint.** Networks retry; clients retry; queues redeliver.
- **Don't dual-write.** "Write to DB and publish to Kafka" without an outbox/CDC is the most common distributed-systems bug.
- **Version on day 1.** A single-version API ages into a breaking change every release.

## See Also

- `architecture-patterns/event-driven/`, `architecture-patterns/microservices/`
- `data-systems/event-logs/`, `data-systems/outbox/`, `data-systems/cdc/`
- `reliability/retries-backoff/`, `reliability/circuit-breaker/`, `reliability/timeouts/`
- `security/authn/`, `security/mtls/`

## References

- Fielding, *Architectural Styles and the Design of Network-based Software Architectures* (REST dissertation) — https://ics.uci.edu/~fielding/pubs/dissertation/top.htm
- gRPC docs — https://grpc.io/docs/
- GraphQL spec — https://spec.graphql.org/
- Kafka docs — https://kafka.apache.org/documentation/
- AWS Builders' Library, *Avoiding fallback in distributed systems* — https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/
