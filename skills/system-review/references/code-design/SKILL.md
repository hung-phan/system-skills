---
name: code-design
description: Index of code-design skills — SOLID, DRY/KISS/YAGNI, coupling and cohesion, code smells, refactoring catalog, GoF design patterns, DDD, bounded contexts, aggregates, TDD/BDD, testing pyramid, clean code principles, pragmatic programmer heuristics, code review, pair programming. Use when refactoring messy code, picking a design pattern, decomposing a domain, deciding what to test, or reviewing a teammate's PR.
---

# Code Design

How to write code that **survives change**. The difference between a codebase you can ship in week 100 and one you can't.

## Skills

### Principles

| Skill | Description |
|-------|-------------|
| [SOLID](solid/) | Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion. Concrete signals, not slogans. |
| [DRY / KISS / YAGNI](dry-kiss-yagni/) | The three slogans worth knowing — and where each one lies to you. |
| [Coupling and Cohesion](coupling-cohesion/) | The two metrics that actually predict change cost. Levels of each. |
| [Clean Code Principles](clean-code-principles/) | Naming, function size, comments, error handling, classes, tests — Bob Martin's catalog distilled. |
| [Pragmatic Programmer Heuristics](pragmatic-programmer/) | Tracer bullets, broken windows, orthogonality, DRY (the original meaning), reversibility, the boy scout rule. |

### Smells & refactoring

| Skill | Description |
|-------|-------------|
| [Code Smells](code-smells/) | Long method, large class, feature envy, primitive obsession, shotgun surgery, etc. — Fowler's catalog with detection cues. |
| [Refactoring Catalog](refactoring-catalog/) | The mechanical moves: Extract Method, Replace Conditional with Polymorphism, Move Field, etc. With test-first cadence. |
| [Characterization Tests](characterization-tests/) | Pin down legacy behavior before changing it. Michael Feathers's *Working Effectively with Legacy Code*. |

### Patterns

| Skill | Description |
|-------|-------------|
| [Design Patterns (GoF)](design-patterns-gof/) | Creational, structural, behavioral — when each is the right tool, when it's overengineering. |
| [Anti-Patterns](anti-patterns/) | God object, anemic domain, lava layer, golden hammer, premature abstraction. What they look like and why they happen. |

### Domain

| Skill | Description |
|-------|-------------|
| [Domain-Driven Design (DDD)](ddd/) | Ubiquitous language, bounded contexts, context maps. Strategic vs tactical DDD. |
| [Aggregates & Entities](aggregates/) | Tactical DDD building blocks. Invariants, transactional boundaries, lifecycles. |
| [Value Objects](value-objects/) | Immutable types that encapsulate domain rules. The opposite of primitive obsession. |

### Testing

| Skill | Description |
|-------|-------------|
| [TDD / BDD](tdd-bdd/) | Red-green-refactor. Test names as specs. When TDD pays back; when it doesn't. |
| [Testing Pyramid](testing-pyramid/) | Many unit, fewer integration, very few e2e. Cohn's pyramid + the ice-cream-cone anti-pattern. |
| [Test Doubles](test-doubles/) | Dummies, stubs, spies, mocks, fakes. Meszaros's terminology. Mocking pitfalls. |
| [Property-Based Testing](property-based-testing/) | Hypothesis, fast-check, QuickCheck. Generative testing. Find edge cases the human didn't. |

### Process

| Skill | Description |
|-------|-------------|
| [Code Review](code-review/) | What reviewers should look for. How to give feedback. PR size thresholds. |
| [Pair Programming](pair-programming/) | Driver/navigator, ping-pong TDD. When pairing pays. When it doesn't. |
| [Trunk-Based Development](trunk-based-development/) | Short-lived branches, feature flags, continuous integration. vs long-lived feature branches. |

## Decision Trees

### "Should I refactor this?"

| Signal | Refactor? |
|--------|-----------|
| Code smell + you're about to change this code | Yes — rule of three / boy scout rule |
| Code smell, no near-term change | No — tracking debt is fine, refactoring it is YAGNI |
| Adding a feature is hard *because of* the structure | Yes — refactor first to make the change easy, then make the easy change |
| It's "ugly" but works | No, unless above |

### "Pick a pattern"

| Need | Pattern |
|------|---------|
| Many subtypes; switch on type smell | Strategy / Polymorphism |
| Algorithm with extension points | Template Method |
| Decouple message sender and handler | Observer / Mediator |
| Decouple object creation from usage | Factory / Abstract Factory / Builder |
| Wrap a third-party API behind your own interface | Adapter / Anti-Corruption Layer |
| Same operation, different objects | Visitor (rare; usually a smell) |
| Stateful object with lifecycle | State |
| Multiple chained responsibilities | Chain of Responsibility |
| Cross-cutting concern (logging, retry) | Decorator |

### "How much do I test?"

| Code shape | Test approach |
|------------|---------------|
| Pure logic with branches | Many unit tests, property-based for invariants |
| Wires a few collaborators | Integration test |
| End-to-end critical path | One e2e + many unit underneath |
| Glue code (1 line, no logic) | Don't test; let types catch it |
| Algorithm with mathematical properties | Property-based test |

## Rules of Thumb

- **Make the change easy, then make the easy change** — Kent Beck. Refactoring is preparation, not cleanup.
- **You aren't going to need it** — until you do. Then add it. Don't pre-build optionality.
- **Duplication is far cheaper than the wrong abstraction** — Sandi Metz. Wait for 3 instances minimum.
- **Couple to abstractions, not concretions** — but don't invent abstractions you don't yet have multiple implementers for.
- **Names are the highest-leverage code change.** A confusing name costs every reader forever. Renaming is free.
- **The test pyramid, not the ice-cream cone.** When you find yourself maintaining hundreds of slow e2e tests, your unit layer is missing.
- **One PR = one logical change.** Reviewers can hold ~400 LOC in their head; bigger PRs get rubber-stamped.

## See Also

- `architecture-patterns/hexagonal/`, `architecture-patterns/clean-architecture/` — domain isolation at the architecture level
- `architecture-patterns/strangler-fig/` — when refactoring is too small a hammer
- `reliability/observability/` — debuggable code is reliable code
- `security/threat-modeling/` — secure code review

## References

- Fowler, *Refactoring: Improving the Design of Existing Code* (2nd ed) — https://martinfowler.com/books/refactoring.html
- Martin, *Clean Code* — https://www.oreilly.com/library/view/clean-code-a/9780136083238/
- Martin, *Clean Architecture* — https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/
- Hunt & Thomas, *The Pragmatic Programmer* (20th anniv. ed.) — https://pragprog.com/titles/tpp20/
- Feathers, *Working Effectively with Legacy Code* — https://www.oreilly.com/library/view/working-effectively-with/0131177052/
- Evans, *Domain-Driven Design* — https://www.dddcommunity.org/book/evans_2003/
- Vernon, *Implementing Domain-Driven Design* — https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/
- Gamma, Helm, Johnson, Vlissides, *Design Patterns: Elements of Reusable Object-Oriented Software*
- Beck, *Test-Driven Development: By Example* — https://www.oreilly.com/library/view/test-driven-development/0321146530/
