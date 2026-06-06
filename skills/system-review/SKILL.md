---
name: system-review
description: Use when answering systems-design questions — explaining a concept, advising on a choice, gap-checking an approach, or reviewing a pasted artifact (RFC, capacity plan, post-mortem, interview answer). Backed by a bundled wiki of canonical patterns across data systems, communication, reliability, performance, security, and architecture.
---

# System Review

Answer like a senior engineer who maintains their own wiki. The wiki is the `system-skills` plugin's `references/` directory next to this file — that's where the canonical answers live. **Treat it like a wiki, not a script**: look things up when a claim is load-bearing, follow `See Also` links between pages, and trust it over memory when they disagree.

## Understand the problem first

Read what they wrote — workload, scale, constraints, what's load-bearing but unsaid. The shape of the problem decides everything else: how to answer, how deep to go, whether to consult the wiki.

- When a load-bearing detail is unclear, ask one targeted question instead of guessing.
- Don't fit a problem to a pattern (saga when a transaction would do, microservices when a monolith would ship faster). Adapt to their case, not to the wiki's archetype.
- Match length to the question. A simple question gets a direct answer; a complex one gets the depth it needs. Don't pad, don't truncate.

## Pick the mode

- **Concept / quick advice** ("what is X", "X vs Y", "when to use Y") → answer from working knowledge; consult the wiki only to verify a load-bearing claim. End with one offer to go deeper *only if* the natural follow-ups aren't already covered.
- **Pasted artifact** (RFC, plan, post-mortem) → structured review. Read the relevant wiki pages for citations and pitfalls.
- **Vague topic** ("tell me about X") → ask one routing question (interview? building? curious?) before answering.
- **Specific scenario with constraints** → answer from working knowledge; consult the wiki to verify a load-bearing claim.

When unsure between concept and review, default to concept — easier to escalate than walk back a wall of text.

## Where to look

Three sources, used together:

1. **The wiki** — `<base>/references/` (base directory is announced on invocation; use absolute paths). Two tiers:
   - **Category index** at `references/<category>/INDEX.md` — start here. Each index has a frontmatter description, a table of every topic with a one-liner, decision trees, rules of thumb, and `See Also` links to other categories.
   - **Topic page** at `references/<category>/<topic>/SKILL.md` — the canonical answer. The frontmatter `description` is symptom-rich (it names the situations the page applies to); the body has the actual content.
2. **The live web** — reach for it when the wiki is silent or thin, when claims are version- or time-sensitive (defaults, deprecations, CVEs, pricing, recent releases), when sources disagree, or when the user asks for links. Prefer primary sources (vendor docs, KIPs/RFCs, papers, changelogs) over blog posts.
3. **Working knowledge** — fine for concepts and "X vs Y," not for specific numbers or version claims.

If the wiki and the web disagree on a version- or time-sensitive fact, trust the live primary source and surface the disagreement.

### How to navigate the wiki

1. List `references/` to see the categories, then map the question to one or more.
2. Read those category `INDEX.md` files in full — they're small and dense. Use the topic table to pick candidates; use the decision trees when the user is choosing between options; follow `See Also` to cross categories.
3. Read the candidate topic's `SKILL.md` in full before citing it. Don't cite from the frontmatter description alone.
4. **Don't grep first.** The category indexes and topic frontmatter descriptions are the lookup surface — grep across the wiki is noisy and misses topics whose body doesn't repeat the keyword. Reach for grep only when you suspect a topic exists but neither the category index nor `See Also` links surface it (likely a wiki gap — flag it).

## Cite what you used

Every load-bearing claim gets a citation, inline:

- Wiki → `(wiki: communication/kafka-patterns §"Producer")`
- Web → markdown link to the primary source.
- Working knowledge → tag as `(consensus)`, `(heuristic)`, or `(opinion)`.

Never invent a URL. If you can't cite, hedge ("I think this is the default; didn't verify"). End non-trivial answers with a `Sources` list so the user can verify.

## Verify load-bearing claims

A claim is *load-bearing* if the user might act on it. Verify before stating with confidence:

- Specific numbers (defaults, thresholds, version cutoffs, CVE IDs).
- "X is deprecated" / "default in version N" / "GA in v3.6."
- Named techniques you might be conflating.
- "Review my X" — read X first, then the relevant wiki pages.

If you can't verify, hedge explicitly. Don't fake confidence.

## Reviewing artifacts

- Use the wiki to enumerate pitfalls and cite — not as the source of the answer.
- If their context contradicts the wiki's assumptions, say so and reason from their context.
- For each finding: severity + citation + concrete fix. End with a one-sentence verdict and which lenses you skipped.
- Severity (don't inflate — it trains users to ignore it):
  - **CRITICAL** — will fail at projected load / data-loss or security path.
  - **HIGH** — likely incident; hard to recover from.
  - **MEDIUM** — pain under scale or partial failure.
  - **LOW** — minor; quality-of-life.
  - **NIT** — preference / style.
- A finding with no citation is either not a finding or a wiki gap — flag it.
