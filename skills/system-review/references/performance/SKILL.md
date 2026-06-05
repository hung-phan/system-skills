---
name: performance
description: Index of performance skills — latency vs throughput, USE/RED methods, profiling, caching (cache-aside / write-through / write-back / refresh-ahead), CDN, connection pooling, batching & pipelining, indexes & query optimization, compression, capacity & load modeling, tail latency, back-of-envelope estimation. Use when hitting a latency or throughput target, picking a caching strategy, debugging tail latency, or estimating capacity for a new system.
---

# Performance

How to make a system **fast and dense**. Performance work is investigative — measure, find the bottleneck, fix the bottleneck, re-measure. Code-level micro-optimizations almost never matter; the bottleneck is usually somewhere else.

## Skills

### Mental models

| Skill | Description |
|-------|-------------|
| [Latency vs Throughput](latency-vs-throughput/) | Two axes that trade against each other. Little's Law. Average vs p99. |
| [Tail Latency](tail-latency/) | Why p99 ≠ average × 100. Dean's "tail at scale." Hedged requests, head-of-line blocking. |
| [USE / RED Methods](use-red-methods/) | USE for resources (utilization/saturation/errors). RED for services (rate/errors/duration). |
| [Back-of-Envelope](back-of-envelope/) | Quick capacity math. Numbers every engineer should know (latency table). |
| [Capacity Modeling](capacity-modeling/) | Forecast headroom. Load testing, growth curves, safety factor. |

### Profiling

| Skill | Description |
|-------|-------------|
| [Profiling](profiling/) | CPU / heap / lock / async profilers. Flame graphs. Differential. perf, py-spy, async-profiler, pprof. |
| [Tracing](tracing/) | Distributed traces (OpenTelemetry). Span fan-out, critical path. |

### Caching

| Skill | Description |
|-------|-------------|
| [Caching Strategies](caching/) | Cache-aside, read-through, write-through, write-back, refresh-ahead. TTL vs invalidation. Cache stampede. |
| [CDN](cdn/) | Edge caching for static + dynamic content. Cache keys, invalidation, geo-routing. |
| [Cache Topologies](cache-topologies/) | Local + remote (L1/L2). Sidecar caches. Sharded caches (Redis Cluster, Memcached consistent hashing). |

### Throughput patterns

| Skill | Description |
|-------|-------------|
| [Connection Pooling](connection-pooling/) | DB / HTTP pools. Sizing. Connection storms. |
| [Batching](batching/) | Group N operations into one round-trip. Latency vs throughput trade. Window size. |
| [Pipelining](pipelining/) | Send N requests without awaiting each. Redis pipelining, gRPC streaming. |
| [Compression](compression/) | gzip vs Brotli vs zstd vs LZ4. CPU vs bytes. When compression *hurts*. |

### Storage performance

| Skill | Description |
|-------|-------------|
| [Indexes & Query Optimization](indexes-query-optimization/) | EXPLAIN plans, covering indexes, partial indexes, query rewrites. |
| [Database Tuning](database-tuning/) | shared_buffers, work_mem, vacuum, autovacuum, checkpoint tuning (Postgres). InnoDB buffer pool (MySQL). |
| [Hot Path Optimization](hot-path-optimization/) | Optimizing the 1-3% of code that handles 99% of traffic. Profile-guided. |

## Investigation Workflow

```
1. MEASURE     What's slow? Where? p50, p95, p99.
                  └─ RED for services, USE for resources
2. PROFILE     Where does the time go inside the slow request?
                  └─ Flame graphs; differential profiling vs baseline
3. THEORIZE    What might cause this?
                  └─ Latency table (CPU vs memory vs disk vs net) bounds the search
4. FIX         Smallest change that resolves it.
5. VERIFY      Re-measure. Did p99 actually move? Did anything else regress?
6. REPEAT
```

## Decision Trees

### "Where do I add a cache?"

| Bottleneck | Layer |
|------------|-------|
| Static assets, public | CDN (edge) |
| Per-user computed view | Server-side cache (Redis) |
| Frequently re-read row | DB read replica + query cache, or app cache |
| Aggregated metric | Materialized view + periodic refresh |
| Authn/authz decisions | In-process LRU with short TTL |
| Third-party API call | App cache with stale-while-revalidate |

### "Pick a caching strategy"

| Workload | Strategy |
|----------|----------|
| Read-heavy, miss-acceptable, simple | Cache-aside |
| Read-heavy, miss-unacceptable | Read-through (cache fills synchronously on miss) |
| Write rare, read hot | Write-through |
| Write-heavy, eventual consistency ok | Write-back |
| Predictable hot keys, no miss penalty | Refresh-ahead |

### "I'm CPU-bound. What now?"

```
Is the CPU work redundant?
├── yes → cache the result
└── no  → can it parallelize?
            ├── yes → fan out to workers
            └── no  → can it move off the request path?
                        ├── yes → async (queue)
                        └── no  → vertical scale or rewrite hot path
```

## Numbers Every Engineer Should Know

(Jeff Dean's latency table, modernized)

| Operation | Time |
|-----------|------|
| L1 cache reference | ~1 ns |
| L2 cache reference | ~4 ns |
| Branch misprediction | ~3 ns |
| Mutex lock/unlock | ~17 ns |
| Main memory reference | ~100 ns |
| Compress 1KB with Snappy | ~2 µs |
| Send 2KB over 1 Gbps network | ~20 µs |
| Read 1MB from main memory | ~250 µs |
| Round trip in same DC | ~500 µs |
| Read 1MB from SSD | ~1 ms |
| Disk seek (HDD) | ~10 ms |
| Read 1MB from disk (HDD) | ~20 ms |
| Send packet CA → NL → CA | ~150 ms |
| TLS handshake (~3 RTTs at 30 ms each) | ~90 ms |

## Rules of Thumb

- **Measure before you optimize.** Otherwise you're guessing — usually wrong.
- **Optimize the bottleneck**, not the function you find easy to read.
- **The fastest way to do something is not to do it.** Cache, deduplicate, defer, batch.
- **Disk is slow, network is slower.** Memory hierarchy bound dominates most workloads.
- **Tail dominates user experience.** A p99 of 2s is 1 in 100 users having a 2s page; that's a lot of users.
- **Throughput and latency trade.** Batching trades latency for throughput; backpressure trades the opposite.
- **Don't cache writes**, don't cache personalized content at the CDN, and don't cache without an invalidation story.
- **A pool of N connections is N concurrent requests**, not "as many as you want". Plan accordingly.

## See Also

- `../data-systems/indexing/`, `../data-systems/replication/`, `../data-systems/partitioning/`
- `../reliability/load-shedding/`, `../reliability/rate-limiting/`, `../reliability/capacity-planning/`
- `../communication/backpressure/`, `../communication/api-gateway/`
- `../architecture-patterns/cqrs/`, `../architecture-patterns/event-driven/`

## References

- Dean, *Numbers Everyone Should Know* — https://research.google/pubs/pub44875/
- Dean & Barroso, *The Tail at Scale* — https://research.google/pubs/pub40801/
- Brendan Gregg, *Systems Performance* — https://www.brendangregg.com/sysperfbook.html
- Brendan Gregg's USE method — https://www.brendangregg.com/usemethod.html
- Tom Wilkie's RED method — https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/
- AWS Builders' Library, *Caching challenges and strategies* — https://aws.amazon.com/builders-library/caching-challenges-and-strategies/
