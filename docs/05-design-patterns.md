# 05 — Design Patterns

Every pattern used in this system, anchored to a concrete flow (never an
abstract definition), including what it costs and what the simpler
alternative would have looked like. Then the patterns deliberately **not**
used, and why — because knowing when *not* to reach for a pattern is the
actual signal of judgment this repo is trying to demonstrate.

## Patterns used

### Modular monolith

- **Where:** the whole system — one ASP.NET Core deployable, internally
  split into Accounts/Ledger/Transfers/Fraud/Compliance/Audit modules that
  only touch their own tables (see [04-architecture.md §3, §5](04-architecture.md#3-components-inside-the-api-c4-level-3)).
- **Problem it solves here:** gives real module boundaries (so the code
  doesn't degrade into a tangled mess, and a future extraction to real
  services would be mechanical) without paying for distributed-systems
  operational cost (service discovery, N deploy pipelines, cross-process
  tracing, N databases) that a solo 2-4-week project can't credibly run to
  a Core standard.
- **Cost:** the boundary is only as real as the discipline enforcing it —
  a rushed change can "cheat" and reach across modules directly unless
  code review (or, if time allows, an architecture test asserting no
  cross-module table access) catches it.
- **Simpler alternative passed up:** a single unstructured project with no
  internal boundaries at all. Rejected because it wouldn't demonstrate — or
  survive — the module-boundary discipline a reviewer is specifically
  checking for at this seniority level.
- **Full write-up:** [ADR-0002](adr/0002-modular-monolith-over-microservices.md).

### Transactional outbox

- **Where:** [04-architecture.md §4.1](04-architecture.md#41-happy-path-transfer-not-flagged-by-any-rule) —
  every state-changing write that needs to trigger an async side effect
  (notifications, projections) writes an `OutboxMessages` row in the same
  DB transaction as the domain write.
- **Problem it solves here:** H4 from [00-exploration.md](00-exploration.md) —
  partial failure between "the ledger posted" and "the event about it was
  published." Without this, a crash between the DB commit and the message
  publish either loses the event or (if publish-then-commit) publishes an
  event for a write that never actually happened.
- **Cost:** a relay process (here, an `IHostedService` polling the outbox
  table) and slightly higher write latency for the polling delay before
  downstream consumers see the event (bounded by NFR-6, 5s).
- **Simpler alternative passed up:** publish directly to Service Bus inside
  the request, no outbox table. Rejected — it's simpler but reintroduces
  H4 exactly: a Service Bus outage or timeout after the DB commit either
  loses the event silently or (if you retry the whole request) risks a
  duplicate ledger posting.

### Saga (orchestration-style, one specific flow)

- **Where:** [04-architecture.md §4.2](04-architecture.md#42-flagged-transfer--hold-review-resolve-the-saga) —
  the hold → human review → clear/reverse flow for fraud-flagged transfers.
- **Problem it solves here:** this flow spans an unbounded, human-scale
  duration (a compliance officer reviewing a case) with a real need for
  compensation if it resolves negatively. A single DB transaction can't
  span that duration.
- **Cost:** more states to reason about (`PendingReview` is now a real,
  externally-visible status, not an implementation detail), and a
  compensating action (returning held funds) has to be written and tested
  as carefully as the forward path.
- **Simpler alternative passed up — and why it's *not* used everywhere:**
  the happy-path transfer (§4.1) is deliberately **not** a saga — it's one
  DB transaction, because it completes in milliseconds inside one process
  and a transaction is strictly simpler and stronger (real atomicity, not
  eventual consistency via compensation) when it's available. Reaching for
  a saga there anyway — because "sagas are a banking pattern" — would be
  the over-engineering failure mode this whole project is trying to avoid
  (Hard Rule 5).

### Idempotent receiver (idempotency keys)

- **Where:** every money-moving endpoint (FR-3.1); dedupe table keyed on
  `(IdempotencyKey, RequestBodyHash)`.
- **Problem it solves here:** H1 — a client retry after a timeout must not
  double-move money. This is the single most load-bearing pattern in the
  system precisely because it's invisible when it's absent (things work
  fine until the one time a retry races a slow response).
- **Cost:** a dedupe table, a retention/expiry policy for old keys (24h,
  per NFR-3), and a hashing check to catch key-reuse-with-different-payload
  (see [04-architecture.md §6](04-architecture.md#6-failure-modes)).
- **Simpler alternative passed up:** "tell clients not to retry blindly."
  Rejected — this is exactly the kind of contract a real API can't enforce
  on its callers, and Stripe/Monzo/Airbnb all converged on the same
  mechanism for the same reason (see [00-exploration.md §1.2](00-exploration.md#12-idempotency-and-exactly-once-money-movement)).

### Append-only journal + projection (event-sourcing-*lite*)

- **Where:** the Ledger module — `JournalEntries`/`JournalLegs` is the
  append-only source of truth; `BalanceProjections` is a derived,
  transactionally-consistent read of it.
- **Problem it solves here:** H3 — auditable history vs. fast balance
  reads, without committing to full event-sourcing infrastructure.
- **Cost:** every balance change is two writes (the journal legs and the
  projection update) inside one transaction, instead of one `UPDATE`.
- **Simpler alternative passed up:** a single mutable `Balance` column,
  updated in place. Rejected explicitly — this is the exact failure mode
  named in the research ([00-exploration.md §1.1](00-exploration.md#11-how-real-ledgercore-banking-systems-are-built))
  as what separates a toy ledger from a real one.
- **Heavier alternative also passed up:** full event sourcing (an event
  store, snapshotting, replay-to-rebuild-state tooling). See
  [ADR-0006](adr/0006-ledger-consistency-model.md) for why that's scope
  this project doesn't need to take on to demonstrate the same underlying
  competency.

### Rules engine (strategy pattern + a case queue)

- **Where:** the Fraud/AML module — each rule (`LargeAmountRule`,
  `NewPayeeVelocityRule`, etc.) implements a common `IFraudRule` interface
  and is evaluated independently; matches produce a named result, never an
  opaque score.
- **Problem it solves here:** H5 — regulators (and, in this project,
  reviewers) need to see *which specific rule* fired and *why*, not just a
  number. Strategy pattern makes each rule independently unit-testable and
  addable without touching the others.
- **Cost:** a rule-versioning story is needed (a rule that changes
  behavior after cases already exist referencing it must not silently
  rewrite history) — handled by storing the rule name+version on the
  `RuleEvaluations` row, not by writing a rule versioning framework.
- **Simpler alternative passed up:** a single large `if/else` chain
  in the transfer-posting code path. Rejected — it fails FR-4.1's
  requirement that each rule be independently testable and it's the first
  thing that becomes unmaintainable as more rules are added.

### Mediator (MediatR) for in-process module decoupling

- **Where:** every cross-module call in [04-architecture.md §3](04-architecture.md#3-components-inside-the-api-c4-level-3)
  goes through an in-process command/query object rather than a direct
  reference between module service classes.
- **Problem it solves here:** keeps the module boundary real (a module
  depends on a command's *shape*, not on another module's concrete
  classes) and gives a single place to hang cross-cutting behavior
  (validation, logging, the audit-event pipeline) as pipeline behaviors.
- **Cost:** a layer of indirection that makes "find where this is handled"
  one hop further than a direct method call — mitigated by consistent
  naming (`{Verb}{Noun}Command`/`Handler` colocated).
- **Simpler alternative passed up:** direct references between module
  service interfaces (e.g. `Transfers` module holds an `ILedgerService`).
  This is genuinely simpler and was seriously considered; MediatR wins
  narrowly because the audit-log requirement (FR-7.1: *every* state
  change produces an audit record) is much cleaner as a pipeline behavior
  than as a call scattered through every handler by hand.

### Resilience: retry + circuit breaker (Polly)

- **Where:** calls to the simulated Identity Verification Provider and
  External Statement Feed, and Service Bus publish calls from the outbox
  relay.
- **Problem it solves here:** these are the system's actual "unreliable
  network call" surfaces; without retry+breaker, a transient blip in a
  simulated external dependency would fail account-opening or delay
  reconciliation unnecessarily, and a sustained outage would let requests
  pile up waiting on a dependency that isn't coming back soon.
- **Cost:** another library dependency and failure modes of its own
  (getting retry/backoff parameters wrong can make an outage worse, not
  better) — mitigated by using Polly's well-tested standard policies
  rather than hand-rolled retry loops.
- **Simpler alternative passed up:** a single retry with no backoff.
  Rejected — doesn't protect a struggling dependency from retry storms,
  which is the actual failure mode being guarded against.

### Options pattern + health checks

- **Where:** all configuration (connection strings, fraud rule thresholds,
  JWT settings) is bound via `IOptions<T>`, validated at startup (fail
  fast, not at first request); `/health` aggregates DB + Service Bus
  reachability.
- **Problem it solves here:** H9 (the Knight Capital deploy lesson) — a
  misconfigured instance should fail to become healthy and never receive
  traffic, rather than partially working.
- **Cost:** negligible — this is standard ASP.NET Core practice, not extra
  architecture.

## Patterns deliberately **not** used

- **Full event sourcing (event store + replay/snapshotting).** The append
  -only-journal-plus-projection pattern above gets the auditability benefit
  real fintechs cite without the operational cost of an event store,
  replay tooling, and snapshot management — a cost that wouldn't pay for
  itself at this system's scale and timeline. See
  [ADR-0006](adr/0006-ledger-consistency-model.md).
- **Microservices / API gateway / service mesh.** See
  [ADR-0002](adr/0002-modular-monolith-over-microservices.md). The module
  boundaries inside the monolith capture the design discipline a
  microservices split would enforce, without the N-deployables,
  N-databases, and distributed-tracing-across-process-boundaries cost.
- **Generic `Repository<T>` / `IRepository<T>` abstraction over EF Core.**
  EF Core's `DbContext` *is* already a Unit of Work + Repository
  abstraction over the database; wrapping it in another generic repository
  interface is a well-known .NET anti-pattern that adds a layer with no
  new capability, just to satisfy "there should be a repository layer."
  Each module exposes purpose-built query/command methods instead (e.g.
  `ILedgerReader.GetBalanceAsync(accountId)`, not
  `IRepository<Account>.GetByIdAsync(id)`).
- **Distributed transactions / 2PC (MSDTC, `TransactionScope` across
  resources).** Rejected everywhere in favor of "one DB transaction where
  possible, saga-with-compensation where a transaction can't span the
  duration." 2PC's availability cost (all participants must be up and
  responsive to commit anything) is exactly the wrong trade-off for a
  payments system, which is why the industry moved away from it — this
  design doesn't reintroduce it.
- **Full CQRS with a physically separate read database.** The system uses
  command/query *segregation* at the code level (via MediatR) but reads
  and writes hit the same database. A physically separate, eventually-
  consistent read store is documented as the scaling next-step (see
  [04-architecture.md §7](04-architecture.md#7-scaling-story)) but isn't
  built, because there's no read load at demo scale that would justify the
  operational cost of keeping a second store in sync.
- **Domain-Driven Design tactical patterns applied uniformly (rich
  aggregates, value objects, and specifications everywhere).** DDD-lite is
  used *only* where the domain genuinely earns it — the Ledger aggregate
  enforces its own invariant (legs sum to zero) in the domain model, not
  just in a service. Accounts and Cases stay closer to simple, well-typed
  records with a repository, because they don't have enough intrinsic
  invariant complexity to justify a full aggregate-with-value-objects
  treatment. Applying heavy DDD tactics uniformly regardless of whether a
  given module needs it is a common over-engineering trap this design
  avoids on purpose.
