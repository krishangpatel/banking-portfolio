# 03 — Non-Functional Requirements

Concrete, measurable targets — numbers, not adjectives. Each is either
verified by an automated test/check (marked **Verified**) or is a stated
design target not continuously enforced by tooling at this scale (marked
**Target**), which is itself an honest, defensible distinction to make in
an interview: "I verify this" vs. "I designed for this and can explain the
gap."

## Correctness & Consistency

- **NFR-1** (Verified) — Under 50 concurrent transfer requests against the
  same source account whose combined amount exceeds the balance, exactly
  the requests that fit succeed; the account balance is never negative at
  any point, including mid-test. Enforced via an xUnit test using
  `Task.WhenAll` against a real SQL Server (Testcontainers), not mocks.
- **NFR-2** (Verified) — A journal entry whose legs do not sum to zero
  cannot be committed. Enforced via a database-level check (constraint or
  transaction-scoped invariant check that throws before commit), tested
  directly against the repository layer, not just through the API.
- **NFR-3** (Verified) — Retrying an identical request with the same
  `Idempotency-Key` within 24 hours never creates a second journal entry,
  regardless of network-level retry count (tested at 1, 2, and 10
  repeats).

## Availability & Reliability

- **NFR-4** (Target) — Single-region deployment; documented target of
  99.5% monthly availability for the demo deployment (≈3.6 hours/month
  downtime budget) — appropriate for a portfolio demo, explicitly *not*
  claimed as production-grade banking availability (real banks target
  99.9%+ with multi-region failover; see [06-component-inventory.md](06-component-inventory.md),
  tier: Acknowledged, for what that would require).
- **NFR-5** (Verified) — Health check endpoint (`/health`) reports
  `Unhealthy` if the database or the message broker is unreachable, and
  the deployment platform's readiness gate uses it before routing traffic
  to a new instance (the Knight Capital lesson from
  [00-exploration.md §1.6](00-exploration.md#16-what-happens-when-banking-software-gets-this-wrong) —
  never route traffic to code that isn't actually ready).
- **NFR-6** (Target) — Outbox relay processes a pending event within 5
  seconds of being written, under normal load (single-instance, demo-scale
  traffic — not a claim about production-scale throughput).

## Performance

- **NFR-7** (Verified, local benchmark) — `POST /transfers` p95 latency
  under 300ms on the local Docker Compose stack, single instance, no
  concurrent load — measured with a lightweight load-test script (k6 or
  `bombardier`), documented as a local baseline, not a production SLA.
- **NFR-8** (Target) — The system is designed to handle up to 50 requests/
  second sustained on a single App Service B1 instance before horizontal
  scaling would be needed — a target derived from B1's resource envelope,
  not independently load-tested at cloud scale (that would burn the
  monthly cloud budget for no interview value).

## Security

- **NFR-9** (Verified) — No endpoint that reads or mutates account,
  transfer, or case data is reachable without authentication; a CI check
  (integration test sweep) asserts every controller action carries an
  explicit `[Authorize]` (or a documented, reviewed `[AllowAnonymous]` —
  only account-opening and login/token endpoints qualify). Anonymous
  access is an explicit opt-in per endpoint, never the default.
- **NFR-10** (Verified) — Passwords/secrets are never logged; a
  CI check greps structured log call sites for known-sensitive field
  names (password, ssn, token) as a lightweight guard, backed by code
  review.
- **NFR-11** (Verified) — No secret (connection string, JWT signing key,
  API key) is committed to source control; local dev uses .NET User
  Secrets / environment variables, cloud deployment uses Azure Key Vault
  or App Service application settings (not files in the repo). Enforced
  by `.gitignore` + a pre-commit/CI secret-scan step.
- **NFR-12** (Target) — All data in transit is over TLS (enforced by the
  hosting platform in the cloud deployment; local dev over HTTP is
  accepted as a documented, standard local-dev exception).

## Auditability & Compliance

- **NFR-13** (Verified) — Every FR-7.1 audit-worthy action produces
  exactly one audit record — not zero (missed), not more than one
  (duplicated) — verified per action type in integration tests.
- **NFR-14** (Target) — Audit and journal data are retained indefinitely
  in this demo (no deletion path exists); a real bank's actual retention
  requirement (commonly 5-7 years depending on record type under BSA
  recordkeeping rules) is documented in
  [06-component-inventory.md](06-component-inventory.md) as the
  Acknowledged real-world target.

## Observability

- **NFR-15** (Verified) — Every request is traceable end-to-end via a
  correlation ID present in every log line and every audit record touched
  during that request; a test asserts the same correlation ID appears in
  the structured logs for a multi-step operation (e.g. a transfer that
  triggers a fraud rule and creates a case).
- **NFR-16** (Target) — Key business metrics (transfers/min, cases
  opened/min, outbox lag) are exported and visible on a dashboard
  (Application Insights in the cloud deployment) within 1 minute of the
  underlying event.

## Cost

- **NFR-17** (Target) — Total cloud hosting cost stays within ~$25-30/
  month while actively running, per [00-exploration.md — Assumptions](00-exploration.md#4-assumptions-i-can-override),
  and can be paused (App Service stopped, SQL Database paused) to near-$0
  between interview cycles. See [04-architecture.md](04-architecture.md)
  cost story and [ADR-0003](adr/0003-deployment-and-hosting.md) for the
  breakdown.

## Maintainability

- **NFR-18** (Verified) — `dotnet test` runs the full unit + integration
  suite in under 5 minutes on CI, so it's actually run on every PR rather
  than skipped for being slow.
- **NFR-19** (Verified) — CI (GitHub Actions) runs build + test + a
  dependency vulnerability check (`dotnet list package --vulnerable`) on
  every pull request; merges to `main` are blocked if any fail.
