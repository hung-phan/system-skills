---
name: interview-templates
description: Index of system-design interview templates — step-by-step framework + worked end-to-end designs (URL shortener, newsfeed, chat, typeahead, web crawler, notifications, rate limiter, distributed cache, distributed file storage, video streaming, ride sharing, payments, distributed counter, recommendation, live comments, geo-spatial search, distributed job scheduler, leaderboard, distributed ID generator, email service, ad-click aggregation, realtime gaming, stock exchange). Use to learn or run the interview format, or to mine a worked design for patterns applicable to a real system.
---

# Interview Templates

End-to-end worked designs in the standard interview format. Even outside an interview, these are the densest applied use of every other skill in the library — read one to see how patterns compose.

## How to Use

1. **For interview prep**: Walk through `framework/` first, then pick three designs spanning storage / streaming / compute and run them out loud.
2. **For real work**: Skim the design closest to your problem; the worked decisions surface trade-offs you'll otherwise discover the hard way.
3. **For learning**: Each template cites the upstream skills (cache, partitioning, idempotency…). Click through to deepen.

## The Framework

| Step | What | Skill |
|------|------|-------|
| 1 | Clarify scope (functional + non-functional reqs) | [framework/](framework/) |
| 2 | Estimate scale (QPS, storage, bandwidth) | `performance/back-of-envelope/` |
| 3 | High-level design (boxes + arrows) | `architecture-patterns/` |
| 4 | Drill into data model + storage | `data-systems/` |
| 5 | Drill into APIs + communication | `communication/` |
| 6 | Identify bottlenecks; add caching, replication, partitioning | `performance/`, `data-systems/` |
| 7 | Identify failures; add reliability patterns | `reliability/` |
| 8 | Wrap up — what would you build first vs later | judgment |

## Skills (Worked Designs)

### Foundational

| Design | What it teaches |
|--------|-----------------|
| [Framework](framework/) | The 8-step interview framework itself. |
| [URL Shortener (TinyURL)](url-shortener/) | Hash-vs-counter, base62, KV store, cache, custom URLs, analytics. |
| [Distributed ID Generator](distributed-id-generator/) | Snowflake, leaf, UUIDs. Time, sequence, region bits. |
| [Rate Limiter](rate-limiter/) | Token bucket, leaky bucket, sliding window. Centralized vs distributed counters. |
| [Distributed Counter](distributed-counter/) | Sharded counters. Eventual consistency vs HLL approximation. |
| [Leaderboard](leaderboard/) | Sorted sets in Redis. Top-K. Real-time updates. |

### Social / content

| Design | What it teaches |
|--------|-----------------|
| [Newsfeed (Twitter/Facebook)](newsfeed/) | Fan-out on write vs on read. Hybrid for celebrities. Push-vs-pull. |
| [Chat System (WhatsApp/Slack)](chat-system/) | WebSockets at scale, message ordering, presence, history retention. |
| [Search Typeahead](search-typeahead/) | Trie + ranking. Personalization. Update cadence. |
| [Recommendation System](recommendation-system/) | Candidate generation + ranking. Embeddings, collaborative filtering. |
| [Live Comments](live-comments/) | Pub/sub at scale, hot-stream celebrity events, moderation. |
| [Notification System](notification-system/) | Multi-channel (push/SMS/email). Retries. User preference store. |
| [Email Service](email-service/) | SMTP, deliverability, bounces, suppression lists, throttling. |

### Storage / streaming

| Design | What it teaches |
|--------|-----------------|
| [Distributed Cache](distributed-cache/) | Memcached / Redis Cluster. Consistent hashing. Cache stampede. |
| [Distributed File Storage (Dropbox/Drive)](distributed-file-storage/) | Chunking, dedup, sync, conflict resolution. |
| [Video Streaming (YouTube/Netflix)](video-streaming/) | CDN, adaptive bitrate, transcoding pipeline, recommendations. |
| [Web Crawler](web-crawler/) | Frontier, politeness, dedup, robots.txt, fault tolerance. |

### Geo / mobility

| Design | What it teaches |
|--------|-----------------|
| [Geo-Spatial Search (Yelp)](geo-spatial-search/) | Geohash, S2, R-tree. Range queries. Hot region rebalancing. |
| [Ride Sharing (Uber)](ride-sharing/) | Real-time matching, dispatching, ETA, surge pricing. |

### Money / events

| Design | What it teaches |
|--------|-----------------|
| [Payment System](payment-system/) | Idempotency, double-entry ledger, reconciliation, Saga. |
| [Stock Exchange / Matching Engine](stock-exchange/) | In-memory order book, deterministic execution, sequencer. |
| [Ad Click Aggregation](ad-click-aggregation/) | Lambda/Kappa stream pipeline, exactly-once, late events. |
| [Realtime Gaming](realtime-gaming/) | UDP vs TCP, lockstep vs prediction, regional servers. |
| [Distributed Job Scheduler](distributed-job-scheduler/) | Cron at scale, leader election, retries, exactly-once. |

## How to Read a Worked Design

Every worked design follows the same structure:

```markdown
# <System>

## Why This Exists / Real-World Flavor

## Step 1 — Requirements
- Functional
- Non-functional (scale, latency, consistency)

## Step 2 — Capacity Estimation
- QPS, storage, bandwidth — back-of-envelope

## Step 3 — High-Level Design
- Boxes-and-arrows (Mermaid diagram)

## Step 4 — Data Model
- Table sketches; partition key choice; index plan

## Step 5 — APIs
- REST/gRPC sketch with request/response shapes

## Step 6 — Deep Dive: <The Hard Part>
- The 1-2 things this design genuinely turns on (e.g. fan-out for newsfeed)

## Step 7 — Bottlenecks & Scale
- Where it breaks at 10× / 100×; the fix

## Step 8 — Reliability & Failure Modes
- What breaks; what to monitor; circuit-breaker / retry placement

## Trade-offs

## What I'd build first vs later

## See Also
```

## Rules of Thumb

- **Clarify before solving.** Half the points are won in the requirements step.
- **Estimate, even roughly.** A "10K QPS" answer constrains 80% of subsequent decisions.
- **Pick the boring storage first.** Postgres, Redis, S3, Kafka. Reach for exotic stores only when explicit constraints force it.
- **Surface trade-offs, don't hide them.** Saying "this is what I'd give up" demonstrates seniority.
- **Reach for the simplest design that meets the stated load.** The interviewer scales it up after you build it.
- **One bottleneck at a time.** Don't pre-shard, pre-cache, pre-replicate everything. Solve the one that hurts first.

## See Also

- `architecture-patterns/INDEX.md` — to pick the shape of the system
- `data-systems/INDEX.md` — to pick storage
- `communication/INDEX.md` — to wire it
- `reliability/INDEX.md` — to keep it alive
- `performance/INDEX.md` — to make it fast

## References

- Xu & Lam, *System Design Interview, vol. 1 & 2* (ByteByteGo) — https://bytebytego.com/
- Donne Martin, *system-design-primer* — https://github.com/donnemartin/system-design-primer
- High Scalability — http://highscalability.com/
- AWS Builders' Library — https://aws.amazon.com/builders-library/
- Google Cloud Architecture Center — https://cloud.google.com/architecture
