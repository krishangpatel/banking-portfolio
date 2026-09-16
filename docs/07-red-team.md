# 07 — Red Team

Fifteen questions written from the perspective of a skeptical staff
engineer reviewing this repo as a take-home — correctness, failure modes,
scaling, security, over-engineering, authenticity, and "what happens at
2 a.m." Two questions (Q2, Q3) exposed real gaps during this pass; the
design was revised in place (see the linked diffs) rather than answered
around. This doc is Krishang's interview prep — the answers are written the
way he should be able to say them out loud, not as marketing copy.

---

**Q1. Your module boundaries are enforced only by code discipline inside
one SQL Server database. What actually stops someone — you, six months
from now, under deadline pressure — from just adding a cross-module join
next sprint?**

An architecture fitness test in CI (`tests/Ledgerline.Architecture.Tests/`,
[06-component-inventory.md §2.5](06-component-inventory.md)), not just
convention. It statically asserts no module's code references another
module's internal namespace or touches another module's tables directly —
cross-module interaction has to go through the mediator. A build that
violates the boundary fails CI, the same way a compile error would. This
is weaker than physically separate databases per service (which is the
real, harder guarantee), and that trade-off is explicit and intentional —
see [ADR-0002](adr/0002-modular-monolith-over-microservices.md) for why
the physical separation isn't worth its operational cost yet.

**Q2. What happens if two requests with the exact same idempotency key
arrive genuinely concurrently — not a retry after the first one finished,
but a real race?**

This was underspecified until this red-team pass. Fixed: the
`IdempotencyKeys.Key` column has a database-level unique constraint, not
just an application-level check-then-insert. Both concurrent requests
attempt to insert; the database allows exactly one; the loser catches the
uniqueness violation and reads back the winner's stored result instead of
proceeding to post a second time. An app-level check-then-insert without
the DB constraint would have exactly this race — the fix is at the layer
that can actually guarantee it. See the added row in
[04-architecture.md §6](04-architecture.md#6-failure-modes).

**Q3. Your saga for flagged transfers has no timeout in the original
design. What happens if a compliance officer never resolves a case — are
funds held forever?**

Also a real gap, also fixed in this pass. Every case now gets a `dueBy`
timestamp (24h from creation, FR-4.2) and a case-aging metric (NFR-16)
surfaces any `Open` case past its `dueBy`. This doesn't auto-resolve the
case — a held transfer should never silently clear or reverse itself
without a human decision, that would defeat the point of the review step —
but it makes an unattended case a visible, alertable condition instead of
a silent hang. In a real bank, this is exactly the kind of SLA a
compliance team would set contractually; a portfolio-scale version of the
same idea is the honest thing to build here.

**Q4. Why SQL Server instead of Postgres, given most of the fintech
engineering content you cited (Formance, several ledger-design blogs) is
Postgres-native?**

SQL Server is a fixed constraint of this project, not a choice made for
this specific concept — the stack (C#, ASP.NET Core, EF Core, SQL Server)
was set before domain/concept selection, deliberately, because the target
roles are enterprise .NET/SQL Server shops. Every ledger design principle
used here (append-only journal, zero-sum invariant, transactional outbox)
is database-agnostic and translates directly to Postgres if the stack
constraint ever changed — nothing here relies on a SQL-Server-specific
feature except where explicitly noted (e.g. `Always Encrypted`/TDE for
at-rest encryption, which Postgres has direct equivalents for).

**Q5. You claim ~50 req/s on a B1 instance and sub-300ms p95 — did you
actually measure that, or is it an assumption dressed up as a number?**

Measured locally, not assumed — [03-non-functional-requirements.md](03-non-functional-requirements.md)
draws the line explicitly: NFR-7 (p95 latency) is **Verified** via a
committed k6/bombardier script against the local Docker Compose stack,
results checked into `tests/load/`. NFR-8 (50 req/s on cloud B1) is marked
**Target**, not Verified, precisely because independently load-testing the
live cloud deployment at that volume would burn a meaningful slice of the
monthly Azure budget for a number with no interview value beyond "we
didn't check." That distinction — stating plainly which numbers are
measured and which are reasoned targets — is itself the answer to "did you
actually measure this."

**Q6. Reconciliation detects a discrepancy between the ledger and the
external statement. Walk me through the actual repair, not just the
detection.**

Detection (FR-5.1) opens a case; repair goes through the same mechanism
every other ledger correction uses (FR-2.3): journal entries are immutable,
so a confirmed discrepancy is corrected by posting a new, opposite
adjusting entry through an ops-only endpoint, referencing the
reconciliation case ID for traceability, never by editing history. There
isn't a separate, special-cased "reconciliation repair" code path — reusing
the one correction mechanism the whole system already has is the design,
not a gap.

**Q7. Calling your hold-review-resolve flow a "saga" is generous — it's a
two-state workflow in one service. Are you overselling the pattern?**

Fair pressure-test, and the honest answer is: the *label* isn't the point,
the *mechanics* are, and the mechanics genuinely match the pattern — a
sequence of local transactions (hold, then either release-to-destination
or release-to-source) with a compensating action if the flow doesn't
complete forward, coordinated without a single ACID transaction spanning
the whole (human-scale-duration) flow. It's a small saga because the
domain need is small — one review step, not a ten-step cross-service
workflow. Building an orchestration framework for a two-step saga would be
the over-engineering failure mode; using the concept without unnecessary
machinery around it is the actual judgment call being demonstrated (see
[05-design-patterns.md](05-design-patterns.md) for the deliberately-simpler
happy path, right next to this).

**Q8. Your outbox relay publishes, then marks a row sent. What if it
crashes after publishing but before marking it sent, then republishes the
same event on restart — are your consumers actually idempotent, or is that
just claimed?**

At-least-once delivery is the explicit, accepted contract (see
[04-architecture.md §6](04-architecture.md#6-failure-modes)) — the relay
does not try to guarantee exactly-once publish, because that guarantee is
expensive and unnecessary if consumers are idempotent instead. Consumers
record a processed-message ID (from the message's stable identifier, not a
timestamp) before acting, and check it first — the same idempotent-receiver
shape used for the transfer API itself (FR-3.1), applied at the messaging
boundary too. This is tested directly: an integration test publishes the
same message twice and asserts the side effect (e.g. a projection update)
happened exactly once.

**Q9. Your fraud rule thresholds ($10,000, $2,500) look hardcoded. If
compliance/legal needed a threshold changed same-day, would that require a
code deploy?**

No — rule *parameters* live in the `FraudRules` table (data, not code;
see [04-architecture.md §5](04-architecture.md#5-data-ownership)) and are
updatable via an audited admin endpoint, so a threshold change is a data
change, same-day, with an audit trail of who changed it and when. Adding a
genuinely *new* rule type (a new class of logic, not a parameter) does
require a code deploy, and that split — parameters are data, logic is code
— is the deliberate line, not an oversight.

**Q10. You picked a modular monolith specifically to avoid microservices
complexity — but you still built a message broker, an outbox, and a saga.
Isn't that the worst of both worlds: distributed-systems complexity without
the deploy/scale benefits of actual services?**

The broker/outbox/saga exist because of a *domain* requirement (H4 — a
review flow spanning human-scale duration, and async side effects like
notifications/reporting projections that shouldn't block the request path),
not because of a services split — a monolith with zero async processing
would be simpler, but it couldn't satisfy FR-3.4 (held/reviewed transfers)
or the reconciliation flow at all. The alternative being weighed was never
"no broker" — it was "fewer services communicating over the broker," which
is exactly the choice made: one deployable, using async messaging only
where the domain genuinely needs decoupling, not for every operation by
default (see the happy-path transfer in
[04-architecture.md §4.1](04-architecture.md#41-happy-path-transfer-not-flagged-by-any-rule),
which is one DB transaction, no broker involved).

**Q11. How is SSN-last-4 actually protected at rest — is it encrypted,
hashed, or just sitting in a column?**

Two layers, and they're different mechanisms doing different jobs: (1)
only the last 4 digits are stored at all — the full SSN never enters this
system's database, because real verification happens at a KYC vendor in
any real deployment (Acknowledged tier;
[06-component-inventory.md §2.2](06-component-inventory.md)), so there's
nothing more sensitive than a partial, low-value-on-its-own fragment to
protect in the first place; (2) Azure SQL's transparent data encryption
(TDE) is on by default for the whole database at rest, and the local
Docker Compose SQL Server container is configured to match. This is a
lighter protection story than a production bank's (which would use
column-level encryption / Always Encrypted for genuinely sensitive fields,
full PII tokenization, and a real vendor never touching our DB at all) —
that gap is named explicitly rather than implied to be solved.

**Q12. Did you actually build this, or is this AI-assisted scaffolding
you can't defend line by line?**

Direct answer: this repo was built with Claude Code as a design and
implementation collaborator, under a process designed specifically to
prevent that from becoming a liability — every significant decision has a
written ADR recording the alternatives *rejected* and why (not just the
choice made, which is the part a copy-pasted tutorial never has), every
build-plan phase requires Krishang's explicit approval and a Socratic
"sketch it yourself first" step before implementation
(see [08-build-plan.md](08-build-plan.md) and the project's own process
rules), and this red-team document exists precisely so the hardest
questions were rehearsed before an interviewer asks them, not improvised
live. The honest test isn't "was AI involved" — used this way, transparently
and disclosed, that's an increasingly normal engineering workflow — it's
"can Krishang explain and defend every decision," which is what the
approval gates and this doc are for.

**Q13. Twenty items in your inventory are "Acknowledged, not built." Isn't
that just a polite way of saying "I didn't do it" — how is this different
from vaporware?**

Vaporware claims something works that doesn't. Every Acknowledged item
explicitly states it is *not* built, names why (usually: requires a
commercial vendor relationship, requires scale/timeline this project
doesn't have, or is a different discipline like applied ML), and — where
there's a real design decision to sketch — names the interface or approach
a real implementation would use (e.g. `IIdentityVerificationProvider` for
KYC). The alternative — quietly building a shallow, unconvincing version of
all twenty just to check a box — would read far worse to a senior reviewer
than an honest "not built, here's why, here's what it would take." The
40/25/35 Core/Demonstrated/Acknowledged split
([06-component-inventory.md §3](06-component-inventory.md#3-tier-summary))
is itself evidence this was a considered scoping decision, not scope that
ran out from underestimating the work.

**Q14. What's the actual rollback story if a bad migration ships — you
said migrations are reviewed and CI-applied, but what if the migration
itself is the bug?**

Two mitigations, not one: migrations follow an expand-contract convention
(add nullable/additive changes first, backfill, deploy code that uses the
new shape, only remove the old shape in a later migration once the new
code has been running safely) so most migrations are non-destructive and a
code rollback doesn't require an undo migration on a live database. For
the genuinely destructive case, the backup/restore runbook
([06-component-inventory.md §2.4](06-component-inventory.md), Demonstrated
tier) documents and has been exercised once locally: Azure SQL
point-in-time restore before any migration classified as destructive, so
"restore to just before the bad migration ran" is a rehearsed action, not
a hope. This is the TSB lesson from
[00-exploration.md §1.6](00-exploration.md#16-what-happens-when-banking-software-gets-this-wrong)
applied directly — the failure mode there was exactly "no rehearsed
recovery path for a bad migration at real scale."

**Q15. It's 2 a.m., the outbox-lag metric has spiked to 10 minutes. Walk
me through your actual diagnosis, not the runbook summary.**

First, `/health` — confirms whether the DB and Service Bus are reachable
at all; if either is down, that's the root cause and the fix is upstream
of the relay. If both are healthy, check whether the backlog is *growing*
(relay isn't keeping up or has stopped) versus *steady* (throughput-bound
under unusually high transfer volume) via the outbox-pending-count metric
over time. If growing: check the relay's structured logs
(correlation-ID-tagged) for repeated exceptions on the same message ID —
that's a poison message; it should already be in the dead-letter table
after N retries (per [04-architecture.md §6](04-architecture.md#6-failure-modes)),
and if it isn't, that's the actual bug to fix, not the symptom. If the
relay process itself isn't running (crashed, no recent log lines), restart
it — it resumes from the outbox table's `Pending` rows, nothing is lost,
because the outbox row was written in the same transaction as the domain
change that needs to be published. The full step list lives in
`docs/runbooks/outbox-lag.md`; this is the reasoning behind those steps,
which is what actually needs to be defensible live.
