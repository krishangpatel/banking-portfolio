# 02 — Functional Requirements

Numbered, testable, grouped by capability. Each requirement is marked
**MVP** (must ship in the Core tier, per [06-component-inventory.md](06-component-inventory.md))
or **Later** (Demonstrated/Acknowledged tier, or explicitly post-portfolio).
Acceptance criteria are written to be directly translatable into
integration tests.

## FR-1: Identity & Account Opening

- **FR-1.1** (MVP) — A new customer can open an account by submitting
  name, date of birth, SSN-last-4 (simulated), and an initial deposit
  amount ≥ $0.
  *Acceptance:* `POST /accounts` with valid payload returns `201` with an
  account in `PendingVerification` status; invalid payload (missing DOB,
  negative deposit) returns `400` with field-level errors.
- **FR-1.2** (MVP) — Account opening runs a simulated CIP (Customer
  Identification Program) check before the account becomes usable.
  *Acceptance:* accounts start `PendingVerification`; a simulated identity
  check transitions the account to `Active` or `Rejected`; no transfer can
  be created against a non-`Active` account (`409`).
- **FR-1.3** (MVP) — Every account has exactly one owner and one USD
  currency; no joint accounts, no multi-currency.
  *Acceptance:* schema-enforced; not user-configurable.
- **FR-1.4** (Later/Acknowledged) — Real identity-verification vendor
  integration (document upload, OCR, liveness check).
  *Acceptance:* N/A — documented interface only, see component inventory.

## FR-2: Ledger & Balances

- **FR-2.1** (MVP) — Every change in an account's balance is recorded as a
  **journal entry** with exactly two legs (one debit, one credit) that sum
  to zero, referencing exactly two accounts (or one account and the
  system's clearing/suspense account for deposits/fees).
  *Acceptance:* attempting to persist a journal entry whose legs don't sum
  to zero, or that has ≠2 legs, fails at the data layer (DB constraint /
  transaction rejection), not just in application code.
- **FR-2.2** (MVP) — An account's balance is always derivable as the sum
  of its posted journal legs, and the stored/cached balance never
  disagrees with that sum.
  *Acceptance:* a test posts N concurrent journal entries against one
  account and asserts the stored balance equals `SUM(legs)` with zero
  drift after all complete.
- **FR-2.3** (MVP) — Journal entries are immutable once posted; corrections
  are made by posting a new, opposite entry (a reversal), never by
  editing or deleting history.
  *Acceptance:* no `UPDATE`/`DELETE` code path exists for posted journal
  rows; a reversal-posting endpoint exists and is the only correction
  mechanism.
- **FR-2.4** (MVP) — A customer or authorized ops user can fetch an
  account's current balance and a paginated statement (journal history).
  *Acceptance:* `GET /accounts/{id}/balance`, `GET /accounts/{id}/statement?cursor=`.

## FR-3: Transfers (Money Movement)

- **FR-3.1** (MVP) — A transfer between two `Active` accounts requires a
  client-supplied idempotency key; repeating the same request with the
  same key returns the original result without re-executing the transfer.
  *Acceptance:* two identical `POST /transfers` calls with the same
  `Idempotency-Key` header produce exactly one journal entry and both
  responses match.
- **FR-3.2** (MVP) — A transfer that would overdraw a source account
  (balance would go negative) is rejected atomically; no partial posting
  is ever observable.
  *Acceptance:* concurrent transfer test — two transfers that together
  exceed the source balance, submitted concurrently, result in exactly one
  succeeding and one rejected with `409`, and the balance never goes
  negative at any point.
- **FR-3.3** (MVP) — A transfer's request/response shape uses ISO
  20022-inspired field names (`debtor`, `creditor`, `remittanceInformation`,
  `endToEndId`) — see [00-exploration.md §1.5](00-exploration.md#15-real-time-payments-and-message-standards-context-not-a-build-target).
  *Acceptance:* schema review / OpenAPI spec inspection.
- **FR-3.4** (MVP) — A transfer flagged by the fraud/AML rules engine (see
  FR-4) is held (`PendingReview` status, funds earmarked but not yet
  posted to the destination) rather than posted immediately, and is
  resolved (cleared or reversed) by the saga described in
  [04-architecture.md](04-architecture.md).
  *Acceptance:* a transfer matching a configured rule (e.g. amount over
  threshold) lands in `PendingReview`, not `Posted`; a compliance-role user
  resolving the case transitions it to `Posted` or `Reversed`.
- **FR-3.5** (Later/Demonstrated) — Scheduled/recurring transfers.
  *Acceptance:* out of MVP; documented as a straightforward extension of
  FR-3.1 in the build plan if time allows.

## FR-4: Fraud & AML Transaction Monitoring

- **FR-4.1** (MVP) — A configurable, named, versioned set of rules
  evaluates every transfer synchronously before it posts (e.g. amount >
  $10,000 single transfer, > $10,000 aggregate in 24h to a new payee,
  first transfer to a brand-new payee over $2,500).
  *Acceptance:* each rule is unit-tested independently against
  fixture transactions with known expected outcomes.
- **FR-4.2** (MVP) — A transfer matched by any rule creates a **case** in
  a review queue, visible only to the `ComplianceOfficer` role, with the
  triggering rule(s) named explicitly (never just a numeric score), and a
  `dueBy` timestamp set 24 hours from creation so an unresolved case is a
  visible, alertable condition rather than funds held indefinitely with no
  signal.
  *Acceptance:* `GET /compliance/cases` (role-gated, `403` for other
  roles) lists cases with rule name, transaction, status, and `dueBy`; a
  case past `dueBy` and still `Open` is surfaced by the case-aging metric
  in [03-non-functional-requirements.md — NFR-16](03-non-functional-requirements.md).
- **FR-4.3** (MVP) — A `ComplianceOfficer` can resolve a case as
  `Cleared` (transfer posts) or `Rejected` (transfer reverses, funds
  return to source).
  *Acceptance:* both resolution paths tested end-to-end including the
  resulting ledger state.
- **FR-4.4** (Later/Acknowledged) — ML-based anomaly scoring alongside the
  rules engine.
  *Acceptance:* N/A — documented as the natural next layer, not built (see
  [00-exploration.md §1.4](00-exploration.md#14-regulatory-reality-us-focused-since-the-job-search-is-nj-based)).
- **FR-4.5** (Later/Demonstrated) — Suspicious Activity Report (SAR)-style
  CSV export for a resolved case.
  *Acceptance:* `GET /compliance/cases/{id}/export` returns a CSV with the
  fields a real SAR narrative would need (parties, amounts, dates, rule
  triggered, resolution).

## FR-5: Reconciliation

- **FR-5.1** (Later/Demonstrated) — A scheduled job compares the internal
  ledger's total posted balance against a simulated external statement
  feed (a fixture representing "what the correspondent bank says our
  balance is") and raises a discrepancy case if they don't match.
  *Acceptance:* seeding a deliberate mismatch causes the next
  reconciliation run to create a case within one run cycle; a matching
  run creates no case.

## FR-6: Access Control

- **FR-6.1** (MVP) — Three roles exist: `Customer` (own accounts/transfers
  only), `SupportOps` (read-only across accounts, no transfer creation),
  `ComplianceOfficer` (case queue access, no direct transfer creation).
  *Acceptance:* role-based authorization tested per endpoint — every
  endpoint has at least one "should be denied" test per role that
  shouldn't have access.
- **FR-6.2** (MVP) — A `Customer` can only read/act on accounts they own;
  attempting to access another customer's account returns `404`, not
  `403` — a `403` would confirm the account exists to an unauthorized
  caller (account-ID enumeration), which the API deliberately avoids.
  *Acceptance:* cross-customer access attempt returns `404`.

## FR-7: Audit Trail

- **FR-7.1** (MVP) — Every state-changing action (account opened, transfer
  posted/held/reversed, case created/resolved, role change) is recorded in
  an audit log distinct from application logs, capturing who, what, when,
  and the before/after state where applicable.
  *Acceptance:* an integration test performs one of each action type and
  asserts a corresponding audit record exists with the correct actor.
- **FR-7.2** (MVP) — Audit records are append-only; no code path updates
  or deletes an audit record.
  *Acceptance:* code review / no `UPDATE`/`DELETE` on the audit table.
