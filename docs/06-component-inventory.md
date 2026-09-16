# 06 — Enterprise Component Inventory

An exhaustive inventory of what a complete enterprise application contains,
built from seven personas, then merged and deduped. Every item states what
it is, why *this* domain needs it, how it's realized in this stack, where a
reviewer finds it in the repo, which interview questions it preps, and its
tier:

- **Core** — fully built and tested. This is the load-bearing majority of
  the interview story.
- **Demonstrated** — a representative slice, built properly (not stubbed),
  proving the pattern is understood and executable, without building every
  variant a full production system would eventually need.
- **Acknowledged** — documented here with how it would be done in a real
  bank's system, deliberately not built. Assume Krishang will be asked "why
  didn't you include X?" for anything in this tier — the answer is written
  down so it doesn't have to be improvised.

## Persona pass (raw, pre-merge)

Each persona was asked: *"What would you expect to find here, and what
would worry you if it were missing?"* Answers below feed the merged
inventory in §2; nothing here is repeated verbatim there.

- **Principal engineer at a bank:** module boundaries that are actually
  enforced, not just diagrammed; a ledger that can't drift; a migration
  story that doesn't require downtime or a leap of faith; ADRs that show
  rejected alternatives, not just the chosen one; a clear story for what
  happens when a downstream dependency (Service Bus, a partner API) is
  slow or down.
- **Security reviewer:** authn/authz on every endpoint, not just the
  obvious ones; no secrets in source control or logs; input validation at
  the boundary; dependency vulnerability scanning in CI; a documented
  threat model, even a lightweight one; awareness of OWASP-class issues
  (injection, broken access control, sensitive data exposure) with a
  stated mitigation per class, not silence.
- **SRE / on-call engineer:** health checks that actually gate traffic;
  structured logs with correlation IDs; metrics and a dashboard; a
  rollback story; graceful shutdown (in-flight requests aren't dropped on
  deploy); a runbook for "the outbox is backing up" or "reconciliation
  found a discrepancy" even if there's no 2 a.m. pager in a portfolio demo.
- **Compliance auditor:** an audit trail that's append-only and covers who
  /what/when; segregation of duties (a customer can't approve their own
  flagged transfer); documented data retention; a KYC/CIP step, even
  simulated; evidence that "compliance" isn't just a folder of policy
  documents disconnected from what the code actually does.
- **QA lead:** a real test pyramid (unit for domain invariants, integration
  against a real database, not all-mocks); tests that exercise concurrency
  and failure, not just the happy path; CI gates that actually block a bad
  merge; a definition of "done" per build-plan phase that includes tests
  and docs, not just code.
- **Product owner:** numbered, testable requirements with acceptance
  criteria; an honest scope story (what's MVP, what's later, what's
  deliberately never); a README that lets a new reader understand the
  system's value in minutes, not hours.
- **New hire onboarding:** one command to run the whole thing locally; a
  glossary for domain terms (what's a "leg," a "case," a "hold"); ADRs
  explaining *why*, so a new engineer doesn't have to reverse-engineer
  intent from code; a repo structure that maps predictably to the
  architecture doc.

## 2. Merged inventory

### 2.1 Domain correctness & the ledger

| Item | Why this domain needs it | How it's realized | Where in repo | Interview questions it preps | Tier |
|---|---|---|---|---|---|
| Double-entry journal with a zero-sum invariant | The #1 thing that separates a real ledger from a toy one (see [00-exploration.md §1.1](00-exploration.md#11-how-real-ledgercore-banking-systems-are-built)) | Append-only `JournalEntries`/`JournalLegs`, invariant enforced at the data layer | `src/Ledgerline.Ledger/` | "How do you guarantee a ledger never drifts?" "Why not a single balance column?" | **Core** |
| Balance projection consistent with journal | Fast reads without trusting a second, independently-mutable source of truth | Projection updated in the same DB transaction as the journal write | `src/Ledgerline.Ledger/` | "How do you keep a cached balance from lying?" | **Core** |
| Idempotent money-moving endpoints | H1 — client retries must not double-move money | Idempotency-Key header + dedupe table keyed on key+payload hash | `src/Ledgerline.Transfers/` | "Walk me through what happens if a client retries a timed-out transfer." | **Core** |
| Concurrency control on account balances | H2 — two transfers racing the same account | Row-level locking / optimistic concurrency inside the posting transaction | `src/Ledgerline.Ledger/` | "What happens when two transfers hit the same account at the same instant?" | **Core** |
| Transactional outbox + relay | H4 — partial failure between a domain write and its async side effects | Outbox table written in the same transaction; `IHostedService` relay | `src/Ledgerline.Infrastructure/Outbox/` | "How do you avoid losing an event, or double-publishing one, on a crash?" | **Core** |
| Saga for held/reviewed transfers | A flow spanning human-scale duration needs compensation, not a transaction | Orchestration-style saga: hold → review → clear/reverse | `src/Ledgerline.Transfers/Saga/` | "Why a saga here and a plain transaction everywhere else?" | **Core** |
| Reconciliation against an external source of truth | H6 — a ledger that's never checked against reality is trusted on faith | Scheduled job diffing ledger totals vs. a simulated statement feed, raising a case on mismatch | `src/Ledgerline.Reconciliation/` | "How would you know if the ledger and the bank's actual balance disagreed?" | **Demonstrated** |
| Full event sourcing (event store, replay, snapshotting) | Real fintechs sometimes go further than append-only-journal-plus-projection | N/A — documented alternative | [ADR-0006](adr/0006-ledger-consistency-model.md) | "Why not full event sourcing?" | **Acknowledged** |

### 2.2 Compliance & fraud

| Item | Why this domain needs it | How it's realized | Where in repo | Interview questions it preps | Tier |
|---|---|---|---|---|---|
| Simulated CIP/KYC gate at account opening | BSA requires identity verification before an account is usable | Account starts `PendingVerification`; simulated provider call transitions it | `src/Ledgerline.Accounts/` | "How would a real KYC vendor plug into this?" | **Core** |
| Rules-based transaction monitoring | Regulators require documented, inspectable monitoring logic (not an opaque score) | `IFraudRule` strategy implementations, each independently testable | `src/Ledgerline.Fraud/Rules/` | "Why rules instead of an ML score? What are the limits of that?" | **Core** |
| Compliance case queue + resolution workflow | Segregation of duties — a flagged transfer needs a second, differently-privileged party to decide | `ComplianceOfficer`-only case endpoints; resolve→saga compensation | `src/Ledgerline.Compliance/` | "Who can clear a flagged transfer, and why not the customer?" | **Core** |
| SAR-style case export | Real compliance officers file structured reports on resolved cases | CSV export with the fields a SAR narrative needs | `src/Ledgerline.Compliance/Export/` | "What would a real regulatory filing need beyond this CSV?" | **Demonstrated** |
| ML-based anomaly scoring alongside rules | Real fraud stacks run rules + ML together (rules catch known patterns, ML catches drift) | N/A — documented as the natural next layer | [00-exploration.md §1.4](00-exploration.md#14-regulatory-reality-us-focused-since-the-job-search-is-nj-based) | "Would you add ML scoring? What would that take?" | **Acknowledged** |
| Real KYC vendor integration (document/OCR/liveness) | A real bank can't self-attest identity | N/A — interface documented (`IIdentityVerificationProvider`), fixture-backed impl only | [04-architecture.md §1](04-architecture.md#1-system-context-c4-level-1) | "What would swapping in a real Persona/Alloy integration involve?" | **Acknowledged** |
| Real payment rail integration (FedNow/RTP/ACH) | Requires bank sponsorship, unreachable for a solo project | N/A — message shape borrows ISO 20022 naming, no rail connection | [00-exploration.md §1.5](00-exploration.md#15-real-time-payments-and-message-standards-context-not-a-build-target) | "Is this connected to a real payment network?" (answer: no, explicitly) | **Acknowledged** |
| Data retention policy documentation | BSA recordkeeping rules require multi-year retention of specific record types | Documented target (5-7 years) vs. demo behavior (indefinite, no deletion path) | [03-non-functional-requirements.md — NFR-14](03-non-functional-requirements.md) | "How long would you actually have to keep this data, and why?" | **Acknowledged** (policy documented; enforcement mechanism not built) |
| Multi-currency / FX | Real banks operate across currencies | N/A | [00-exploration.md §4](00-exploration.md#4-assumptions-i-can-override) | "How would you add a second currency?" | **Acknowledged** |

### 2.3 Security

| Item | Why this domain needs it | How it's realized | Where in repo | Interview questions it preps | Tier |
|---|---|---|---|---|---|
| Authentication (JWT bearer) | Every non-account-opening endpoint touches money or PII | ASP.NET Core Identity issues tokens; JWT bearer middleware | `src/Ledgerline.Api/Auth/` | "Walk me through a request's auth from token to handler." | **Core** |
| Role-based authorization (Customer/SupportOps/ComplianceOfficer) | Segregation of duties (FR-6, compliance) | Policy-based `[Authorize]` per endpoint, tested per role | `src/Ledgerline.Api/Auth/` | "How do you know an endpoint isn't accidentally open?" | **Core** |
| Anti-enumeration on cross-account access | Prevent account-ID probing revealing which IDs are valid | `404`, not `403`, for cross-customer access attempts | `src/Ledgerline.Accounts/` | "Why 404 instead of 403 for someone else's account?" | **Core** |
| No secrets in source control | A committed connection string or signing key is a full compromise | `.gitignore` + user-secrets/env locally, Key Vault in cloud, CI secret-scan | `.gitignore`, [ADR-0003](adr/0003-deployment-and-hosting.md) | "Where do your secrets live, in each environment?" | **Core** |
| Dependency vulnerability scanning in CI | A banking API is a high-value target for a known-CVE dependency exploit | `dotnet list package --vulnerable` gate + Dependabot | `.github/workflows/` | "How would you know if a dependency had a new CVE?" | **Core** |
| Input validation at the API boundary | Broken/missing validation is the #1 OWASP-class risk for a money-moving API | FluentValidation on every request DTO, tested for boundary cases | `src/Ledgerline.Api/Validation/` | "What happens if I send a negative transfer amount?" | **Core** |
| Parameterized queries (SQL injection defense) | Classic, still-relevant risk for anything touching money | EF Core parameterizes by default; no raw string-concatenated SQL anywhere in the codebase | `src/Ledgerline.*/Data/` | "How do you prevent SQL injection here?" | **Core** |
| Rate limiting on public endpoints | Prevents brute-force and abuse of account-opening/login | ASP.NET Core built-in rate limiter middleware, applied to auth + account-opening | `src/Ledgerline.Api/` | "How would you stop someone hammering the login endpoint?" | **Demonstrated** |
| Security headers (HSTS, CSP-equivalent for an API, X-Content-Type-Options) | Standard defense-in-depth, cheap to add | Middleware, tested via integration test asserting header presence | `src/Ledgerline.Api/` | "What headers does your API set and why?" | **Demonstrated** |
| Lightweight threat model | A security reviewer expects evidence threats were considered, not just controls applied ad hoc | STRIDE-style pass over the transfer and case-review flows | [07-red-team.md](07-red-team.md) | "Walk me through your threat model for the transfer endpoint." | **Demonstrated** |
| Formal penetration test / third-party security audit | Real banks are required to have these periodically | N/A | — | "Has this been pen-tested?" (answer: no, explicitly, and why that's normal for a portfolio project) | **Acknowledged** |
| SOC 2 / PCI-DSS certification process | Real regulatory/industry certification | N/A | — | "What would SOC 2 readiness require here?" | **Acknowledged** |
| Secrets rotation automation | Real production systems rotate signing keys/credentials on a schedule | N/A — manual rotation documented as a runbook step | — | "How would you rotate the JWT signing key without an outage?" | **Acknowledged** |

### 2.4 Observability & SRE

| Item | Why this domain needs it | How it's realized | Where in repo | Interview questions it preps | Tier |
|---|---|---|---|---|---|
| Structured logging with correlation IDs | H7/NFR-15 — trace one request across every module it touches | Serilog, correlation ID middleware, enriched on every log line | `src/Ledgerline.Api/` | "How would you debug a transfer a customer says failed?" | **Core** |
| Health checks gating traffic | H9 — never route traffic to an unready instance (the Knight Capital lesson) | `/health` aggregates DB + Service Bus reachability; deployment readiness probe uses it | `src/Ledgerline.Api/HealthChecks/` | "What does your health check actually verify?" | **Core** |
| Metrics + dashboard (transfers/min, outbox lag, cases opened) | NFR-16 — operational visibility into the system's core flows | OpenTelemetry metrics exported to Application Insights (cloud) | `src/Ledgerline.Infrastructure/Telemetry/` | "How would you know if the outbox relay stopped keeping up?" | **Demonstrated** |
| Distributed tracing | Follow a request across the API → outbox → async consumer boundary | OpenTelemetry tracing, same correlation ID as logs | `src/Ledgerline.Infrastructure/Telemetry/` | "How do you trace an async side effect back to its originating request?" | **Demonstrated** |
| Graceful shutdown | In-flight requests/DB transactions shouldn't be dropped on deploy | ASP.NET Core's built-in shutdown hooks + a documented drain window | `src/Ledgerline.Api/Program.cs` | "What happens to an in-flight transfer during a deploy?" | **Demonstrated** |
| Runbooks (outbox backlog, reconciliation discrepancy, failed deploy) | An SRE expects a documented first response, not tribal knowledge | Markdown runbooks | `docs/runbooks/` | "Walk me through what you'd do at 2 a.m. if the outbox started backing up." | **Demonstrated** |
| Backup/restore drill | Data loss in a ledger is catastrophic and irreversible | Documented Azure SQL point-in-time-restore process, exercised once locally against a restored container | `docs/runbooks/backup-restore.md` | "How do you recover from an accidental bad migration?" | **Demonstrated** |
| On-call rotation / paging integration | Real production systems page a human | N/A — solo portfolio project has no on-call rotation to build | — | "How would this fit into an on-call rotation?" | **Acknowledged** |
| Chaos engineering (fault injection at scale) | Real banks test resilience by breaking things on purpose | N/A — failure modes reasoned about and unit/integration-tested individually instead (see [04-architecture.md §6](04-architecture.md#6-failure-modes)) | — | "Have you run chaos experiments against this?" (answer: no — explain the cheaper alternative used instead) | **Acknowledged** |
| Multi-region / disaster recovery | Real banks can't have a single region be a single point of failure | N/A | [04-architecture.md §7](04-architecture.md#7-scaling-story) | "What's your DR story?" | **Acknowledged** |

### 2.5 Quality engineering

| Item | Why this domain needs it | How it's realized | Where in repo | Interview questions it preps | Tier |
|---|---|---|---|---|---|
| Unit tests for domain invariants | The ledger/idempotency/rules logic is where correctness bugs are most expensive | xUnit, no mocking of the domain model itself | `tests/Ledgerline.Domain.Tests/` | "What's your unit test strategy for the ledger?" | **Core** |
| Integration tests against a real database | Mocked-DB tests wouldn't catch the concurrency/constraint behavior this system depends on | Testcontainers (real SQL Server in Docker), per NFR-1/NFR-2 | `tests/Ledgerline.Integration.Tests/` | "Why real SQL Server instead of an in-memory provider?" | **Core** |
| Concurrency/failure-injection tests | H2 — the whole point of the invariant is that it holds under concurrency, which only a concurrency test proves | `Task.WhenAll` race tests against the same account | `tests/Ledgerline.Integration.Tests/` | "Show me the test that proves two simultaneous transfers can't overdraw an account." | **Core** |
| CI gate (build, test, vuln scan) blocking bad merges | QA/compliance expect merges to be gated, not just tested locally | GitHub Actions on every PR | `.github/workflows/ci.yml` | "What stops a broken change from reaching main?" | **Core** |
| Architecture fitness test (module boundary enforcement) | The modular-monolith boundary is only real if something checks it | A test asserting no module references another module's internal namespace directly | `tests/Ledgerline.Architecture.Tests/` | "How do you keep the module boundaries from eroding over time?" | **Demonstrated** |
| Local load test (k6/bombardier) | NFR-7 needs a real measurement, not a guess | Script + documented results against the local Docker Compose stack | `tests/load/` | "What's your actual measured latency, and how did you measure it?" | **Demonstrated** |
| Contract tests between API and consumers | Prevents an API and consumer schema from silently drifting | N/A — single-team, single-repo project has no external consumer yet | — | "How would you contract-test this API for external consumers?" | **Acknowledged** |
| Formal test-data management platform | Large orgs need synthetic-data generation/masking pipelines | N/A — fixtures/seed scripts suffice at this scale | `tests/Fixtures/` | "How would test data management change at bank scale?" | **Acknowledged** |

### 2.6 Developer experience & documentation

| Item | Why this domain needs it | How it's realized | Where in repo | Interview questions it preps | Tier |
|---|---|---|---|---|---|
| One-command local run | Hard Rule 8 — any reviewer must be able to clone and run it | `docker compose up`, documented in the README | `docker-compose.yml`, `README.md` | "How do I run this?" (should never need a follow-up question) | **Core** |
| README (10-minute understanding) | The recruiter/hiring-manager audience never gets past the README if it isn't clear | Problem statement, architecture diagram, quickstart, what it is/isn't | `README.md` | — | **Core** |
| ADRs for every significant decision | A principal engineer/new hire needs *why*, not just *what* | One file per decision under `docs/adr/` | `docs/adr/` | "Why did you choose X over Y?" — answer is already written | **Core** |
| OpenAPI/Swagger + Postman/Bruno collection | The "see it work" surface, given the no-UI decision | Swagger UI at `/swagger`, exported collection in repo | `docs/api/` | "Can I try this without reading the code?" | **Core** |
| Domain glossary | New-hire persona — "leg," "case," "hold," "posting" aren't self-evident terms | Glossary section in README or a dedicated doc | `docs/glossary.md` | — | **Demonstrated** |
| Seed/demo data script | A reviewer shouldn't have to manually create accounts to see the system work | A seed script creating a handful of accounts, one flagged-transfer scenario, one reconciliation-mismatch scenario | `scripts/seed-demo-data.sh` | — | **Demonstrated** |
| Internal style guide / contribution conventions | Large orgs standardize this across many contributors | N/A — solo project, conventions documented inline in ADRs instead of a separate CONTRIBUTING doc | — | "How would you onboard a second engineer onto this?" | **Acknowledged** |

### 2.7 Platform & deployment

| Item | Why this domain needs it | How it's realized | Where in repo | Interview questions it preps | Tier |
|---|---|---|---|---|---|
| EF Core migrations, reviewed and CI-applied | H8 — the TSB lesson: schema change is a first-class, tested artifact | Migrations committed, applied via CI/CD step, never run ad hoc | `src/Ledgerline.Infrastructure/Migrations/` | "How does a schema change reach production here?" | **Core** |
| Containerized local environment | Consistent dev environment, matches Hard Rule 8 | Docker Compose (API, SQL Server, Service Bus emulator) | `docker-compose.yml` | — | **Core** |
| Cloud deployment (Azure App Service + SQL + Service Bus + App Insights) | A live, linkable demo for recruiters, within budget | See [04-architecture.md §8](04-architecture.md#8-cost-story), [ADR-0003](adr/0003-deployment-and-hosting.md) | `infra/` (Bicep/ARM or `azd` config) | "Where does this actually run?" | **Demonstrated** |
| Infrastructure as code | Reproducible environments, standard enterprise practice | Bicep templates for the Azure resources | `infra/` | "How would you recreate this environment from scratch?" | **Demonstrated** |
| API versioning scheme | Real APIs evolve without breaking existing clients | URL-based `v1` prefix from day one; policy for `v2` documented | `src/Ledgerline.Api/` | "How would you ship a breaking change without breaking existing clients?" | **Demonstrated** |
| Kubernetes / container orchestration | Large-scale deployment platform | N/A — a single App Service instance is the right scale for this system; documented as the next step *if* scale demanded it | [04-architecture.md §7](04-architecture.md#7-scaling-story) | "Why not Kubernetes?" | **Acknowledged** |
| Multi-region active-active deployment | Real banks need regional failover | N/A | [04-architecture.md §7](04-architecture.md#7-scaling-story) | "What's missing for this to survive a region outage?" | **Acknowledged** |
| Feature flag system | Large orgs decouple deploy from release | N/A — a solo project with one consumer of its own changes doesn't need this yet | — | "How would you add feature flags here?" | **Acknowledged** |

## 3. Tier summary

| Tier | Count | What it signals |
|---|---|---|
| Core | 24 | Fully built, fully tested — the substance of the interview story |
| Demonstrated | 15 | Built properly as a representative slice — proves the pattern is understood and executable |
| Acknowledged | 20 | Documented, deliberately not built — proves the boundary of solo/timeline-appropriate scope is a judgment call, not an oversight |

This distribution is itself a talking point: roughly 40% Core, 25%
Demonstrated, 35% Acknowledged. A portfolio project that was 100% Core would
either be far too small to be impressive, or (more likely) would be
quietly cutting corners on testing/completeness to fake breadth. A project
that was mostly Acknowledged would be all claims and no substance. This
ratio is the actual evidence of judgment the brief asks this repo to
demonstrate.
