# System Skills

A Claude Code plugin with a comprehensive system design and software engineering skills library — reference guides with diagrams, code patterns, decision tables, and "why this exists" context for every major system design topic.

Distilled from *Designing Data-Intensive Applications* (Kleppmann), *Site Reliability Engineering* (Beyer et al.), *Refactoring* and *Clean Code* (Fowler / Martin), *Fast Data Architectures for Streaming Applications* (Wampler), and the *System Design Interview* canon (Xu / Lam) — plus production patterns from real-world distributed systems.

**Author**: [hung-phan](https://github.com/hung-phan)

## Install

Register as a plugin marketplace, then install:

```
/plugin marketplace add hung-phan/system-skills
/plugin install system-skills@system-skills
```

### Local install (for development)

```bash
claude plugin marketplace add ./
claude plugin install system-skills@system-skills
```

## How to Use

The plugin exposes **one** slash command: `/system-review`. It is the front door for everything in this library — paste a design and it walks the bundled `references/` catalog (the canonical patterns across 8 domains) to produce cited, prioritized findings.

### Run a review

```
/system-review review my RFC for the new payment service: <paste doc>
/system-review audit this architecture diagram for missing reliability concerns
/system-review score my interview answer for "design Twitter"
/system-review would this design survive 10x traffic on Black Friday?
```

`/system-review` walks the bundled references library, identifies the patterns your design touches, reads the canonical `## Common Pitfalls` and `## Trade-offs` sections, and produces a prioritized list of findings — each cited to a specific reference so you can read the source. Use it for design docs, RFCs, capacity plans, post-mortems, code reviews of pattern implementations, and pre-launch readiness audits.

### Browse the references directly

The references live under `skills/system-review/references/`, grouped by domain. Each is a self-contained markdown file with diagrams, code patterns, decision tables, common pitfalls, and a `## See Also` block linking to adjacent references.

| Domain | What's inside |
|--------|---------------|
| `references/architecture-patterns/` | Monolith, microservices, event-driven, CQRS, event sourcing, hexagonal, saga, BFF, service mesh, sidecar, lambda, kappa, … |
| `references/data-systems/` | Relational vs NoSQL, indexing, replication, partitioning, consistency models, CAP/PACELC, distributed transactions, consensus, event logs, stream/batch processing, OLTP vs OLAP, lakehouse, CDC, outbox, vector/time-series DBs |
| `references/communication/` | REST, gRPC, GraphQL, WebSockets, SSE, long-polling, webhooks, message queues, pub/sub, Kafka patterns, RPC, API gateway, service discovery, idempotency, API versioning, backpressure |
| `references/reliability/` | SLO/SLI/SLA, error budgets, retries & backoff, circuit breakers, bulkheads, rate limiting, load shedding, timeouts, health checks, graceful degradation, chaos engineering, observability, incident response, postmortems, capacity planning, deployment strategies, disaster recovery |
| `references/code-design/` | SOLID, DRY/KISS/YAGNI, coupling & cohesion, code smells, refactoring catalog, design patterns (GoF), DDD, aggregates, TDD/BDD, testing pyramid, clean code, pragmatic programmer, pair programming & code review |
| `references/security/` | Threat modeling, authn (OAuth2/OIDC/SAML/JWT), authz (RBAC/ABAC), encryption at rest/transit, secrets management, OWASP Top 10, CSRF/XSS/SQLi/SSRF/IDOR, zero-trust, mTLS, defense in depth, audit logging, compliance |
| `references/performance/` | Latency vs throughput, USE/RED methods, profiling, caching topologies, CDN, connection pooling, batching, indexes/query optimization, compression, capacity modeling, tail latency, back-of-envelope estimation |
| `references/interview-templates/` | Framework + worked walkthroughs for URL shortener, Twitter newsfeed, chat, typeahead, web crawler, notifications, rate limiter, distributed cache/file storage/job scheduler/ID generator, video streaming, ride sharing, payments, recommendation, live comments, geo-spatial, leaderboard, email, ad-click, realtime gaming, stock exchange |

### Three usage patterns

1. **You have a design and want feedback** → `/system-review <paste>`. It runs the canonical checklist and cites every finding.
2. **You want the deep treatment of one topic** → ask Claude directly ("explain CQRS tradeoffs", "how does Raft achieve consensus?", "review my code for SOLID violations"). `/system-review` activates and reads the matching reference; or browse to `references/<domain>/<topic>/SKILL.md` yourself.
3. **You want a keyword you don't have a path for** → `grep -rli "<keyword>" skills/system-review/references/` finds every reference that mentions it. Canonical home is the deepest match.

## Repo Layout

```
skills/
└── system-review/
    ├── SKILL.md                    # the only skill exposed to Claude
    └── references/                 # canonical patterns the skill walks
        ├── architecture-patterns/
        ├── data-systems/
        ├── communication/
        ├── reliability/
        ├── code-design/
        ├── security/
        ├── performance/
        └── interview-templates/
docs/                               # maintainer guidelines (not runtime)
.claude-plugin/                     # plugin + marketplace metadata
```

> **Maintainer guidelines** (not runtime skills) live in `docs/`:
> - [`docs/acquire-system-skill.md`](docs/acquire-system-skill.md) — add a new reference or update one file
> - [`docs/refine-system-skill.md`](docs/refine-system-skill.md) — deep-research + propagate updates across multiple references

## Reference Format

Every reference follows a consistent format:
- **Why This Exists** — the problem it solves and when to reach for it
- **Diagrams** — Mermaid diagrams for architecture, sequence, and state
- **Code / pseudocode** — realistic, copy-pasteable patterns
- **Trade-offs & Decision tables** — when to use this vs alternatives, what breaks
- **Common pitfalls** — failure modes, anti-patterns, war stories
- **References** — verified links to canonical sources, books, papers, and battle-tested writeups
- **See Also** — cross-references to adjacent skills

## Philosophy

> *Not all problems need a microservice.*

Software engineering is the discipline of choosing the **simplest** solution that meets the actual constraints — and being able to defend why. Each reference in this library leads with the *problem* before the pattern, surfaces the *cost* of every solution, and explicitly says when **not** to reach for it.

The five questions every reference answers:

1. What problem does this solve?
2. What does it cost (latency, complexity, ops, money)?
3. When does it fail or break down?
4. What's the simpler alternative?
5. When is the simpler alternative wrong?

## Contributing

Maintainer guidelines live in `docs/` (not under `skills/`, so they don't burn context tokens for end users). Read the relevant one before editing:

| Task | Read |
|------|------|
| Add one new reference or lightly edit one file | [`docs/acquire-system-skill.md`](docs/acquire-system-skill.md) |
| Refresh a topic that spans multiple references (e.g. consistency touches replication, transactions, consensus) | [`docs/refine-system-skill.md`](docs/refine-system-skill.md) |
| Audit coverage of an area after a major release / paradigm shift | [`docs/refine-system-skill.md`](docs/refine-system-skill.md) |
| Restructure the reviewer itself | edit `skills/system-review/SKILL.md` directly |

These docs encode the quality standards, folder conventions, intake questions, and formatting rules. Hand them to Claude as the spec for the change you want — e.g. "follow `docs/acquire-system-skill.md` to add a skill on outbox pattern with Postgres + Debezium".
