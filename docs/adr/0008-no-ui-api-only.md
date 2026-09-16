# ADR-0008: No frontend — API-only

## Status

Accepted

## Context

Hard Rule 2 leaves "whether there's a UI" to be decided and justified in
writing, not defaulted. The target roles (mid-level ASP.NET Core, backend-
focused) and the tight timeline (2-4 weeks, 10-20 hrs/week) are both direct
inputs to this decision.

## Decision

**No frontend is built.** The system is API-only. The "see it work"
surface for a reviewer is: an OpenAPI/Swagger UI at `/swagger`, an
exported Postman/Bruno collection committed to the repo, README
walkthroughs (including screenshots/short screen recordings of example
requests and responses), and the seed/demo data script that sets up
representative scenarios (a normal transfer, a flagged transfer working
through the case-review flow, a reconciliation mismatch) ready to explore
immediately.

## Alternatives considered

- **A minimal read-only admin/ops dashboard** (e.g., a small Blazor page
  showing accounts, recent transfers, and open cases). Genuinely
  considered — it would make the fraud-review and reconciliation flows
  more visually compelling for a recruiter skimming quickly. Rejected for
  the Core/Demonstrated build specifically because of opportunity cost:
  the 2-4 week budget is tight, the target roles are backend-only, and
  time spent on frontend polish is time not spent deepening the backend
  correctness/testing story that this audience actually evaluates. Noted
  as the first thing to add if the timeline turns out to have slack (see
  [08-build-plan.md](../08-build-plan.md)) — it would be additive, not a
  rework, since it would only consume the existing API.
- **A full customer-facing web app.** Rejected outright — building a
  believable customer UI (account dashboards, transfer flows, auth UI) is
  a large scope addition with no correctness or backend-depth payoff, and
  risks diluting the reviewer's attention away from the actual point of
  the project (per Hard Rule 5, this would be scope added "to show it
  off," which this ADR exists specifically to avoid).

## Consequences

- The README and API documentation quality become *more* important, not
  less — they're now the primary way anyone experiences the system
  without reading code, so they're treated as a Core deliverable with the
  same rigor as the code itself.
- If Krishang's read of interview feedback later suggests a UI would help
  (e.g., a hiring manager specifically wants to click through something),
  this decision is revisited — it's recorded as a considered trade-off
  under known constraints, not a permanent architectural commitment.
