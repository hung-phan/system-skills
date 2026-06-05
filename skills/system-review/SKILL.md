---
name: system-review
description: Critique a system-design proposal, RFC, architecture diagram, or running design against the canonical patterns bundled in this skill's `references/` library. Reach for this when the user says "review my design", "audit this RFC", "what am I missing in this architecture?", "is this monolith ready to split?", "score my interview answer", "find the gaps before we ship", "would this survive Black Friday?", or pastes a system diagram / capacity plan / data model and asks for feedback. Walks the references library to ground each finding in a canonical source rather than freelancing opinions, and produces a prioritized list of issues with severity, evidence, and the specific reference the maintainer should consult.
---

# System Review

Review a design — proposal, RFC, diagram, code, capacity plan, post-mortem draft, or interview answer — against the canonical patterns in this skill's `references/` library. Each finding cites a specific reference so the maintainer can read the source rather than trust the reviewer's memory.

## Why This Exists

**Problem.** Designs fail in stereotyped ways: skipping non-functional requirements, picking microservices for a 2-team org, mocking idempotency without a key, building a saga without compensations, choosing eventual consistency for a financial ledger, sizing capacity without back-of-envelope math, instrumenting nothing, threat-modeling never. A reviewer who has read DDIA, the SRE book, and a few Pat Helland papers catches most of these in 15 minutes. A reviewer who freelances from memory misses the half they don't recognize and rationalizes the half they do.

**Key insight.** The skill library in this repo *is* the checklist. A review that walks the library — reading the relevant `## Common Pitfalls`, `## Trade-offs`, and `## Decision Table` of every adjacent skill — is structurally more thorough than one driven by recall. The reviewer's job is to (1) match the design to the relevant skills, (2) read those skills' pitfall sections, (3) report concrete gaps with citations, and (4) flag where the design contradicts the library's stated trade-offs.

**Reach for this when:**
- User pastes an RFC, design doc, architecture diagram, or capacity plan and asks for feedback.
- User describes a system in prose ("we're building X with Postgres + Kafka + Redis ...") and asks "what would break?", "what am I missing?", "is this right?".
- User shares a code review for a non-trivial change (saga step, retry policy, cache invalidation, consensus integration) and wants a second opinion.
- User asks to score an interview answer or post-mortem draft against this library's canon.
- User wants a pre-launch readiness audit ("would this survive 10x traffic?", "what reliability work is missing?").

**Don't reach for this when:**
- User wants a *recommendation* for what to build ("should I use Kafka or RabbitMQ?") — that's a routing question, read the matching domain index directly (e.g. `references/communication/SKILL.md`) for the decision table.
- User wants the *deep treatment* of one topic ("explain Raft") — read the canonical reference directly (`references/data-systems/consensus/SKILL.md`), don't run a full review.
- User is asking for an implementation ("write me a circuit breaker in Go") — read the relevant reference and copy from its code section.
- The artifact is too small to review (one function, two boxes on a whiteboard) — answer the question directly.

## How to Run a Review

The reviewer always follows the same five steps. Skipping any of them is the difference between this skill and "vibes-based feedback".

### Step 1 — Classify the artifact and the workload shape

What kind of design is being reviewed?

| Artifact | Treat as |
|----------|----------|
| RFC / design doc | Full system review (steps 2-5 below) |
| Architecture diagram only | Identify components -> load their skills -> critique by adjacency |
| Capacity plan / cost model | `performance/back-of-envelope/` + `performance/capacity-modeling/` lenses |
| Code change implementing a pattern (saga step, circuit breaker, idempotency key) | Load that pattern's skill and check the implementation against its `## Common Pitfalls` |
| Interview answer | Score against `interview-templates/framework/` 8-step rubric |
| Post-mortem draft | Score against `reliability/postmortems/` rubric |
| Migration plan | Cross-check against `architecture-patterns/strangler-fig/` + relevant target-state skills |

What workload shape does the design describe?

- Read-heavy / write-heavy / balanced.
- Synchronous user-facing / async batch / streaming.
- Single-region / multi-region.
- Multi-tenant / single-tenant.
- Latency-critical / throughput-critical / cost-critical.

These shape constraints determine which skills are load-bearing for the review.

### Step 2 — Discover the relevant skills

The library is plain markdown bundled at `references/` next to this SKILL.md. Use the filesystem directly — don't guess paths from memory. From the repo root, the references live at `skills/system-review/references/`; from this skill's own directory, just use `references/`.

```bash
# Set your starting directory (repo root assumed below)
REFS=skills/system-review/references

# What sub-references exist in a domain?
ls $REFS/data-systems/
ls $REFS/reliability/
ls $REFS/architecture-patterns/

# Where is a topic covered? (canonical home is the deepest match)
grep -rli "saga"        $REFS/
grep -rli "idempoten"   $REFS/
grep -rli "consensus\|raft\|paxos" $REFS/
grep -rli "back.?pressure" $REFS/

# One-line summary of every reference (great for picking adjacent ones)
for f in $REFS/*/SKILL.md $REFS/*/*/SKILL.md; do
  echo "$f:"
  grep -m1 "^description:" "$f"
done

# Find every reference that mentions an exact tool/term
grep -rl "Kafka\|Pulsar"      $REFS/
grep -rl "Postgres\|MySQL"    $REFS/
grep -rl "STRIDE\|threat model" $REFS/
```

For each component the design names (e.g. "Kafka", "Redis cache", "matching service", "saga orchestrator"), build a small list of references you'll consult (paths relative to `references/`):

```
Component: Kafka topic partitioned by user_id
  -> communication/kafka-patterns/        (partitioning, ordering, retention)
  -> data-systems/event-logs/             (durability, replay, retention)
  -> data-systems/partitioning/           (hot key risk)
  -> reliability/observability/           (lag, ISR, under-replicated metrics)

Component: Saga across payment + inventory + email
  -> architecture-patterns/saga/          (orchestration vs choreography)
  -> data-systems/distributed-transactions/  (when 2PC actually wins)
  -> data-systems/outbox/                 (publish-with-DB-tx semantics)
  -> communication/idempotency/           (every step needs a key)
  -> reliability/timeouts/                (each step is a hop)
```

If the user explicitly mentions a workflow stage (interview, post-mortem, capacity plan, migration), load the matching template:

```bash
cat references/interview-templates/framework/SKILL.md     # 8-step interview review
cat references/reliability/postmortems/SKILL.md           # Postmortem scoring
cat references/architecture-patterns/strangler-fig/SKILL.md  # Migration safety
cat references/performance/back-of-envelope/SKILL.md      # Estimation review
```

### Step 3 — Read the load-bearing pitfall sections

For each shortlisted skill, **read its `## Common Pitfalls`, `## Trade-offs`, and `## Decision Table` sections**. These are where the canon's accumulated war stories live. Don't substitute your own memory.

```bash
# Pitfalls section for one reference
awk '/^## Common Pitfalls/,/^## /' references/architecture-patterns/saga/SKILL.md

# Trade-offs across multiple
for f in saga distributed-transactions outbox; do
  echo "=== $f ==="
  awk '/^## Trade-offs/,/^## /' references/*/$f/SKILL.md
done

# Decision tables
grep -A 30 "^## Decision Table" references/architecture-patterns/microservices/SKILL.md
```

The "Don't reach for this when" subsection of each skill's `## Why This Exists` block is the single highest-signal place to find anti-pattern warnings.

### Step 4 — Cross-cutting checklist

Run the design through every applicable lens. Use this as a generative prompt — for each row that's *relevant*, produce findings. Each row points at a skill you can grep for the exact criteria.

| Lens | Question | Canonical skills |
|------|----------|------------------|
| **Scope clarity** | Are functional + non-functional requirements written down? | `interview-templates/framework/` |
| **Estimation** | Is there a back-of-envelope: QPS, storage, bandwidth, peak factor? | `performance/back-of-envelope/`, `performance/capacity-modeling/` |
| **Architecture choice** | Is the chosen style (monolith / micro / event-driven / serverless) defended against the simpler one? | `architecture-patterns/SKILL.md` |
| **Data model** | Right database family for the access pattern? Joins, transactions, scans? | `data-systems/SKILL.md`, `data-systems/oltp-vs-olap/` |
| **Replication & partitioning** | Replication strategy (sync/async/leaderless) and shard key articulated? Hot-key risk? | `data-systems/replication/`, `data-systems/partitioning/` |
| **Consistency** | Stated explicitly (linearizable / sequential / read-your-writes / eventual)? | `data-systems/consistency-models/`, `data-systems/cap-pacelc/` |
| **Distributed transactions** | Cross-service writes use saga + compensations + idempotency? | `architecture-patterns/saga/`, `data-systems/distributed-transactions/`, `data-systems/outbox/` |
| **Communication** | Sync/async chosen consciously? REST/gRPC/queue picked for the right reason? | `communication/sync-vs-async/`, `communication/grpc/`, `communication/rest/` |
| **Idempotency & retries** | Every external call has an idempotency key and a retry policy with backoff? | `communication/idempotency/`, `reliability/retries-backoff/` |
| **Backpressure** | Producer outpaces consumer story? Bounded queues? | `communication/backpressure/` |
| **Caching** | Cache-aside / write-through / write-back chosen for the access pattern? Invalidation story? Stampede protection? | `performance/caching/`, `performance/cache-topologies/` |
| **CDN / edge** | Static and cacheable dynamic content offloaded? | `performance/cdn/` |
| **SLOs** | SLI defined, SLO target written, error budget policy? | `reliability/slo-sli-sla/`, `reliability/error-budgets/` |
| **Failure isolation** | Circuit breakers, bulkheads, timeouts, load shedding at the right layers? | `reliability/circuit-breaker/`, `reliability/bulkheads/`, `reliability/timeouts/`, `reliability/load-shedding/` |
| **Rate limiting** | Edge protection + per-tenant quotas? | `reliability/rate-limiting/` |
| **Graceful degradation** | What ships when a downstream is down? | `reliability/graceful-degradation/` |
| **Observability** | Logs (structured), metrics (RED/USE), traces (correlation IDs)? | `reliability/observability/`, `performance/use-red-methods/`, `performance/tracing/` |
| **Deployment** | Blue-green / canary / feature flags? Rollback path? | `reliability/deployment-strategies/`, `reliability/feature-flags/` |
| **DR** | RPO / RTO stated; multi-region failover practiced? | `reliability/disaster-recovery/` |
| **Capacity** | Headroom plan; autoscaling triggers; quota guardrails? | `reliability/capacity-planning/` |
| **Authn / authz** | OAuth2/OIDC/SAML/mTLS chosen for the right audience? RBAC/ABAC clear? | `security/authn/`, `security/authz/`, `security/oauth-oidc/`, `security/mtls/` |
| **Encryption** | At rest, in transit, key rotation? | `security/encryption-at-rest/`, `security/encryption-in-transit/`, `security/secrets-management/` |
| **Threat model** | STRIDE/DREAD applied to the data-flow diagram? | `security/threat-modeling/` |
| **OWASP / app-layer** | SQLi / XSS / CSRF / SSRF / IDOR / deserialization / supply chain? | `security/owasp-top-10/`, `security/sqli/`, `security/xss/`, `security/csrf/`, `security/ssrf/`, `security/idor/`, `security/deserialization/`, `security/vulnerability-management/` |
| **Multi-tenancy** | Isolation model (silo / pool / hybrid); noisy-neighbor protection | `security/multi-tenancy/` |
| **Audit / compliance** | Audit log immutable, retained per regulation? | `security/audit-logging/`, `security/compliance/` |
| **Code design** | SOLID / coupling / DDD bounded contexts visible in the module split? | `code-design/SKILL.md`, `code-design/coupling-cohesion/`, `code-design/ddd/` |
| **Testing** | TDD / pyramid / contract tests / chaos tests planned? | `code-design/tdd-bdd/`, `code-design/testing-pyramid/`, `reliability/chaos-engineering/` |
| **Migration** | If touching legacy: strangler fig, dual-write, cutover plan? | `architecture-patterns/strangler-fig/`, `data-systems/cdc/` |

Not every lens applies to every design. A 2-week prototype RFC for an internal tool needs a *small* subset; a payments rewrite needs all of them.

### Step 5 — Produce the review

Output is a prioritized finding list. Each finding has:

```
[SEVERITY] <one-line title>

What:        <what is wrong / missing / contradictory>
Why it matters: <concrete failure mode if unaddressed>
Reference:   <relative path to the canonical reference under references/>
Specifically: <quote the relevant pitfall / trade-off / decision row>
Fix:         <what to add or change>
```

Severity scale:

| Severity | Meaning |
|----------|---------|
| **CRITICAL** | The design will fail in production at projected load (data loss, security hole, availability incident). Do not ship without addressing. |
| **HIGH** | Significant risk. Ship-blocking unless you can articulate the conscious trade-off. |
| **MEDIUM** | Real gap. Should be addressed in v1; document the deferral if not. |
| **LOW** | Nit, minor improvement, or future optimization. |

End with:
- A one-sentence summary verdict ("ship-ready / ship with the 3 HIGHs addressed / requires another design pass").
- The skills the *author* should read before iterating.
- The lenses you skipped (and why) so the user can challenge the scope.

## Worked Example (abridged)

User pastes: *"We're building a payment-recording service. Single Postgres instance. POST /charge writes to DB then publishes to a Kafka 'charge-completed' topic. Email worker consumes it. Three engineers, 6-month timeline."*

Reviewer's run:

1. **Classify.** Internal payments service, write-heavy on the hot path, multi-step (DB + Kafka + email), one team. Workflow stage: design doc.
2. **Discover.**
   ```bash
   ls references/data-systems/      # outbox, replication, relational, partitioning...
   ls references/communication/     # idempotency, kafka-patterns...
   ls references/security/          # encryption-at-rest, audit-logging...
   ls references/reliability/       # circuit-breaker, retries-backoff, observability...
   grep -l "dual.write\|outbox" references/data-systems/    # -> outbox is canonical
   ```
3. **Read pitfalls.**
   ```bash
   awk '/^## Common Pitfalls/,/^## /' references/data-systems/outbox/SKILL.md
   awk '/^## Common Pitfalls/,/^## /' references/communication/idempotency/SKILL.md
   ```
4. **Apply lenses.** Estimation, data model, distributed transactions, idempotency, observability, audit, encryption.
5. **Output:**

```
[CRITICAL] Dual-write between Postgres and Kafka is not atomic.
What:        POST /charge writes to DB and publishes to Kafka in two steps.
              If the process crashes between them, the DB has the charge but Kafka doesn't (or vice versa).
Why:         Lost or duplicated emails; reconciliation cost grows with volume.
Reference:   references/data-systems/outbox/SKILL.md
Specifically: "Common Pitfalls > Naked publish after DB commit" — the canonical fix is the outbox table + CDC.
Fix:         Write to an `outbox` table in the same DB transaction; have Debezium or a poller forward rows to Kafka.

[CRITICAL] No idempotency key on POST /charge.
What:          Retries from clients or LB will double-charge.
Why:           Direct customer-impact bug; well-known failure mode for payments.
Reference:     references/communication/idempotency/SKILL.md
Specifically:  "Reach for this when ... duplicate charges from naive retries"; "Common Pitfalls > Idempotency window too small".
Fix:           Require Idempotency-Key header; persist (key, response) for 24-72h; return cached response on replay.

[HIGH] Single Postgres = single point of failure.
Reference:     references/data-systems/replication/SKILL.md
Specifically:  "Don't reach for this when ... you cannot tolerate any window of unavailability."
Fix:           At minimum a synchronous replica + automated failover.

[HIGH] No SLO / error budget defined.
Reference:     references/reliability/slo-sli-sla/SKILL.md
Fix:           Pick SLI (POST /charge success rate; p99 latency); set SLO target; document policy.

[MEDIUM] No audit log of charges.
Reference:     references/security/audit-logging/SKILL.md, references/security/compliance/SKILL.md
Fix:           Append-only audit table; retention per PCI-DSS if applicable.

...(continues for observability, encryption-at-rest, capacity, threat model)

VERDICT: requires another design pass. The two CRITICALs (dual-write, idempotency) are
non-negotiable for a payments path. Author should read outbox/, idempotency/, and saga/
before the next iteration.

LENSES SKIPPED: code-design (no code shared), CDN (internal service), multi-tenancy (single-tenant from prompt).
```

## Common Pitfalls

- **Reviewing from memory.** Skipping step 3 (read the pitfall sections) collapses this skill back into freelancing. Always quote the canonical source.
- **Severity inflation.** Marking everything CRITICAL trains the user to ignore severity. Reserve CRITICAL for "will fail in production at projected load."
- **Findings without citations.** Every finding cites a specific skill path. If you can't find a skill that backs the finding, either it's not a finding or the library has a gap (note it).
- **Ignoring scope.** A 2-week internal-tool RFC doesn't need 30 findings. Match depth to stakes.
- **Reviewing the strawman.** Authors sometimes describe a deliberately-simplified design and the *real* design has the missing pieces. Ask before flagging "missing X" if it's plausibly already there.
- **Skipping the verdict.** A review without a one-line verdict and a "skipped lenses" footer is incomplete — the user can't tell whether you ran the full checklist or stopped early.

## Trade-offs

| Benefit of running the full review | Cost |
|------------------------------------|------|
| Catches anti-patterns the author missed | Time-consuming for small designs |
| Citations make findings defensible | Requires the library to be installed and current |
| Severity scale focuses author energy | Severity inflation is a real failure mode |
| Skipped-lenses footer makes scope explicit | Adds prose the user may skim past |

## Decision Table — When to Reach for `/system-review` vs Direct Reference Read

| Situation | Reach for |
|-----------|-----------|
| User has a design and wants critique | `/system-review` (this skill) |
| User wants the deep treatment of one topic | read the canonical reference directly (e.g. `references/data-systems/consensus/SKILL.md`) |
| User wants implementation code | the canonical reference — its code section is copy-pasteable |
| User wants an interview-style walk-through | `references/interview-templates/framework/` + the matching template |
| User wants to browse what's available in a domain | `ls references/<domain>/` then read the domain's `SKILL.md` index |

## References

- Beyer et al. — *Site Reliability Engineering* — https://sre.google/sre-book/table-of-contents/ — production-readiness review patterns.
- Kleppmann — *Designing Data-Intensive Applications* (DDIA) — chs. 5-9 are the canonical reference for replication, partitioning, transactions, consistency.
- Pat Helland — *Life Beyond Distributed Transactions: An Apostate's Opinion* — https://queue.acm.org/detail.cfm?id=3025012 — the foundational paper for "don't try to make distributed systems behave like a single database."
- Adrian Cockcroft — Netflix's *production-readiness review* talks — https://www.infoq.com/presentations/microservices-production-readiness/ — original PRR checklist this skill's lens table is descended from.
- Newman — *Building Microservices* (2nd ed.) — chs. 9-12 on testing, deployment, observability for distributed systems.
- AWS Well-Architected Framework — https://aws.amazon.com/architecture/well-architected/ — vendor-flavored but rigorous review pillars.

## See Also

- [`references/interview-templates/framework/`](references/interview-templates/framework/) — the 8-step framework whose deep-dive step this reviewer runs over a real design instead of a hypothetical one.
- [`references/reliability/postmortems/`](references/reliability/postmortems/) — review a *failed* design after the fact.
- [`references/reliability/chaos-engineering/`](references/reliability/chaos-engineering/) — review tells you what *should* break; chaos tells you what *does* break.
- [`references/code-design/code-review/`](references/code-design/code-review/) — the line-level analogue: review code, not architecture.
- [`references/architecture-patterns/SKILL.md`](references/architecture-patterns/SKILL.md) — first stop for "is this the right architecture?".
- [`references/data-systems/SKILL.md`](references/data-systems/SKILL.md) — first stop for "is this the right database / partition / consistency model?".
- [`references/reliability/SKILL.md`](references/reliability/SKILL.md) — first stop for "is this design failure-aware?".
- [`references/security/SKILL.md`](references/security/SKILL.md) — first stop for "is this design threat-modeled?".
- [`references/performance/SKILL.md`](references/performance/SKILL.md) — first stop for "are the numbers right?".
- [`references/communication/SKILL.md`](references/communication/SKILL.md) — first stop for "is the wire protocol / async story coherent?".
- [`references/code-design/SKILL.md`](references/code-design/SKILL.md) — first stop for "is the code well-structured?".
- [`references/interview-templates/SKILL.md`](references/interview-templates/SKILL.md) — first stop for "score my interview answer."
- [`../../docs/refine-system-skill.md`](../../docs/refine-system-skill.md) — maintainer guide for propagating updates when a review surfaces a library gap.
