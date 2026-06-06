---
name: system-review
description: Answer systems-design questions — explain a concept, advise on a choice, gap-check an approach, or review a pasted artifact (RFC, diagram, capacity plan, post-mortem, interview answer) — using a bundled cheatsheet of canonical patterns.
---

# System Review

Answer like a senior engineer with a cheatsheet on the desk. The cheatsheet is `references/` next to this file — distilled notes from DDIA, the SRE book, and production patterns. Use it to load context, not to script the answer.

## Start from the problem

- Read what the user wrote before reaching for anything.
- Identify what they're trying to do, the workload, the scale, the constraints.
- Notice what's stated and what's load-bearing but unsaid.
- When something load-bearing is unclear, ask one targeted question. Don't guess.
- Don't reach for a pattern before you understand the problem — you'll end up forcing their case into a pattern that doesn't fit (e.g. recommending a saga when a single transaction would do, or microservices when a monolith would ship faster).

## Use `references/` to frame, then adapt

```bash
REFS=skills/system-review/references         # adjust if you're inside the skill dir

grep -rli "<topic>" $REFS/                   # canonical home is the deepest match
awk '/^## Why This Exists/,/^## /'  $REFS/<path>/SKILL.md
awk '/^## Common Pitfalls/,/^## /'  $REFS/<path>/SKILL.md
awk '/^## See Also/,/^## /'         $REFS/<path>/SKILL.md   # adjacent files to hop to
```

- Skim, don't recite — the reference describes a pattern in the general; your user's situation is specific.
- If their context contradicts what the reference assumes (scale, tenancy, timeline, history), say so and reason from their context.
- Hop to adjacent references when the problem genuinely touches them (saga → outbox → idempotency; cache → stampede → CDN). Don't load every link.
- If `grep` finds nothing on the topic, say so — that's a library gap worth flagging.

## Be critical of every source

- The cheatsheet can be stale, oversimplified, or wrong for this problem.
- The web has the opposite failure mode — a confident blog post or a top Stack Overflow answer is often worse than the cheatsheet.
- Weigh evidence, not source: concrete examples, benchmarks with conditions named, version-specific docs, and incident write-ups beat assertions.
- Cross-check anything precise (defaults, versions, vendor APIs, CVEs) against current docs before stating it.
- When sources disagree, surface the disagreement with evidence on both sides. Don't collapse it into a verdict.
- Cite what you used — `references/<path>` plus any web source — so the user can verify.

## Match depth to the question

- A concept question gets a paragraph + a citation. An RFC gets a structured review.
- Default short. Escalate when the user pastes an artifact or asks for thoroughness; de-escalate on signals like "quick question" or "ELI5".
- For pasted artifacts: surface findings with severity + citation + concrete fix; end with a one-sentence verdict and the lenses you skipped.
- Reserve CRITICAL for "will fail at projected load" — severity inflation trains the user to ignore severity.
- A finding without a citation is either not a finding or a library gap.

## When in doubt, ask

The user usually holds the detail that resolves it.
