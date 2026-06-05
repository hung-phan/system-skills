# Acquire System Skill — Maintainer Guideline

> **Status**: Repo guideline for maintainers of the `system-skills` plugin. Not a runtime skill — read this before adding or editing a single SKILL.md.

Create a new skill or update one existing skill in the `system-skills` plugin library.

## When to Use This vs. refine-system-skill

| Situation | Use |
|-----------|-----|
| New topic, no existing coverage | `acquire-system-skill` (this doc) |
| One file is wrong / outdated, fix is local | `acquire-system-skill` (this doc) |
| Topic spans 2+ folders (e.g. consistency touches replication/, transactions/, consensus/) | [`refine-system-skill.md`](refine-system-skill.md) |
| Need primary-source research before writing | [`refine-system-skill.md`](refine-system-skill.md) (Phase 3 calls deep-research) |
| Major paradigm shift, want to audit coverage | [`refine-system-skill.md`](refine-system-skill.md) |

If you start under this guideline and discover the change actually touches multiple files, **stop and switch to [`refine-system-skill.md`](refine-system-skill.md)** — don't try to coordinate cross-file updates from here.

## Phase 0 — Intake (do this BEFORE research)

Don't start researching or writing on a one-line request. Most quality problems trace back to skipping this step. Ask the user the smallest set of questions that resolve real ambiguity — usually 2-4, never more. Skip any question whose answer is already obvious from the request.

Required intake before proceeding:

| Question | Why it matters |
|----------|---------------|
| **What's the topic, in one sentence?** | Forces the user to commit to a scope. "Add caching" is not a scope; "Add a skill on cache-aside vs write-through for read-heavy hot keys" is. |
| **Add new, or update existing?** | If updating, which file path? If adding, what's the proposed folder placement? |
| **What's the audience problem?** | What will the reader be trying to do when they reach this skill? "Pick between Redis and Memcached" is a different skill than "Implement consistent hashing in Go". |
| **Anchor sources?** | Any specific paper, book chapter, or doc the user wants treated as canonical (DDIA, SRE Book, original Raft paper, AWS Builders' Library)? Avoids ranking-roulette during research. |
| **Out-of-scope items?** | What should this skill explicitly NOT cover? (e.g. "skip the in-memory cache patterns — that's a separate skill"). Prevents scope creep mid-write. |
| **Existing skills to cross-reference?** | If the user already knows the related neighbors, capture them now — they become "See also" links instead of duplicated content. |

Stop asking once the answers are enough to start drafting. If the user says "you decide", record the assumption explicitly in the skill's first draft and flag it in your reply so they can correct it before merge.

## Critical Thinking, Not Template-Filling

This is a judgment skill, not a form. Every step below has a decision behind it:

- **Placement**: Which folder? Default rules: data plane / state → `data-systems/`; production-runtime concerns → `reliability/`; wire format → `communication/`; multi-system pattern → `architecture-patterns/`; code structure → `code-design/`.
- **Trigger phrases in `description:`**: What would a user actually type? Generic descriptions never match. Include real symptoms ("p99 latency spike", "duplicate charges", "stale reads after write").
- **`Why This Exists`**: What problem does this solve that the next-best alternative doesn't? If you can't articulate the alternative, you don't yet understand the topic well enough.
- **Decision table**: Where does this fail? When is it the wrong choice? A skill that only says when to use itself is half-written.
- **References**: Are these primary sources (papers, RFCs, vendor docs, the SRE book), or is this a Medium post quoting a Medium post?

If any answer comes out as "I don't know", stop and research — don't write through the gap.

## Repository Structure

```
skills/
├── system-router/                 (top-level router — entry point)
├── architecture-patterns/         (monolith, microservices, event-driven, CQRS,
│                                   event sourcing, hexagonal, layered, clean,
│                                   strangler fig, saga, BFF, service mesh,
│                                   sidecar, pipes-filters, lambda, kappa)
├── data-systems/                  (relational, document, KV, wide-column, vector,
│                                   time-series, search, graph, indexing,
│                                   replication, partitioning, consistency,
│                                   CAP/PACELC, distributed transactions,
│                                   consensus, event logs, stream/batch,
│                                   OLTP/OLAP, warehouses, lakehouse, CDC, outbox)
├── communication/                 (REST, gRPC, GraphQL, WebSockets, SSE,
│                                   webhooks, queues, pub/sub, Kafka patterns,
│                                   API gateway, service discovery,
│                                   idempotency, versioning, backpressure)
├── reliability/                   (SLO/SLI/SLA, retries, circuit breaker,
│                                   bulkhead, rate limit, load shed, timeouts,
│                                   health checks, graceful degradation, chaos,
│                                   observability, incident response, postmortems,
│                                   capacity, deployment, DR)
├── code-design/                   (SOLID, DRY/KISS/YAGNI, coupling/cohesion,
│                                   smells, refactoring catalog, GoF patterns,
│                                   DDD, TDD/BDD, testing pyramid, clean code,
│                                   pragmatic programmer, code review)
├── security/                      (threat modeling, authn, authz, encryption,
│                                   secrets, OWASP, zero-trust, mTLS,
│                                   defense in depth, audit, compliance)
├── performance/                   (latency vs throughput, USE/RED, profiling,
│                                   caching, CDN, connection pooling, batching,
│                                   indexes, compression, capacity, tail latency,
│                                   BOTE)
└── interview-templates/           (framework + worked designs)
```

The two meta-guidelines (`acquire-system-skill.md`, `refine-system-skill.md`) live in `docs/` — not under `skills/` — because they're for repo maintainers and would otherwise burn context tokens for end users every session.

## Quality Standards

### 1. Problem-First (most important)

Lead with **why this exists**, not code:

```markdown
## Why This Exists

**Problem**: [What breaks without this? What pain does this solve?]

**Key insight**: [The core idea in plain English — one sentence]

**Reach for this when**: [Decision criteria vs alternatives]

**Don't reach for this when**: [Where the simpler alternative wins]
```

### 2. SKILL.md Format

```markdown
---
name: skill-name
description: What it does. Use when [specific symptom-level triggers].
---

# Title

## Why This Exists

## [Core sections — diagrams, code, sequence flows]

## Trade-offs

## Common Pitfalls

## Decision Table

## References

## See Also
```

### 3. Real, Verified Links

Every URL must return HTTP 200: `curl -sI "URL" | head -1`.

Prefer **primary sources**:
- Original papers (arXiv / Google Research / Microsoft Research / VLDB / SOSP)
- RFCs (https://www.rfc-editor.org/rfc/rfcXXXX)
- The SRE Book + Workbook (free at https://sre.google/books/)
- AWS Builders' Library, Google Cloud Architecture Center, Azure patterns
- Vendor docs (Postgres, Kafka, Redis, etcd)
- DDIA chapters
- Authoritative blogs (Pat Helland, Martin Fowler, "morning paper")

### 4. Diagrams

System design without diagrams is unreadable. Every architectural skill should include at least one Mermaid diagram. Common shapes:

- Request flow → `sequenceDiagram`
- Architecture → `flowchart` / `graph`
- State machines → `stateDiagram-v2`

### 5. Decision Table

```markdown
| Scenario | Use This | Use Alternative | Why |
|----------|----------|-----------------|-----|
| ≤ 1 team, simple domain | Monolith | — | Ops cost of microservices is unjustified |
| Many teams, polyglot | Microservices | Modular monolith | Independent deploys |
```

### 6. Index Files

Each folder has its own `SKILL.md` listing children in a table. Keep index files as concise routers.

### 7. Trade-off Honesty

```markdown
## Trade-offs

| Benefit | Cost |
|---------|------|
| Independent deploys | Network calls between services (latency + partial failures) |
| Tech freedom per service | Polyglot ops burden |
| Smaller blast radius per change | Distributed-systems debugging |
```

### 8. "Don't Do This"

Most engineers reach for fancy patterns when a simpler tool would work. Each skill should say where the simpler tool wins:

- "Use a monolith until 3+ teams can't ship independently."
- "Don't reach for Kafka if you have one producer and one consumer; a queue is enough."
- "Don't shard before you've added a read replica and a cache."

---

## Workflow: Adding a New Skill

1. **Decide placement**.
2. **Research** primary sources.
3. **Write `skills/<folder>/<topic>/SKILL.md`**:
   - [ ] YAML frontmatter with `name` and `description` (symptom triggers)
   - [ ] `## Why This Exists` (Problem / Key insight / Reach when / Don't reach when)
   - [ ] At least one Mermaid diagram (or strong reason none is needed)
   - [ ] Code or pseudocode example
   - [ ] `## Trade-offs` table
   - [ ] `## Common Pitfalls` list
   - [ ] `## Decision Table` vs alternatives
   - [ ] `## References` with verified links
   - [ ] `## See Also`
4. **Update parent index**.
5. **Verify links**.

## Workflow: Updating an Existing Skill

1. Read the current file in full first.
2. Identify which quality standards are missing.
3. Common gaps: no diagram, no trade-off table, no "Don't reach for this when", outdated tool versions, broken links.
4. Add missing sections without removing existing content unless explicitly replacing stale info.

## Anti-Patterns

- ❌ Pattern-as-fashion (presenting a pattern as universally good)
- ❌ Diagram-less prose for architectural topics
- ❌ Code dumps without "why"
- ❌ Links without verifying 200
- ❌ Frontmatter `description` without symptom triggers
- ❌ Missing `## Why This Exists`
- ❌ Hand-waving past failure modes
- ❌ Hardcoded absolute paths
