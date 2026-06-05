---
name: reliability
description: Index of reliability and SRE skills — SLO/SLI/SLA, error budgets, retries+backoff, circuit breakers, bulkheads, rate limiting, load shedding, timeouts, health checks, graceful degradation, chaos engineering, observability (logs/metrics/traces), incident response, postmortems, capacity planning, deployment strategies (blue-green/canary/feature flags), disaster recovery. Use when designing for failure, debugging cascading failures, defining SLOs, planning rollouts, or running incidents.
---

# Reliability

How a system **stays up** when its parts don't. This folder is the runtime survival kit — every distributed system needs most of these.

## Skills

### Define what "reliable" means

| Skill | Description |
|-------|-------------|
| [SLO / SLI / SLA](slo-sli-sla/) | Service level objectives, indicators, and agreements. The contract reliability work is sized against. |
| [Error Budgets](error-budgets/) | Operationalize SLOs. The "freedom to be unreliable" you've earned this quarter. |

### Failure containment

| Skill | Description |
|-------|-------------|
| [Retries & Backoff](retries-backoff/) | Exponential backoff with jitter, bounded retry budgets, idempotency interaction. |
| [Circuit Breaker](circuit-breaker/) | Stop calling a downstream that's failing. Half-open recovery, per-host vs aggregate. |
| [Bulkheads](bulkheads/) | Partition resources so one bad caller can't drain the whole pool. |
| [Rate Limiting](rate-limiting/) | Token bucket, leaky bucket, sliding window. Per-user / per-tenant / per-route. |
| [Load Shedding](load-shedding/) | Drop excess traffic at the edge before it kills you. Priority queues + admission control. |
| [Timeouts](timeouts/) | Every wait has a ceiling. Cascading-timeout budgets. |
| [Health Checks](health-checks/) | Liveness vs readiness. What goes in each. Trapdoor health checks. |
| [Graceful Degradation](graceful-degradation/) | Serve a worse answer instead of no answer. Fallback caches, default responses, feature drops. |

### Production engineering

| Skill | Description |
|-------|-------------|
| [Chaos Engineering](chaos-engineering/) | Inject failures in prod to surface latent bugs before they surface themselves. Game days, principles, tooling. |
| [Observability](observability/) | Logs, metrics, traces — when each pays off. Cardinality, sampling, RED/USE methods. |
| [Incident Response](incident-response/) | The on-call playbook. Roles (IC / OCC), severity, comms, escalation. |
| [Postmortems](postmortems/) | Blameless writeups. Five whys vs systemic causes. Action items that don't rot. |
| [Capacity Planning](capacity-planning/) | Forecast headroom. Load + growth + safety factor. Provision before you bleed. |
| [Disaster Recovery](disaster-recovery/) | RPO / RTO. Multi-region, multi-AZ, restore drills. The backup you don't test isn't a backup. |
| [Runbooks](runbooks/) | Step-by-step recovery instructions for known failure modes. Living documents. |

### Rollouts

| Skill | Description |
|-------|-------------|
| [Deployment Strategies](deployment-strategies/) | Blue-green vs canary vs rolling vs shadow. Recipe per traffic shape. |
| [Feature Flags](feature-flags/) | Decouple deploy from release. Kill-switches, percent rollouts, targeted releases. |

## Decision Trees

### "I have a downstream that occasionally fails. What do I add?"

```
Is the failure transient?
├── yes → retry with exponential backoff + jitter
│         └── is the call mutating? → MUST be idempotent first
└── no  → don't retry; surface the error

Is it failing more than X% of calls?
├── yes → circuit breaker (open the breaker)
└── no  → keep going

Is one bad caller drowning out everyone else?
└── yes → bulkhead (per-tenant pool / queue)

Are we still saturated?
└── yes → load shedding (drop low-priority requests)
```

### "What deploy strategy?"

| Constraint | Use |
|------------|-----|
| Stateless service, want zero-downtime, ok with double-running | Blue-green |
| Want to expose new version to a subset before full rollout | Canary |
| Many instances, ok rolling slowly | Rolling |
| Need to A/B-test or kill instantly | Feature flags |
| Stateful service / DB migration | Custom — coordinate with schema-evolution |

### "How much do I instrument?"

| What you need | Use |
|---------------|-----|
| What broke? | Logs (structured) |
| How much / how often / how slow? | Metrics |
| Where in the request path? | Distributed traces |
| Who saw it? | Real user monitoring + session replay |
| Will it break? | Synthetic checks + canaries |

## The SRE Mindset

> "Hope is not a strategy." — Site Reliability Engineering (Google)

Core principles distilled from the SRE Book + Workbook:

1. **Reliability is a feature, not a phase.** Negotiated like any other.
2. **Service Level Objectives, not 100%.** A 100% target is more expensive than the user wants. Set the SLO at where the next 9 isn't worth the cost.
3. **Error budget = how much unreliability you've earned.** Spend it on velocity (deploys, experiments). Empty? Stop deploying, fix what's broken.
4. **Toil is the enemy.** Anything an operator does manually that a script could do is debt. Halve the toil quarter over quarter.
5. **Postmortems are blameless.** Systems fail; people surface failures. Punishing surfacing makes them go underground.
6. **Defense in depth.** Every layer assumes the previous one failed.
7. **Eliminate cascading failure.** The thing that pages you at 3 AM is rarely the original cause; it's the amplifier.

## Rules of Thumb

- **You can't operate what you can't see.** Observability before optimization. Before scaling. Before refactoring.
- **Set the SLO before adding features.** Otherwise reliability is whatever happens to fall out.
- **Every retry needs a budget.** A retry storm is a self-DDoS.
- **Every timeout is too long.** And every timeout is too short. Tune empirically; default to "shorter than you think".
- **Test recovery, not just backups.** Untested DR is faith.
- **Canary before blue-green for risky changes.** Blue-green flips 100% at once.
- **Feature flag every risky deploy.** Decouple release from deploy.

## See Also

- `architecture-patterns/saga/`, `architecture-patterns/event-driven/` — pattern-level resilience
- `communication/idempotency/`, `communication/backpressure/`
- `data-systems/replication/`, `data-systems/consensus/` — durability primitives
- `performance/tail-latency/`, `performance/capacity-modeling/`
- `security/audit-logging/` — overlap with observability

## References

- Beyer, Jones, Petoff, Murphy — *Site Reliability Engineering* (free) — https://sre.google/sre-book/table-of-contents/
- Beyer et al. — *The Site Reliability Workbook* (free) — https://sre.google/workbook/table-of-contents/
- Adkins, Beyer et al. — *Building Secure and Reliable Systems* (free) — https://sre.google/books/building-secure-reliable-systems/
- Rosenthal et al. — *Chaos Engineering: System Resiliency in Practice* — https://www.oreilly.com/library/view/chaos-engineering/9781492043850/
- AWS Builders' Library — https://aws.amazon.com/builders-library/
