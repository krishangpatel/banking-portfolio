# ADR-0002: Modular monolith over microservices

## Status

Accepted

## Context

The chosen concept (ADR-0001) naturally decomposes into distinct
capabilities — Accounts, Ledger, Transfers, Fraud/AML, Compliance/Case
Review, Audit — that a "composable banking" framing (see
[00-exploration.md §1.1](../00-exploration.md#11-how-real-ledgercore-banking-systems-are-built))
would treat as independently deployable services. The question is whether
to actually deploy them as separate services or as one deployable with
internal module boundaries.

## Decision

**One deployable** (a modular monolith): a single ASP.NET Core process
containing all modules, each owning its own tables, communicating only
through an in-process mediator, backed by one SQL Server database. Module
boundaries are enforced by an architecture fitness test in CI, not just
convention.

## Alternatives considered

- **True microservices**, one per module, each with its own database,
  communicating over the network (REST/gRPC/events). Rejected: this adds
  service discovery, N independent CI/CD pipelines, N databases to manage
  and secure, and distributed tracing across real process boundaries — all
  real operational surface area that a solo engineer in a 2-4 week,
  10-20 hrs/week budget cannot build *and test* to the same Core-tier
  standard as a monolith. A shallow microservices implementation (one that
  skips proper service-to-service resilience, has no real per-service
  ownership story, or fakes independent deployability) reads *worse* to a
  senior reviewer than an honest, well-built monolith — it signals
  resume-driven design (Hard Rule 5) rather than judgment. It also directly
  works against the Knight Capital lesson (H9,
  [00-exploration.md §1.6](../00-exploration.md#16-what-happens-when-banking-software-gets-this-wrong)):
  more independently-deployable services means more ways for a deploy to
  leave the system in a mixed, partially-upgraded state.
- **A single unstructured project with no internal module boundaries at
  all.** Simpler to start, but doesn't demonstrate — or survive — the
  boundary discipline a reviewer at this level is specifically checking
  for, and would make a future real extraction to services a rewrite
  instead of a mechanical change.
- **A "modulith" with a separate database per module, same deployable.**
  A genuine middle ground (physical data isolation without deploy
  isolation), seriously considered. Rejected only for cost/complexity at
  this scale — cross-module transactions (e.g., the transfer saga
  touching Ledger and Compliance) would need distributed-transaction
  handling that a shared database avoids for free, and the marginal
  realism gained didn't justify that cost inside this timeline. This is
  exactly the next step if the system ever needed physical separation —
  see [04-architecture.md §7](../04-architecture.md#7-scaling-story).

## Consequences

- One deploy pipeline, one health check, one place to look for "is the
  system up." Directly supports NFR-5 and the Knight Capital-derived
  design goal (H9).
- Module boundaries are only as real as the fitness test enforcing them —
  documented as a real, named limitation (weaker than physical database
  separation) rather than implied to be equivalent to true service
  isolation.
- Still requires real async infrastructure (outbox, message broker, saga)
  for the flows that genuinely need it (H4) — this is *not* microservices-
  lite, it's a monolith that uses distributed-systems patterns exactly
  where the domain requires them and nowhere else (see
  [05-design-patterns.md](../05-design-patterns.md) and
  [07-red-team.md Q10](../07-red-team.md)).
