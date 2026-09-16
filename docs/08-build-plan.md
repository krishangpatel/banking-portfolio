# 08 — Build Plan

Fifteen phases (0-14), each sized for roughly one 2-5 hour session — fits
inside the 2-4 week / 10-20 hrs/week budget from
[00-exploration.md §4](00-exploration.md#4-assumptions-i-can-override) with
margin. **Every phase ends with the system in a runnable state** (`docker
compose up` works, health check is green, CI is green) — nothing lands
half-done, per Hard Rule 6 and the Phase 2 process rules in the project
instructions.

Per the project's Phase 2 process: before each piece, a short brief (what,
why, which pattern, what Krishang should be able to explain afterward);
default to Socratic — Krishang sketches the approach first, then it's
compared and corrected — unless he says "just build it." After each piece:
runs, tests pass, docs/ADRs updated, README current, committed.

---

## Phase 0 — Solution skeleton & CI baseline

**Build:** Solution structure with one project per module boundary
(`Ledgerline.Api`, `Ledgerline.Accounts`, `Ledgerline.Ledger`,
`Ledgerline.Transfers`, `Ledgerline.Fraud`, `Ledgerline.Compliance`,
`Ledgerline.Audit`, `Ledgerline.Infrastructure`), a `docker-compose.yml`
(API + SQL Server container, Service Bus emulator container per
[ADR-0004](adr/0004-messaging-technology.md)), a `/health` endpoint, and a
GitHub Actions workflow that builds and runs an (initially empty) test
project.
**Demonstrates:** solution architecture matching
[04-architecture.md](04-architecture.md), containerized local dev,
CI from commit one.
**Krishang should be able to explain afterward:** why the project layout
mirrors the module boundaries, and what `/health` will eventually check.
**Leaves runnable:** `docker compose up` → `/health` returns `200`.

## Phase 1 — Accounts module

**Build:** `Account` domain model, EF Core migration, `POST /accounts`,
simulated CIP check (`IIdentityVerificationProvider` fixture-backed
implementation, Polly retry policy), `GET /accounts/{id}`. Unit + Testcontainers
integration tests (FR-1).
**Demonstrates:** EF Core modeling and migrations, the resilience pattern
for a simulated external dependency.
**Explain afterward:** the `PendingVerification` → `Active`/`Rejected`
state machine and why CIP happens before an account is usable.
**Leaves runnable:** an account can be opened and verified end-to-end via
Swagger.

## Phase 2 — Auth & RBAC

**Build:** ASP.NET Core Identity, JWT issuance, the three roles and
policy-based authorization ([ADR-0005](adr/0005-authentication-and-authorization.md)),
a seed script creating one demo user per role. Per-endpoint "denied for
the wrong role" tests (FR-6).
**Demonstrates:** segregation of duties as an enforced, tested property,
not a comment.
**Explain afterward:** why cross-customer account access returns `404` not
`403` (FR-6.2).
**Leaves runnable:** all three demo users can log in and get correctly
scoped tokens.

## Phase 3 — Ledger module

**Build:** append-only `JournalEntries`/`JournalLegs`, the zero-sum
invariant enforced at the data layer, the `BalanceProjections` table
updated in the same transaction ([ADR-0006](adr/0006-ledger-consistency-model.md)),
`GET /accounts/{id}/balance` and `/statement`. Concurrency race test
(NFR-1) and invariant-violation test (NFR-2) against real SQL Server.
**Demonstrates:** the system's central correctness guarantee, tested the
way [ADR-0007](adr/0007-testing-strategy.md) commits to.
**Explain afterward:** what happens, step by step, when 50 concurrent
postings hit one account.
**Leaves runnable:** journal entries can be posted directly (internal/test
endpoint) and balance always matches `SUM(legs)`.

## Phase 4 — Transfers module (idempotent, happy path only)

**Build:** `POST /transfers` with `Idempotency-Key` handling, the dedupe
table with its unique constraint (closing the race from
[07-red-team.md Q2](07-red-team.md)), overdraft rejection, ISO
20022-inspired request/response shape (FR-3.1-3.3). Concurrent-retry and
concurrent-overdraft tests.
**Demonstrates:** the idempotent-receiver pattern end-to-end through the
HTTP layer.
**Explain afterward:** the exact sequence in
[04-architecture.md §4.1](04-architecture.md#41-happy-path-transfer-not-flagged-by-any-rule).
**Leaves runnable:** a full customer→customer transfer works over the API,
retried safely.

## Phase 5 — Outbox & messaging

**Build:** `OutboxMessages` table written in the same transaction as a
transfer posting, the relay `IHostedService`, Service Bus wiring (emulator
locally), one real downstream consumer (a notification/log consumer) to
prove the full pipe, dead-letter handling for poison messages.
**Demonstrates:** the transactional outbox pattern and at-least-once +
idempotent-consumer delivery (Q8).
**Explain afterward:** what happens if the relay crashes mid-publish.
**Leaves runnable:** a posted transfer's event is observably consumed
downstream within NFR-6's 5-second target.

## Phase 6 — Fraud/AML rules engine, case queue, and the saga

**Build:** `IFraudRule` strategy implementations (FR-4.1), the hold →
case → review → clear/reverse saga (FR-3.4,
[04-architecture.md §4.2](04-architecture.md#42-flagged-transfer--hold-review-resolve-the-saga)),
`dueBy` SLA + case-aging (fixing [Q3](07-red-team.md)), `ComplianceOfficer`
resolution endpoints. Per-rule unit tests + end-to-end saga tests (both
resolution paths).
**Demonstrates:** strategy pattern, a real compensating-transaction saga,
segregation of duties in the review step.
**Explain afterward:** why this flow is a saga and the happy-path transfer
isn't ([Q7](07-red-team.md)).
**Leaves runnable:** a large/new-payee transfer is held, appears in the
compliance queue, and can be cleared or rejected with correct ledger
outcomes.

## Phase 7 — Audit trail

**Build:** `Audit` module, a MediatR pipeline behavior that records every
state-changing command to `AuditLog` (append-only, FR-7). Tests asserting
one audit record per action, correctly attributed.
**Demonstrates:** cross-cutting audit as infrastructure, not per-handler
boilerplate.
**Explain afterward:** why audit is a pipeline behavior rather than a call
in every handler ([05-design-patterns.md — Mediator](05-design-patterns.md)).
**Leaves runnable:** every action taken in Phases 1-6 now produces a
correctly-attributed audit record.

## Phase 8 — Reconciliation

**Build:** `IExternalStatementFeed` fixture, the scheduled reconciliation
job, discrepancy case creation (FR-5.1), the manual adjusting-entry repair
endpoint (reusing FR-2.3's reversal mechanism, per
[07-red-team.md Q6](07-red-team.md)).
**Demonstrates:** an event-driven reconciliation pipeline and a real
repair path, not detection-only.
**Explain afterward:** why repair reuses the existing correction mechanism
instead of a special-cased one.
**Leaves runnable:** seeding a deliberate mismatch produces a case within
one run cycle; resolving it posts a correct adjusting entry.

## Phase 9 — Observability

**Build:** Serilog structured logging with correlation-ID enrichment
(NFR-15), OpenTelemetry tracing/metrics, the outbox-lag and case-aging
metrics (NFR-16), expanded `/health` (DB + Service Bus reachability, NFR-5).
**Demonstrates:** end-to-end traceability of a multi-step operation.
**Explain afterward:** how to trace one customer's complaint ("my transfer
didn't go through") from a correlation ID to root cause.
**Leaves runnable:** a transfer that triggers a fraud rule and creates a
case is traceable end-to-end via one correlation ID in the logs.

## Phase 10 — Security hardening pass

**Build:** rate limiting on auth/account-opening, security headers,
input-validation review (FluentValidation coverage audit), CI secret-scan
step, dependency vulnerability scan gate (NFR-9-11, NFR-19), the
architecture fitness test enforcing module boundaries ([Q1](07-red-team.md)),
a short written threat model (STRIDE pass on transfer + case-review flows).
**Demonstrates:** security as a reviewed, tested property.
**Explain afterward:** the threat model, and why each control maps to a
specific threat.
**Leaves runnable:** CI now blocks on vulnerable dependencies and on
module-boundary violations.

## Phase 11 — Test pyramid completion & load test

**Build:** fill any remaining gaps against
[02-functional-requirements.md](02-functional-requirements.md) acceptance
criteria and [03-non-functional-requirements.md](03-non-functional-requirements.md)
Verified items, the k6/bombardier local load test script with committed
results (NFR-7), final CI gate (build + full test suite + vuln scan
blocking merge, under NFR-18's 5-minute budget).
**Demonstrates:** the finished test pyramid and a measured, not assumed,
performance number ([Q5](07-red-team.md)).
**Explain afterward:** the actual measured p95 latency and how it was
measured.
**Leaves runnable:** full CI suite green, load test results committed.

## Phase 12 — Docs & developer-experience polish

**Build:** final README (10-minute-understanding target, architecture
diagram embedded, quickstart, screenshots/short recordings of example
flows), domain glossary, seed/demo-data script covering all three showcase
scenarios (plain transfer, flagged-transfer-through-review, reconciliation
mismatch), exported Postman/Bruno collection, the three runbooks referenced
throughout the docs (`outbox-lag.md`, `backup-restore.md`,
`reconciliation-discrepancy.md`).
**Demonstrates:** documentation as a first-class deliverable (audience #1
and #2 from [01-vision.md](01-vision.md)).
**Explain afterward:** nothing new — this phase is explanation, made
legible to someone who hasn't read the code.
**Leaves runnable:** unchanged; this phase touches no application code.

## Phase 13 — Cloud deployment

**Build:** Azure pricing re-verified against the live calculator (closing
the estimate caveat in [ADR-0003](adr/0003-deployment-and-hosting.md)),
Bicep IaC for App Service/SQL/Service Bus/App Insights/Key Vault, a
manually-triggered (cost-controlled) deploy workflow, the pause/resume
runbook, a live smoke test against the deployed environment.
**Demonstrates:** infrastructure as code and a real, working cloud
deployment within budget (NFR-17).
**Explain afterward:** the full request path in the cloud deployment,
and how to pause it to near-$0.
**Leaves runnable:** a live URL exists in the README, and the local Docker
Compose path still works unchanged.

## Phase 14 — Final polish & interview rehearsal

**Build:** re-run every question in
[07-red-team.md](07-red-team.md) against the *finished* system and correct
any answer that drifted from what was actually built during Phases 0-13;
final README/ADR consistency pass; tag `v1.0`.
**Demonstrates:** the discipline of keeping documentation truthful as
implementation evolves — a real gap between "what the docs claim" and
"what the code does" is exactly the kind of thing a senior reviewer finds
and loses trust over.
**Explain afterward:** everything — this is the rehearsal.
**Leaves runnable:** the tagged `v1.0` is the version defended in
interviews.
