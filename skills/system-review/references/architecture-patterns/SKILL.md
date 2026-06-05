---
name: architecture-patterns
description: Index of high-level architecture patterns — monolith, modular monolith, microservices, SOA, serverless, event-driven, CQRS, event sourcing, hexagonal/ports-adapters, layered, clean architecture, strangler fig, saga, BFF, service mesh, sidecar, pipes-and-filters, lambda, kappa. Use when choosing the overall shape of a system, deciding if a problem warrants microservices, or navigating between architecture-level patterns.
---

# Architecture Patterns

How to **shape** a system at the highest level — services, deployments, data flow, team boundaries. The decision here propagates everywhere downstream.

## Skills

| Skill | Description |
|-------|-------------|
| [Monolith](monolith/) | Single deployable unit. The default for one team / one domain — fastest to ship, simplest to operate. |
| [Modular Monolith](modular-monolith/) | One deploy, many enforced internal modules. Keeps monolith velocity while preserving the option to split later. |
| [Microservices](microservices/) | Independent deployables per bounded context. Buy team independence with distributed-system tax. |
| [SOA](soa/) | Pre-microservices service orientation — coarser-grained services, ESB, contract-first. Still shows up in enterprise. |
| [Serverless](serverless/) | Functions-as-a-Service + managed backing services. Cost-per-request, autoscale-to-zero, vendor-coupled. |
| [Event-Driven](event-driven/) | Producers emit events; consumers react asynchronously. Decouples in time + identity at the cost of debuggability. |
| [CQRS](cqrs/) | Separate read and write models. Use when read patterns differ enough from write patterns that one schema can't serve both well. |
| [Event Sourcing](event-sourcing/) | Persist *changes* (events) instead of state. Gives you a free audit log + temporal queries; pays in projection complexity. |
| [Hexagonal / Ports-Adapters](hexagonal/) | Domain at the center, IO at the edge. Test domain logic without spinning up infra. |
| [Layered Architecture](layered/) | Classic UI → service → repository → DB. Easy onboarding, leaks abstractions when not enforced. |
| [Clean Architecture](clean-architecture/) | Concentric circles — entities → use-cases → adapters → frameworks. Explicit dependency rule. |
| [Strangler Fig](strangler-fig/) | Migrate a legacy system route-by-route by routing traffic through a new system that takes over piece by piece. |
| [Saga](saga/) | Long-running multi-service workflow with compensations instead of distributed transactions. Choreography vs orchestration. |
| [BFF (Backend-for-Frontend)](bff/) | Per-client API tier (web, iOS, Android) over a shared service layer. Removes the "kitchen-sink API" problem. |
| [Service Mesh](service-mesh/) | Sidecar-deployed proxy fleet handling mTLS, retries, traffic shifting, observability for east-west traffic. |
| [Sidecar](sidecar/) | Process running alongside the main app providing cross-cutting concerns (logging, proxy, secrets injection). |
| [Pipes and Filters](pipes-and-filters/) | Process data through a chain of independent filters connected by pipes. Used in stream processing, log shipping, ETL. |
| [Lambda Architecture](lambda-architecture/) | Parallel batch + speed layers reconciled at query time. Fault-tolerant; doubles your code. |
| [Kappa Architecture](kappa-architecture/) | Stream-only, single code path. Replay history through the stream when correctness changes. |

## When to Use What

| Constraint | Start with |
|------------|------------|
| 1 team, simple domain, want to ship | Monolith |
| 1 team, growing domain, want optionality | Modular Monolith |
| 3+ teams blocking each other on deploys | Microservices |
| Cost sensitive, bursty traffic | Serverless |
| Heavy async workflow, multiple consumers per event | Event-Driven |
| Read patterns radically different from write patterns | CQRS |
| Audit / regulatory requirement to know "what happened when" | Event Sourcing |
| Domain logic getting tangled with the framework / DB | Hexagonal or Clean Architecture |
| Stuck with a legacy system you can't rewrite | Strangler Fig |
| Multi-step workflow across services without 2PC | Saga |
| Multiple client UIs with different data needs | BFF |
| Cross-cutting reliability concerns spanning many services | Service Mesh + Sidecar |
| Stream + batch with different SLAs | Lambda |
| Stream-only with correctness via reprocessing | Kappa |

## The Architectural Default

> Start with a **modular monolith** unless you have a concrete forcing function for distribution.

Forcing functions that justify microservices:
- ≥3 teams whose deploy cadence is blocked by each other
- Components with genuinely different scaling envelopes (e.g. ML inference vs CRUD API)
- Polyglot requirement (real, not aspirational)
- Regulatory / data-residency boundaries

If none of those apply, microservices add cost (network, ops, observability, debugging, eventual consistency) without paying it back. **Not all problems need a microservice.**

## See Also

- `data-systems/SKILL.md` — once you've shaped the system, pick storage
- `communication/SKILL.md` — sync vs async, REST vs gRPC, queues
- `reliability/SKILL.md` — patterns to keep distributed systems alive
- `code-design/ddd/` — bounded contexts ground the service split
- `interview-templates/framework/` — applying these to a real design problem
