# 00 — Exploration

Phase 1 research notes: how real banking/fintech systems are built and
regulated, the hard problems that separate a toy banking app from something
a bank engineer would respect, the candidate concepts generated and scored
against them, the recommendation, and the assumptions this design rests on.

---

## 1. Domain research

Sources are linked inline. This is a synthesis, not a literature dump — each
point below fed directly into a design decision later in this doc set.

### 1.1 How real ledger/core-banking systems are built

- Modern cores separate the **general ledger (GL)**, which enforces
  double-entry at the system level, from the **customer-facing account
  layer**, which aggregates and presents balances. The GL is the source of
  truth; balances are a derived/projected view, never a mutable field
  updated in place. ([Formance — Core Banking System Architecture](https://www.formance.com/blog/financial-operations/core-banking-system-architecture))
- The failure modes explicitly called out by practitioners: storing a
  single mutable "balance" column as the source of truth (so a crash mid
  -update loses money), and allowing one side of a journal entry to post
  without the other (so debits and credits drift out of balance). A correct
  ledger makes both of these structurally impossible, not just
  "checked for" — e.g. a single journal-entry write that always contains
  both legs, and a CHECK/invariant that a journal never nets to non-zero.
  ([Framnex — Real-Time Ledger with Double-Entry](https://framnex.com/blog/guides/designing-a-real-time-ledger-system-with-double-entry-logic))
- Reporting/analytics is deliberately run off a **read replica or a
  CDC-fed warehouse**, not the primary OLTP path, so analytical queries
  never contend with (or slow down) real money movement. This is the same
  shape as CQRS without requiring full event sourcing.
  ([Crassula — Banking Database Design 2026](https://crassula.io/guides/banking-database-design/))
- "Composable banking" (2026 framing): KYC, ledger, payments, and
  onboarding as independently deployable capabilities behind APIs. Useful
  as a target *shape* even inside a single deployable — it argues for
  strong module boundaries now, not a rewrite later. ([Crassula — Core
  Banking Systems 2026](https://crassula.io/guides/core-banking/))

### 1.2 Idempotency and exactly-once money movement

- Stripe's idempotency system centers on an `Idempotency-Key` request
  header: the server persists the key with the result of the first
  successful attempt, and any retry with the same key returns that same
  result rather than re-executing the side effect. This is *the* mechanism
  that makes "retry on timeout" safe for a money-moving API. ([Mayank Raj —
  Idempotency Keys: Why Retrying Without Them Is Financial Suicide](https://mayankraj.com/blog/idempotency-keys-exactly-once-payments/), [Tanuj
  Garg — Payment Gateway Architecture](https://tanujgarg.com/blog/payment-gateway-idempotency-ledger-design))
- Monzo's international payments processing performs explicit checks so a
  payment is applied to an account **once and exactly once**, leaning
  heavily on idempotency in the decision layer, not just at the HTTP
  boundary. ([Monzo — Building a Processing System for International
  Payments](https://monzo.com/blog/building-a-processing-system-for-international-payments))
- Airbnb's payments team documents the general pattern: a client-generated
  idempotency key, a dedupe table keyed on it, and the guarantee that a
  duplicate request short-circuits to the original result instead of
  re-charging. ([Airbnb Engineering — Avoiding Double Payments in a
  Distributed Payments System](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb))
- **Design implication:** every money-moving endpoint in this system takes
  a client-supplied idempotency key; there is a dedupe table; retries are
  safe by construction, not by convention.

### 1.3 Event-driven ledgers, CQRS, and reconciliation

- CQRS is commonly paired with event sourcing in fintech specifically
  because the write path (strict, auditable, append-only) has very
  different requirements from the read path (fast, denormalized, can lag
  slightly). Separating them lets each be optimized without compromising
  the other. ([Hedge Think — Fintech Compliance-First Architecture](https://www.hedgethink.com/fintech-software-development-a-compliance-first-architecture-guide/), [Icon Solutions — CQRS & Event Sourcing in
  Financial Services](https://iconsolutions.com/blog/cqrs-event-sourcing))
- Streaming reconciliation pipelines continuously diff internal ledger
  state against an external system of record (a processor, a correspondent
  bank statement, a card network file) and raise a discrepancy workflow
  the moment they diverge, rather than waiting for a nightly batch.
  ([Oceanobe — Event-Driven Ledger Architectures](https://oceanobe.com/news/event-driven-ledger-architectures/1872))
- **Design implication:** we adopt the *shape* of event sourcing (an
  immutable, appended journal is the system of record; balances are a
  projection) without committing to full event-sourcing infrastructure
  (event store, snapshotting, replay tooling) — see [ADR-0006](adr/0006-ledger-consistency-model.md)
  for why the heavier version is rejected as scope that wouldn't pay for
  itself in 2-4 weeks.

### 1.4 Regulatory reality (US-focused, since the job search is NJ-based)

- The Bank Secrecy Act (BSA), implemented through FinCEN regulations,
  requires a **Customer Identification Program (CIP)** at onboarding,
  ongoing **transaction monitoring**, **suspicious activity reporting**,
  recordkeeping, and a designated compliance officer. KYC (identity at
  onboarding) and AML (the broader ongoing program) are related but
  distinct obligations. ([Treasury Prime — Fintech AML/BSA Policy
  Requirements](https://www.treasuryprime.com/blog/bsa-aml-policy-requirements), [Lithic — Fintech AML Law
  Requirements](https://www.lithic.com/blog/bsa-aml))
- Transaction monitoring in practice is a **rules engine over a risk
  score**: defined thresholds, documented logic, and either automatic
  action or an analyst queue for anything the rules flag. Modern stacks
  run rules *and* ML together — rules for explainable, auditable
  thresholds a regulator can inspect; ML for behavioral drift rules can't
  anticipate. ([Unit21 — Fraud Rules Engine](https://www.unit21.ai/fraud-aml-dictionary/fraud-rules-engine), [Trio — Fraud Detection
  System Architecture](https://trio.dev/fraud-detection-system-architeture-fintech/))
- **Design implication:** compliance is not a bolt-on. The system needs an
  identity/CIP step at account opening, a rules-based transaction
  monitoring slice with documented, inspectable logic, and a case
  workflow for anything flagged — sized to what one engineer can build
  credibly (a real rules engine with a handful of real rules, not a
  fake one; ML scoring is explicitly out of scope and acknowledged, not
  built — see [06-component-inventory.md](06-component-inventory.md)).

### 1.5 Real-time payments and message standards (context, not a build target)

- FedNow (Federal Reserve) and RTP (The Clearing House) are the two US
  instant-payment rails in 2026, both settling in seconds and both
  clearing in **ISO 20022**, a structured message format carrying far more
  data per payment than legacy formats (roughly 10x). Combined they reach
  about 75% of US bank accounts. ([FRB Services — FedNow and ISO
  20022](https://www.frbservices.org/financial-services/fednow/what-is-iso-20022-why-does-it-matter), [Corpay — RTP vs
  FedNow 2026](https://www.corpay.com/resources/blog/real-time-payments))
- **Design implication:** we are not connecting to a real payment rail
  (that requires bank sponsorship and is out of reach for a solo
  portfolio project) but the **internal transfer API's message shape and
  the outbox event schema deliberately borrow ISO 20022 field-naming
  conventions** (e.g. `debtor`, `creditor`, `remittanceInformation`) so the
  design reads as rail-aware, not naive. This is called out explicitly in
  interview prep so it isn't mistaken for a claim that we implemented
  FedNow.

### 1.6 What happens when banking software gets this wrong

- **Knight Capital (2012):** a botched deploy left old test code live in
  production; the trading algorithm executed unintended trades for 45
  minutes, costing $440M and ending the company. The root cause was a
  **deployment process** that could not guarantee old code was fully
  retired, not a logic bug per se. ([Yegor Sychev — Biggest Software
  Failures in Financial/Banking Sector](https://medium.com/@yegor-sychev/the-biggest-software-failures-in-the-financial-and-banking-sector-fde6a0107a53))
- **TSB Bank (2018):** a core banking migration of ~5M accounts went live
  without adequate production-scale rehearsal; ~2M customers were locked
  out, and the incident ultimately ended with TSB outsourcing its banking
  platform entirely. Root causes cited: insufficient testing at real
  scale, an unrealistic timeline inherited from a dissimilar prior
  migration, and data-migration testing that didn't replicate production.
  ([Panorama Consulting — 4 Lessons from TSB](https://www.panorama-consulting.com/tsb-software-failure/), [IceDQ — TSB
  Data Migration Failure](https://icedq.com/resources/case-studies/tsb-bank-data-migration-failure))
- **Design implication:** this project can't rehearse a 5M-account
  migration, but it takes the underlying lesson seriously: **migrations
  and deploys are first-class, tested artifacts** (EF Core migrations are
  reviewed and applied through CI, never run ad hoc against production;
  the build plan includes a rollback story), and **testing must run
  against data volumes and failure injection that resemble the real
  invariant being protected** (concurrent transfers on the same account,
  not just the happy path).

---

## 2. Hard problems this design must confront head-on

Generated from the research above. Each one maps to a specific mechanism in
[04-architecture.md](04-architecture.md) — none of these are allowed to be
"solved" only in prose.

| # | Hard problem | Where it's confronted |
|---|---|---|
| H1 | Exactly-once money movement under client retries | Idempotency-Key + dedupe table on every money-moving endpoint |
| H2 | Ledger correctness under concurrency (two transfers hitting the same account at once) | Double-entry journal with a DB-level invariant + row-level locking / optimistic concurrency on account balance projection |
| H3 | Immutable, auditable history vs. fast balance reads | Append-only journal is the source of truth; balance is a transactionally-consistent projection, not a separately-trusted field |
| H4 | Partial failure across module boundaries (post to ledger, but fraud check or notification fails) | Transactional outbox + async relay; saga for the hold-then-release flow on risk-flagged transfers |
| H5 | Regulatory transaction monitoring with inspectable, documented logic | Rules-based fraud/AML engine with named, versioned rules and a case queue, not an opaque score |
| H6 | Reconciliation against an external source of truth | Scheduled reconciliation job diffing ledger state against a simulated external statement feed, raising discrepancies as cases |
| H7 | Auditability — who did what, when, and why, forever | Immutable journal + structured audit log distinct from application logs, retained separately from operational data |
| H8 | Safe, rehearsed schema change (the TSB lesson) | EF Core migrations reviewed in PRs, applied via CI, never ad hoc; migration dry-runs in the build plan |
| H9 | Deploys that can't leave stale/mixed code running (the Knight Capital lesson) | Single deployable modular monolith (no partial-deploy skew across services), health checks gating traffic cutover |
| H10 | Least-privilege access to money-moving and PII data | Role-based auth (Customer / Support-Ops / Compliance) enforced at the API and query level, not just UI-hidden |

---

## 3. Candidate concepts (12 generated)

1. **Core Ledger & Internal Payments Platform** — accounts, double-entry
   ledger, idempotent transfers between accounts, a fraud/AML slice, audit
   trail.
2. **Loan Origination & Servicing Platform** — application → underwriting
   → amortization schedule → payment servicing state machine.
3. **Fraud & AML Transaction Monitoring System** (standalone) — rules
   engine + case management over a stream of synthetic transactions.
4. **Open Banking / Account Aggregation Platform** — OAuth2 consent flows,
   read-only account/transaction APIs across simulated "linked" banks.
5. **Card Issuing & Authorization Simulator** — auth/capture/settlement
   flow modeled on ISO 8583-style card rails.
6. **Ledger-as-a-Service for Small Business Bookkeeping** — multi-tenant
   chart of accounts, invoicing, double-entry, closely modeled on
   Formance/Modern Treasury.
7. **Real-Time Payments Gateway** — FedNow/RTP-style instant transfer
   simulator with ISO 20022-shaped messages.
8. **KYC/Onboarding Identity Verification Platform** — document upload,
   risk scoring, CIP workflow.
9. **Investment Portfolio Rebalancing System** — order management,
   position tracking, rebalancing algorithm.
10. **Bill Pay / Recurring Payments Scheduler** — subscription billing,
    dunning/retry logic, scheduled transfers.
11. **Regulatory Reporting Platform** — SAR/CTR-style filing pipeline over
    a data warehouse fed by CDC from a ledger.
12. **Core Ledger + Payments + Fraud Slice (combined)** — #1, #3, and a
    thin slice of #7's message shape and #11's reporting, unified into one
    coherent "digital bank backend."

### 3.1 Scoring rubric

Each candidate scored 1-5 (5 = best) on six axes, defined before scoring:

- **Breadth** — how many distinct enterprise components (auth, async
  messaging, background workers, observability, compliance, etc.) does the
  concept *naturally* require, without padding?
- **Depth** — how domain-complex is the core problem itself (not the
  scaffolding around it)?
- **Solo-buildability** — realistic to build to a credible, tested state
  in 2-4 weeks at 10-20 hrs/week by a ~3-year engineer, per [Assumptions](#assumptions-i-can-override) below.
- **Legibility** — can a hiring manager understand what it is and why it's
  impressive in 5 minutes from the README alone?
- **Interview-story value** — how many distinct, meaty "tell me about a
  hard technical decision" stories does building this generate?
- **Clone risk** — inverted score (5 = low risk / distinctive, 1 = high
  risk / looks like every other tutorial). Weighted double, because this
  is an explicit failure mode called out in the brief.

### 3.2 Scoring matrix

| Candidate | Breadth | Depth | Solo-buildability | Legibility | Interview-story | Clone-risk (inv., ×2) | **Total /35** |
|---|---|---|---|---|---|---|---|
| 1. Core Ledger & Payments | 5 | 4 | 4 | 5 | 5 | 3×2=6 | **29** |
| 2. Loan Origination | 4 | 4 | 3 | 4 | 4 | 4×2=8 | **27** |
| 3. Fraud/AML standalone | 3 | 4 | 4 | 3 | 4 | 4×2=8 | **26** |
| 4. Open Banking Aggregation | 3 | 3 | 4 | 3 | 3 | 3×2=6 | **22** |
| 5. Card Issuing Simulator | 4 | 5 | 2 | 3 | 4 | 3×2=6 | **24** |
| 6. Ledger-as-a-Service (multi-tenant) | 4 | 4 | 3 | 3 | 4 | 2×2=4 | **22** |
| 7. Real-Time Payments Gateway | 3 | 4 | 3 | 4 | 3 | 3×2=6 | **23** |
| 8. KYC/Onboarding | 2 | 3 | 4 | 3 | 3 | 3×2=6 | **21** |
| 9. Investment Rebalancing | 3 | 4 | 2 | 3 | 3 | 4×2=8 | **23** |
| 10. Bill Pay Scheduler | 3 | 2 | 5 | 3 | 2 | 2×2=4 | **19** |
| 11. Regulatory Reporting | 3 | 3 | 3 | 2 | 3 | 4×2=8 | **22** |
| **12. Combined (Ledger + Payments + Fraud slice)** | **5** | **5** | **4** | **5** | **5** | **4×2=8** | **32** |

Rationale for the standouts:

- **#1 and #12** score highest on breadth and legibility because
  money-movement-with-a-real-ledger is the one banking sub-domain that
  *structurally requires* touching nearly every enterprise component:
  strict consistency, async processing, compliance, auditability, and
  security, all in service of a story anyone (recruiter to staff
  engineer) immediately understands: "it's a bank account."
- **#12 beats #1** specifically on clone-risk and interview-story value:
  adding the fraud/AML rules slice and the ISO 20022-shaped transfer
  message differentiates it from the many "double-entry ledger" tutorial
  repos that exist (we found one directly in research —
  [AryanDevCodes/double-ledger-banking-system](https://github.com/AryanDevCodes/double-ledger-banking-system))
  while staying inside the 2-4 week budget because the fraud slice reuses
  the same event stream the ledger already produces, rather than being a
  bolt-on subsystem.
- **#5 (card issuing)** has the best raw depth score but the worst
  solo-buildability — ISO 8583 auth/capture/settlement state machines are
  genuinely hard to get right and hard to demo convincingly without a real
  network, so the risk of a shallow, unconvincing implementation is high.
- **#2 (loan origination)** is a strong runner-up: excellent
  solo-buildability and low clone-risk, but its core hard problem
  (amortization + a servicing state machine) is fundamentally a
  **workflow/state-machine** problem, not a **concurrency/consistency**
  problem — it doesn't force the same depth of engagement with H1-H4
  (idempotency, concurrent ledger writes, sagas) that the brief's hard
  problems center on.

### 3.3 Recommendation

**Recommended: #12 — Core Ledger & Payments Platform with an integrated
fraud/AML slice** (working name **Ledgerline**, finalized in
[01-vision.md](01-vision.md)).

**Runner-up: #2 — Loan Origination & Servicing Platform.**
What would flip the recommendation: if Krishang's target roles skew
heavily toward loan-servicing or workflow-engine-shaped teams specifically
(some NJ-area regional banks and card issuers do), the state-machine depth
of loan servicing would out-legibility a ledger for that narrower audience.
Absent that signal, the ledger/payments concept is the safer, more broadly
recognized choice for "mid-level ASP.NET Core, open to fintech-adjacent
roles generally."

---

## 4. Assumptions I can override

Recorded per Hard Rule/Phase 0 instructions — override any of these at any
time.

- **Timeline:** 2-4 weeks to interview-ready.
- **Time budget:** 10-20 hrs/week once building starts (~20-80 hours total
  build time — the build plan in [08-build-plan.md](08-build-plan.md) is
  sized to fit inside this even at the low end).
- **Domain sub-area:** left fully open to Phase 1 scoring (see §3.3 above)
  rather than a stated preference.
- **Resume gaps:** none specifically requested to be proven out — component
  selection is driven by domain requirements and engineering merit (Hard
  Rule 5), not resume optics.
- **Deployment budget:** ~$25-30/month while actively interviewing,
  pausable after landing a job, slightly-over acceptable if justified. This
  *overrides* Hard Rule 8's zero-budget default and unlocks a real cloud
  deployment (see [04-architecture.md](04-architecture.md) cost story and
  [ADR-0003](adr/0003-deployment-and-hosting.md)) — but the "clone and run
  locally with one command" baseline in Hard Rule 8 still applies
  regardless of cloud spend.
- **Repo visibility:** public under Krishang's real name from day one —
  every commit is reviewer-visible immediately.
- **Target tech:** no specific job postings to weight toward — stack
  choices stay generically recognizable mainstream enterprise .NET rather
  than chasing niche tech from one listing.
- **UI:** no frontend is built (Hard Rule 2 leaves this to my judgment).
  Rationale: the target roles are backend ASP.NET Core; a frontend would
  consume a meaningful slice of a tight 2-4 week budget for a capability
  the audience (recruiters skimming a README, engineers reading code)
  doesn't need to evaluate backend depth. A well-documented Swagger/OpenAPI
  surface, a Postman/Bruno collection, and README walkthrough
  screenshots/GIFs substitute for a UI as the "way to see it work."
  Revisit if Krishang disagrees.
- **`.gitignore` change adding `CLAUDE.md`:** confirmed intentional by
  Krishang, left as-is. Since `CLAUDE.md` was already committed/tracked in
  the initial commit, this doesn't actually untrack it going forward
  (`.gitignore` only affects files that aren't already tracked) — noted
  here so it isn't a mystery later, no action taken.
- **Currency/locale:** USD only, US regulatory framing (BSA/AML/FinCEN,
  not PSD2/GDPR), consistent with NJ-based US job targeting. Multi-currency
  is explicitly out of scope (see [06-component-inventory.md](06-component-inventory.md), tier: Acknowledged).
- **"Bank" is simulated, not real:** no real money moves, no real payment
  rail connection, no real KYC vendor call. Every external dependency a
  real bank would have (payment rail, KYC provider, card network) is
  simulated behind an interface that documents what the real integration
  would look like. This is stated explicitly and repeatedly in the docs
  and README so it's never mistaken for a claim otherwise.
