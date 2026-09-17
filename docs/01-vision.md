# 01 — Vision

## What it is

**Ledgerline** is a simulated digital-bank backend: account management, a
real double-entry ledger, idempotent internal money transfers, and an
integrated fraud/AML transaction-monitoring slice, built as a modular
ASP.NET Core monolith with the async/eventing, compliance, and observability
machinery a real bank engineering team would recognize as load-bearing —
not decorative.

A customer can open an account, and money can move between accounts through
an API that treats every transfer as something that must survive retries,
concurrent access, partial failure, and regulatory scrutiny — because in a
real bank, it must. Every transfer produces an immutable, auditable journal
entry; risky transfers are held and routed through a rules-based review
case before they clear; a scheduled job reconciles the ledger against a
simulated external statement and raises a case the moment they diverge.

No real money moves. No real payment rail, KYC vendor, or card network is
connected — those are simulated behind interfaces that document what the
real integration would look like (see [06-component-inventory.md](06-component-inventory.md)
for what's Acknowledged vs. actually built).

## Where this sits in the real world

This isn't a novel problem — no bank is waiting for Ledgerline, and it
doesn't claim to be. It's a solo-scale, faithfully-engineered recreation of
a problem space that real companies solve commercially today: **Increase,
Modern Treasury, Formance, Unit, and Treasury Prime** are "ledger
infrastructure" / banking-as-a-service companies whose entire product is
exactly this — a correct double-entry ledger, idempotent money movement,
and compliance workflows, sold to other companies as an API so those
companies don't have to build core banking internals themselves. (Formance
in particular came up directly in the domain research —
[00-exploration.md §1.1](00-exploration.md#11-how-real-ledgercore-banking-systems-are-built).)
Inside an actual bank, this same problem space is owned by a core-banking
/ payments platform team — the group responsible for "the ledger is always
right" being true so every other team can build on top of it without
thinking about it.

That's the accurate pitch, and the one to actually give a recruiter or
interviewer: *not* "I solved a banking problem," but **"I built a
solo-scale version of the exact problem class that Increase and Modern
Treasury solve commercially, and that a bank's core-banking platform team
owns internally, to prove I understand the hard problems in that space
(idempotency, ledger consistency, compliance-as-a-workflow) and can
engineer correct solutions to them — not just talk about them."** It's
checkable, not just asserted: someone can go read Increase's or Modern
Treasury's own engineering docs on idempotency and ledger design and see
the same problem, solved the same way, in this repo.

## Who it's for

1. **A recruiter**, skimming the README for two minutes, sees a clear
   one-paragraph description, an architecture diagram, and a "run it in one
   command" promise that actually works.
2. **A hiring manager**, reading the docs for ten minutes, sees numbered
   functional/non-functional requirements, an honest tiered component
   inventory, and ADRs that show *why*, not just *what*.
3. **A senior engineer**, reading the code for an hour, finds double-entry
   invariants enforced at the data layer (not just checked in a service),
   idempotency handled correctly, a real (if small) saga, tests that
   exercise concurrency and failure — not just happy-path CRUD.
4. **An interviewer**, asking Krishang to defend it live, gets specific,
   confident answers because [07-red-team.md](07-red-team.md) already
   rehearsed the hardest questions and the ADRs record the alternatives
   that were rejected and why.

## Why it's impressive

- It picks the one banking sub-domain — money movement over a ledger —
  that structurally *requires* the full enterprise toolkit (strict
  consistency, async processing, compliance, auditability, security)
  rather than bolting those things onto a domain that doesn't need them.
- It confronts the domain's real hard problems (H1-H10 in
  [00-exploration.md](00-exploration.md)) with actual mechanisms —
  idempotency keys and a dedupe table, an append-only journal with a
  balance-never-drifts invariant, a transactional outbox, a small but real
  saga for held/reviewed transfers, a rules engine with named/versioned
  rules feeding a case queue — not prose claiming they're "handled."
- It is honest about scope: every component is tiered Core / Demonstrated
  / Acknowledged (see [06-component-inventory.md](06-component-inventory.md)),
  so nothing is presented as done that isn't, and nothing is silently
  missing without an explanation a reviewer can find before they have to
  ask.
- It's built and defensible by one mid-level engineer inside a 2-4 week,
  10-20 hrs/week budget — ambitious without tipping into "this clearly
  wasn't built by the person claiming it," which is the single fastest way
  a portfolio project backfires at Krishang's level.

## What it deliberately is not

- **Not a payments company.** No real payment rail (FedNow/RTP/ACH), no
  real card network, no real money. The transfer API's message shape
  borrows ISO 20022 field-naming conventions so the design reads as
  rail-aware, but this is explicitly *not* a claim of rail integration.
- **Not a microservices platform.** It's a modular monolith with clean
  internal module boundaries and one deployable, on purpose — see
  [ADR-0002](adr/0002-modular-monolith-over-microservices.md). Real
  microservices would add operational surface area (service discovery,
  distributed tracing across process boundaries, per-service CI/CD, N
  databases) that this timeline and this budget can't credibly support to
  a Core-tier standard, and a shallow microservices implementation reads
  worse to a senior reviewer than an honest, well-built monolith.
- **Not an ML fraud product.** The fraud/AML slice is a rules engine with
  named, documented, versioned rules — explainable and auditable, which is
  what a regulator actually wants to see. ML-based scoring is real-world
  best practice *alongside* rules (per the research in
  [00-exploration.md](00-exploration.md)) but is explicitly Acknowledged,
  not built — it's a different skill (applied ML, not backend
  engineering) and out of scope for this portfolio's target roles.
- **Not a full KYC/identity platform.** Account opening includes a
  simulated CIP (Customer Identification Program) step against a fake
  identity-verification interface — it demonstrates that the *workflow*
  exists and where a real vendor (Persona, Alloy, Socure) would plug in,
  without building document-verification/OCR/liveness-check technology
  that isn't backend-engineering work either.
- **Not a UI product.** API-only, with OpenAPI/Swagger docs, a Postman/
  Bruno collection, and README walkthroughs as the "see it work" surface.
  See the UI assumption in [00-exploration.md §4](00-exploration.md#4-assumptions-i-can-override).
- **Not multi-currency, multi-tenant, or globally distributed.** USD only,
  single organization, single-region deployment. Each of these is a real
  enterprise concern and each is Acknowledged in
  [06-component-inventory.md](06-component-inventory.md) with a stated
  design sketch, not built.
