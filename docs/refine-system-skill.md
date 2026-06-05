# Refine System Skill — Maintainer Guideline

> **Status**: Repo guideline for maintainers of the `system-skills` plugin. Not a runtime skill — read this when a topic refresh spans multiple files.

Deep-research a topic and propagate updates across every skill it touches.

## When to Use This vs. acquire-system-skill

| Situation | Use |
|-----------|-----|
| One new skill, single folder | [`acquire-system-skill.md`](acquire-system-skill.md) |
| Quick edit to one file | [`acquire-system-skill.md`](acquire-system-skill.md) |
| **Topic spans ≥2 files** | `refine-system-skill` (this doc) |
| Want primary-source deep dive before editing | `refine-system-skill` (this doc) |
| Major release / paradigm shift, audit coverage | `refine-system-skill` (this doc) |
| Inconsistent claims between two existing skills | `refine-system-skill` (this doc) |

## Phases

### Phase 1 — Scope & Blast Radius

Before any research, identify every file the change *might* touch:

```bash
# Wide net
grep -rli "<topic-keyword>" skills/ | sort

# Narrow to canonical homes
ls skills/<likely-folder>/
```

Build a blast-radius map:

```markdown
| File | Role | Likely change |
|------|------|---------------|
| skills/system-review/references/data-systems/consensus/SKILL.md | Canonical home | Full rewrite of "modern variants" section |
| skills/system-review/references/data-systems/replication/SKILL.md | Cross-references | Update one paragraph + link |
| skills/system-review/references/reliability/disaster-recovery/SKILL.md | Mentions consensus quorum | Update one sentence |
```

Confirm the map with the user before researching. If it grew beyond what they expected, ask them to scope it down.

### Phase 2 — Deep Research

Spin up the `superpowers:deep-research` skill (or run multiple parallel web searches) to gather **primary sources**:

- Original paper(s) — arXiv, ACM, USENIX, VLDB, SOSP/OSDI
- RFCs
- Vendor documentation (Postgres / Kafka / Redis / etcd)
- Practitioner writeups (AWS Builders' Library, Google Cloud Architecture Center, High Scalability, Pat Helland, Martin Fowler, "morning paper")
- Reference implementations (etcd's Raft, jepsen test reports for the system)

Deliverable from research: a one-page brief that includes
- one-sentence "why this exists"
- the canonical paper / RFC / blog
- 3-5 trade-offs (benefits ↔ costs)
- 3-5 common failure modes from production reports
- a decision table vs the leading alternative

### Phase 3 — Designate Canonical Home

When a topic touches multiple folders, **one** SKILL.md owns the full treatment; others link to it.

| Topic shape | Canonical home |
|-------------|---------------|
| Algorithm or formal property | `data-systems/<topic>/` (e.g. consensus, CRDTs) |
| Production runtime concern | `reliability/<topic>/` (e.g. circuit breaker, load shedding) |
| Wire-protocol concern | `communication/<topic>/` (e.g. idempotency, versioning) |
| Code structure / pattern | `code-design/<topic>/` |
| Workload pattern (read-heavy, write-heavy, OLAP) | `data-systems/<topic>/` |
| Multi-system architecture pattern | `architecture-patterns/<topic>/` |

Cross-references must say "See `<canonical>` for the full treatment" — don't fork the explanation.

### Phase 4 — Propagate Updates

Touch every file in the blast-radius map. For each:

- **Canonical home**: full update — diagrams, trade-offs, decision table, references.
- **Cross-references**: update only the paragraph that references the topic. Keep the link.
- **Index files** (folder-level `SKILL.md`): update the description line in the table if the scope changed.
- **`## See Also`**: bidirectional. If A links to B, B should link back.

After every batch, run the consistency check:

```bash
# Find lingering references to the OLD claim
grep -rli "<old-claim>" skills/

# Find broken intra-library links
grep -roE "\[.*\]\([^)]*\)" skills/ | grep -v "https"
```

### Phase 5 — Verify

Required before declaring done:

- [ ] Every URL added or modified returns HTTP 200 (`curl -sI`)
- [ ] Every internal link resolves to an existing file
- [ ] No two files contradict each other on the same fact
- [ ] Every changed file still satisfies `acquire-system-skill`'s quality standards
- [ ] `## See Also` is bidirectional everywhere it was touched

## Workflow Example: "Refine our Consistency coverage"

```
Phase 1  blast radius
  grep -rli "consistency\|linearizab\|serializab\|isolation" skills/
    → data-systems/consistency-models/SKILL.md  (canonical)
    → data-systems/replication/SKILL.md
    → data-systems/distributed-transactions/SKILL.md
    → data-systems/consensus/SKILL.md
    → architecture-patterns/cqrs/SKILL.md
    → architecture-patterns/event-sourcing/SKILL.md
    → architecture-patterns/saga/SKILL.md

Phase 2  deep-research
  → DDIA ch. 7-9, Brewer's CAP, Abadi's PACELC, Daniel Abadi 2024 update,
    Jepsen consistency model hierarchy, Spanner TrueTime paper,
    Aphyr / Kyle Kingsbury hierarchy diagram

Phase 3  canonical home
  → data-systems/consistency-models/  (full hierarchy + decision tree)
  → others link to it for definitions

Phase 4  propagate
  → canonical: rewrite hierarchy, add isolation-vs-consistency split, decision tree
  → replication: rewrite "what consistency you get" paragraph + link
  → distributed-transactions: re-anchor isolation vs consistency
  → CQRS / event-sourcing / saga: clarify their consistency story
  → router: update relevant lines

Phase 5  verify
  → links 200 ok, no contradictions, See Also bidirectional, indexes refreshed
```

## Anti-Patterns

- ❌ Updating only the canonical home and leaving cross-references stale.
- ❌ Forking the explanation in cross-references instead of linking. Two truths drift.
- ❌ Researching from secondary blogs only. The original paper / RFC always wins ties.
- ❌ Skipping the blast-radius map and discovering 4 more files mid-edit.
- ❌ Cross-references without `## See Also` reciprocity.
