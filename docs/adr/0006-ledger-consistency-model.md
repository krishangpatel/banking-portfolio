# ADR-0006: Ledger consistency model — append-only journal + projection, not full event sourcing

## Status

Accepted

## Context

The ledger is this system's most important correctness surface (H2, H3).
Real ledger design research
([00-exploration.md §1.1, §1.3](../00-exploration.md)) surfaces two
credible patterns: (a) an append-only journal as the source of truth with
a derived balance projection, and (b) full event sourcing — an event
store, snapshotting, and replay-to-rebuild-state tooling, often paired with
full CQRS.

## Decision

**Append-only journal + transactionally-consistent projection** (the
"event-sourcing-*lite*" shape): `JournalEntries`/`JournalLegs` is the
immutable source of truth; `BalanceProjections` is a read-optimized
derived view, updated in the *same* database transaction as the journal
write — never a separately-trusted, independently-mutable value.

## Alternatives considered

- **A single mutable `Balance` column, updated in place on every
  transfer.** Rejected outright — this is the specific failure mode the
  research explicitly names as what separates a toy ledger from a real
  one: a crash mid-update can lose money, and there's no way to
  reconstruct history or prove correctness after the fact. Not a serious
  option for this project.
- **Full event sourcing** (an event store as the persistence model, current
  state rebuilt by replaying events, with snapshotting to bound replay
  cost, typically paired with full CQRS and a physically separate read
  store). This is a real, credible pattern used by some fintechs and was
  seriously considered — it's the "more senior-looking" choice on paper.
  Rejected for this project specifically because of what it would cost
  inside a 2-4 week, 10-20 hrs/week budget: an event store abstraction,
  snapshot strategy, replay tooling, and a physically separate read
  model all need to be built *and correctly tested* to be a credible Core-
  tier claim — and a half-built event-sourcing implementation (replay
  that "mostly works," no real snapshotting, a read model that's really
  just another table kept in sync by hand) would be a worse signal than
  the deliberately simpler pattern actually chosen, done completely and
  correctly. The append-only-journal-plus-projection pattern gets the
  same core property real fintechs cite as valuable — immutable,
  auditable history as the system of record — without needing the event-
  store/replay machinery, because balances here are simple sums of
  journal legs, not complex aggregates that need incremental event
  replay to reconstruct.
- **Physically separate read database (full CQRS) from day one.**
  Rejected for the same reason as full event sourcing — no read load at
  demo scale justifies the operational cost of keeping a second store in
  sync. Documented as the explicit next scaling step
  ([04-architecture.md §7](../04-architecture.md#7-scaling-story)), not
  built now.

## Consequences

- Every balance-changing operation is two writes in one transaction (the
  journal legs, then the projection update) rather than one `UPDATE` —
  a small, deliberate cost for a large correctness guarantee.
- If this system's needs ever grew toward genuinely complex, replay-
  worthy aggregates (which they haven't — an account balance is just a
  sum), migrating toward full event sourcing later would be a natural,
  incremental evolution of the same "immutable log is truth" foundation,
  not a rewrite.
- This is one of the clearest "judgment, not just capability" stories in
  the whole repo, and is treated as such in interview prep — see
  [07-red-team.md](../07-red-team.md) and
  [05-design-patterns.md](../05-design-patterns.md) for how this decision
  is defended live.
