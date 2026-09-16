# ADR-0007: Testing strategy — real database over mocks for anything the ledger depends on

## Status

Accepted

## Context

The system's central correctness claims (NFR-1, NFR-2: the ledger never
drifts, concurrent transfers never overdraw an account) are claims about
behavior *under real database transaction semantics and real concurrency*
— row locking, constraint enforcement, isolation levels. These are exactly
the behaviors a mocked or in-memory database provider cannot faithfully
reproduce.

## Decision

**A three-layer test pyramid, with the middle layer against a real
database:**

1. **Unit tests** (xUnit) for domain logic that has no database
   dependency at all — fraud rule evaluation, journal-leg validation
   logic, idempotency-key hashing.
2. **Integration tests** (xUnit + **Testcontainers**) against a real,
   ephemeral SQL Server container spun up per test run — covering
   repository/EF Core behavior, the concurrency tests (NFR-1), the
   constraint-enforcement tests (NFR-2), and the outbox/saga flows
   end-to-end against real transactions.
3. **A handful of full end-to-end tests** via `WebApplicationFactory`,
   covering the complete happy-path and flagged-transfer flows through the
   actual HTTP pipeline (auth, validation, everything).

EF Core's in-memory provider is **not used** for any test that touches
transaction, concurrency, or constraint behavior — it does not enforce
relational constraints or real transaction isolation, and using it for
these tests would produce tests that pass for the wrong reason.

## Alternatives considered

- **EF Core in-memory provider for all data-layer tests.** Faster and
  simpler to set up — no Docker dependency in CI — and seriously tempting
  for that reason. Rejected specifically because it cannot validate the
  claims that matter most here (NFR-1, NFR-2): it doesn't enforce
  database-level constraints or real locking/isolation behavior, so a test
  suite built on it could pass while the actual SQL Server deployment
  still allowed a race condition or a constraint violation. A green test
  suite that doesn't prove the thing it claims to prove is worse than no
  test.
- **Mocking the `DbContext`/repository interfaces entirely (fully isolated
  unit tests for everything, including data-access code).** Rejected for
  the same reason, one layer further removed — mocking away the database
  means never actually exercising the SQL that enforces the ledger
  invariant, which is precisely the thing under test.
- **A real, persistent shared test database (not per-run ephemeral).**
  Rejected — shared test state causes flaky, order-dependent tests
  (especially for the concurrency tests, which deliberately create
  contention) and doesn't reset cleanly between CI runs. Testcontainers'
  per-run ephemeral container avoids this at the cost of slightly slower
  CI (bounded by NFR-18: full suite under 5 minutes).

## Consequences

- CI requires Docker-in-Docker (or an equivalent) to run Testcontainers —
  a real but standard requirement for GitHub Actions runners, not an
  exotic one.
- The test suite is slower than a pure-mock suite would be, but still
  fits NFR-18's 5-minute budget, verified directly rather than assumed.
- This is a deliberate, defensible answer to "why real SQL Server instead
  of an in-memory provider" — see [07-red-team.md](../07-red-team.md) — and
  it's the same reasoning, applied consistently, behind using
  Testcontainers rather than mocks anywhere the ledger's actual guarantees
  are on the line.
