# ADR-0001: Domain concept — Core Ledger & Payments Platform with fraud/AML slice

## Status

Accepted

## Context

The domain (banking/fintech) was fixed by the project brief; which specific
part of banking to build was explicitly left open, to be decided by
research and scoring rather than defaulting to the first obvious idea. Full
research and a 12-candidate scoring matrix are in
[00-exploration.md](../00-exploration.md); this ADR records the decision
itself and the reasoning at ADR granularity.

## Decision

Build **Ledgerline**: account management, a real double-entry ledger,
idempotent internal money transfers, and an integrated rules-based
fraud/AML monitoring slice with a compliance case-review workflow — a
"digital bank backend," not a payments company, a lending platform, or a
card issuer.

## Alternatives considered

- **Loan Origination & Servicing Platform** (runner-up, scored 27/35 vs.
  32/35). Strong solo-buildability and low clone-risk, but its core hard
  problem is a workflow/state-machine problem (amortization, servicing
  state transitions), not the concurrency/consistency problem
  (idempotency, concurrent ledger writes, sagas under partial failure)
  that this project's target hard problems center on. Would become the
  right choice if Krishang's target roles skewed specifically toward
  loan-servicing/workflow-engine teams.
- **Card Issuing & Authorization Simulator** (24/35). Highest raw domain-
  depth score, but the worst solo-buildability — ISO 8583-style
  auth/capture/settlement state machines are genuinely hard to build
  convincingly without a real network to validate against, raising the
  risk of a shallow, unconvincing implementation within the timeline.
- **Standalone Fraud/AML Transaction Monitoring System** (26/35). Good
  depth and low clone-risk, but scored lower on breadth — without a real
  ledger underneath it, the transactions being monitored are synthetic
  data rather than the system's own state, which is a materially less
  impressive and less legible story in 5 minutes.
- **Open Banking Aggregation, Ledger-as-a-Service, Real-Time Payments
  Gateway, KYC/Onboarding, Investment Rebalancing, Bill Pay Scheduler,
  Regulatory Reporting** — all scored lower (19-23/35), generally because
  they're either narrower in the enterprise components they naturally
  require, or (Bill Pay, KYC standalone) too shallow in domain complexity
  to generate enough interview-story value on their own.

Full scoring matrix: [00-exploration.md §3.2](../00-exploration.md#32-scoring-matrix).

## Consequences

- The system must confront real concurrency and consistency problems
  (idempotency, concurrent ledger writes, partial-failure handling) as its
  central engineering challenge — this shaped every architecture decision
  that follows (ADR-0002 through ADR-0006).
- The concept overlaps with a well-known category ("double-entry ledger
  demo") that exists in other portfolios and tutorials (one direct example
  surfaced during research). Differentiation comes from execution depth
  (the fraud/AML slice, the saga-based review flow, the honest tiered
  scope, the rehearsed red-team) rather than from picking an obscure
  concept — see [01-vision.md](../01-vision.md) for why a recognizable
  domain with standout execution beats an obscure domain for this
  audience.
- No real money, payment rail, or KYC vendor is connected — this is a
  simulation of a bank's internals, stated explicitly and repeatedly so it
  is never mistaken for a claim otherwise.
