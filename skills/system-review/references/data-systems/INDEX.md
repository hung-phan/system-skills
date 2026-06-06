---
name: data-systems
description: Index of data-system skills — relational vs document vs wide-column vs KV vs vector vs time-series vs graph vs search; storage engines (B-tree, LSM, hash); replication (sync/async/leaderless); partitioning/sharding; consistency models; CAP/PACELC; distributed transactions; consensus (Raft, Paxos); event logs (Kafka); stream/batch processing; OLTP vs OLAP; warehouses/lakehouse; CDC; outbox; schema evolution. Use when picking a database, deciding replication or sharding strategy, choosing a consistency model, or debugging data-level problems (hot partition, stale reads, slow queries).
---

# Data Systems

Pick storage, replication, partitioning, and consistency. Most production incidents — duplicate writes, stale reads, hot shards, runaway disk — trace back to a decision in this folder.

## Skills

### Storage models

| Skill | Description |
|-------|-------------|
| [Relational](relational/) | Postgres / MySQL / SQL Server. ACID, joins, mature ops, the right default for OLTP. |
| [Document DB](document-db/) | MongoDB, DocumentDB, CouchDB. Schema flux, hierarchical documents, single-aggregate access. |
| [Key-Value](key-value/) | Redis, Memcached, DynamoDB-on-keys. Sub-ms latency, simple ops on a single key. |
| [Wide-Column](wide-column/) | Cassandra, ScyllaDB, Bigtable, HBase. Massive write throughput, time-series, predictable scale. |
| [Search Engine](search-engine/) | Elasticsearch / OpenSearch / Solr / Meilisearch. Inverted index, full-text, faceted filtering. |
| [Graph DB](graph-db/) | Neo4j, JanusGraph, Memgraph. Multi-hop traversal, social/identity/recommendation graphs. |
| [Vector DB](vector-db/) | pgvector, Qdrant, Weaviate, Pinecone, Milvus. Embedding similarity for RAG and semantic search. |
| [Time-Series DB](time-series-db/) | Prometheus, InfluxDB, TimescaleDB, VictoriaMetrics. Timestamped writes, range queries, downsampling. |
| [OLAP / Warehouse](olap-warehouse/) | BigQuery, Snowflake, Redshift, ClickHouse, DuckDB. Columnar, analytical queries over billions of rows. |
| [Lakehouse](lakehouse/) | Delta Lake, Iceberg, Hudi. ACID over object storage; SQL on Parquet without copying it. |
| [Event Logs](event-logs/) | Kafka, Pulsar, Redpanda. Append-only ordered log; topic = unbounded queue + retention. |
| [NoSQL Overview](nosql-overview/) | When and why NoSQL — vs the relational default. The decision tree. |

### Storage engines

| Skill | Description |
|-------|-------------|
| [Indexing](indexing/) | B-tree vs LSM vs hash. How indexes speed reads (and slow writes). Covering indexes. |
| [Storage Engines](storage-engines/) | InnoDB, RocksDB, WiredTiger, MyRocks, Cassandra's storage. Page-oriented vs log-structured. |

### Distribution

| Skill | Description |
|-------|-------------|
| [Replication](replication/) | Single-leader, multi-leader, leaderless. Sync vs async. Lag, failover, split brain. |
| [Partitioning](partitioning/) | Range vs hash vs directory. Hot partitions, rebalancing, secondary indexes across shards. |
| [Consistency Models](consistency-models/) | Linearizable, sequential, causal, monotonic-read, eventual — the formal hierarchy. |
| [CAP and PACELC](cap-pacelc/) | What you actually give up under partition. PACELC adds latency/consistency in normal operation. |
| [Distributed Transactions](distributed-transactions/) | 2PC, 3PC, percolator, Spanner-style. When to choose 2PC vs Sagas. |
| [Consensus](consensus/) | Paxos, Multi-Paxos, Raft, EPaxos. The algorithm under etcd, Consul, Spanner, Kafka KRaft. |
| [CRDTs](crdts/) | Conflict-free replicated data types. State and op-based. Where eventual ≠ destruction. |
| [Schema Evolution](schema-evolution/) | Backward / forward / full compatibility. Avro / Protobuf rules. Online migrations. |

### Probabilistic & skew-aware techniques

| Skill | Description |
|-------|-------------|
| [Bloom Filter](bloom-filter/) | Bloom, counting Bloom, cuckoo, xor filters. Probabilistic set membership; LSM read path; reduce-side joins; cache-miss filtering. |
| [Data Skew](data-skew/) | Salting, secondary sort, broadcast joins, AQE. When 1 task runs for 2 hours and 999 finish in 30s. |

### Workload patterns

| Skill | Description |
|-------|-------------|
| [OLTP vs OLAP](oltp-vs-olap/) | Transactional vs analytical. Why one DB rarely serves both well. |
| [Stream Processing](stream-processing/) | Kafka Streams, Flink, Spark Structured Streaming. Windowing, watermarks, exactly-once. |
| [Batch Processing](batch-processing/) | Spark, MapReduce, Hadoop. Idempotent jobs, partitioned input/output, retries. |
| [CDC (Change Data Capture)](cdc/) | Debezium, logical replication, Maxwell. Stream DB changes into Kafka without dual-write. |
| [Outbox Pattern](outbox/) | Atomic publish-with-write via a side table + relay. Solves the dual-write problem. |
| [Materialized Views](materialized-views/) | Precompute denormalized read models from authoritative writes; refresh strategies. |

## Decision Trees

### Pick a database

| Workload | Use |
|----------|-----|
| OLTP, joins, ACID, ad-hoc queries | Relational (Postgres) |
| OLTP, document-shaped aggregates, schema flux | Document (MongoDB) |
| Massive writes, predictable queries (time-series, IoT, audit) | Wide-column (Cassandra/ScyllaDB) |
| Sub-ms KV / cache / leaderboard / session | Key-Value (Redis / Memcached) |
| Full-text + filtering / faceted search | Search engine (Elasticsearch / OpenSearch) |
| Multi-hop relationships (fraud, social, identity) | Graph (Neo4j) |
| Embedding similarity (RAG, semantic search) | Vector (pgvector / Qdrant / Pinecone) |
| Metrics, time-series at high cardinality | Prometheus / TimescaleDB / InfluxDB |
| Analytical queries on TB+ | OLAP (BigQuery / Snowflake / ClickHouse / DuckDB) |
| Lakehouse pattern over S3/GCS | Iceberg / Delta / Hudi |
| Append-only event log + multiple consumers | Kafka / Pulsar |

### Replicate or shard?

| Symptom | Try first | Then |
|---------|-----------|------|
| DB CPU saturated on **reads** | Add a read replica | Add cache; route reads by route |
| DB CPU saturated on **writes** | Optimize indexes, batch | Partition (shard) |
| DB exceeds disk on a single host | Archive cold data | Partition |
| Need cross-region availability | Multi-region replica | Multi-leader (with conflict policy) or Spanner-class DB |
| Single-leader is the bottleneck | Move read traffic off | Multi-leader or leaderless |

### Pick a consistency model

| Need | Use |
|------|-----|
| "Show me the last write I made" | Read-your-writes |
| "Don't show me older data than I just saw" | Monotonic reads |
| Cause-effect ordering across users | Causal consistency |
| "Looks like a single node" semantics | Linearizability (consensus) |
| Multi-row atomicity within one DB | Serializable isolation (single DB) |
| Multi-service atomicity | Saga (compensations), not 2PC |

## Rules of Thumb (DDIA-distilled)

- **Postgres until proven otherwise.** Most teams reach for NoSQL too early. JSONB + partial indexes cover most "we need flexibility" claims.
- **Don't shard before you've added a read replica + cache.** Sharding is the most expensive scaling step; it constrains every future query.
- **Replication is async by default.** If you need sync, you need consensus. Don't pretend a single async follower is "almost sync".
- **Two writes ≠ one transaction.** The dual-write problem (write to DB + publish to Kafka) eats teams alive. Use the **outbox pattern** or CDC.
- **The log is the database.** Event logs (Kafka) + materialized views (Postgres / Elasticsearch) is a powerful pattern; CQRS makes it explicit.
- **Schema migrations are deploys.** Backward-compatible writes first, then deploy readers, then enforce.

## See Also

- `architecture-patterns/cqrs/`, `architecture-patterns/event-sourcing/`, `architecture-patterns/lambda-architecture/`, `architecture-patterns/kappa-architecture/`
- `communication/idempotency/` — what to do once you accept distributed writes can retry
- `reliability/disaster-recovery/`, `reliability/capacity-planning/`
- `performance/caching/`, `performance/indexes-query-optimization/`

## References

- Kleppmann, *Designing Data-Intensive Applications* (DDIA), O'Reilly 2017 — chapters 2-12
- Helland, *Life Beyond Distributed Transactions* — https://queue.acm.org/detail.cfm?id=3025012
- Abadi, *Consistency Tradeoffs in Modern Distributed Database System Design* (PACELC) — https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf
- Brewer, *CAP Twelve Years Later* — https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/
