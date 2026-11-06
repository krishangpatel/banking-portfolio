# Ledgerline

A core-banking backend (accounts, double-entry ledger, transfers, fraud/AML
review, reconciliation, audit) built to demonstrate enterprise .NET
engineering depth and systems design judgment. Full design docs — vision,
requirements, architecture diagrams, ADRs, the build plan — live in
[`docs/`](docs/).

**Status:** Phase 0 of 15 (solution skeleton & CI baseline) — in progress.
This README will grow with each phase; the full quickstart, architecture
diagram, and demo walkthrough land in Phase 12 per
[docs/08-build-plan.md](docs/08-build-plan.md).

## Run it locally

```
docker compose up
```

Brings up the API, a SQL Server container, and the Azure Service Bus
emulator (with its own dependent SQL Server instance). No cloud account or
`.env` file needed — `.env.example` documents the optional overrides.

```
curl http://localhost:8080/health
```

Should return `200`. At this phase that only confirms the API process is
up — it doesn't check the database or Service Bus yet (that lands in
[Phase 9](docs/08-build-plan.md#phase-9--observability)).

## Solution layout

One project per module boundary from
[docs/04-architecture.md](docs/04-architecture.md), per
[ADR-0002](docs/adr/0002-modular-monolith-over-microservices.md):

```
src/
  Ledgerline.Api/              — the single deployable (ASP.NET Core Web API)
  Ledgerline.Accounts/
  Ledgerline.Ledger/
  Ledgerline.Transfers/
  Ledgerline.Fraud/
  Ledgerline.Compliance/
  Ledgerline.Audit/
  Ledgerline.Infrastructure/   — shared cross-cutting code, no module logic
tests/
  Ledgerline.Tests/
```

Each module references `Infrastructure` only — never another module — so
the boundary is real at the project-reference level even before Phase 10
adds an automated architecture fitness test on top of it.
