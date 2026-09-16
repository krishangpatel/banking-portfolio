# ADR-0003: Deployment and hosting

## Status

Accepted

## Context

Hard Rule 8 defaults to a zero hosting budget with local-only execution as
the baseline. Phase 0 revised this: Krishang is willing to spend ~$25-30/
month (slightly over if justified) on hosting while actively interviewing,
pausable to near-$0 afterward. This unlocks a real, linkable cloud
deployment as an addition to — not a replacement for — the local baseline.

## Decision

**Two supported ways to run the system, both real:**

1. **Local (required, $0):** `docker compose up` runs the API, a SQL
   Server container, and the Azure Service Bus emulator container. This
   remains the baseline every reviewer can use with zero cloud account,
   satisfying Hard Rule 8 regardless of the cloud budget decision below.
2. **Cloud (optional, live demo link, target ≤$25-30/month):** Azure App
   Service (Basic B1) for the API, Azure SQL Database (General Purpose
   Serverless, 1 vCore, auto-pause after 60 minutes idle) for the
   database, Azure Service Bus (Basic tier) for messaging, Application
   Insights (pay-as-you-go, first 5GB/month free) for observability, Azure
   Key Vault for secrets. Full cost breakdown in
   [04-architecture.md §8](../04-architecture.md#8-cost-story).

Infrastructure is defined as code (Bicep) under `infra/`, and the pause
runbook (stop the App Service, pause the SQL database) is documented so
cost can drop to near-$0 between interview cycles without deleting the
environment.

## Alternatives considered

- **Local-only, no cloud deployment** (the Hard Rule 8 default). Rejected
  once Krishang explicitly opted into a budget — a live, clickable demo
  link is a real, low-cost improvement to the recruiter-skimming-the-
  README experience (audience #1 in [01-vision.md](../01-vision.md)), and
  the budget makes it affordable without compromising the "clone and run
  locally" requirement, which is kept regardless.
- **Azure Container Apps (consumption-based) instead of App Service.**
  Genuinely competitive on cost (scales to zero automatically, no explicit
  pause step needed) and was close. App Service was chosen instead because
  it's the more broadly recognized, "this is what an enterprise .NET team
  actually uses" choice for a portfolio audience of hiring managers and
  senior engineers evaluating mainstream competency — Hard Rule 2's
  "prefer mainstream choices a hiring manager recognizes over clever ones"
  applies directly here. Container Apps remains a reasonable substitution
  if the pause/resume runbook proves annoying in practice.
- **AWS or GCP instead of Azure.** Rejected — Azure is explicitly named in
  the fixed core stack (Hard Rule 2), and it's the cloud most consistent
  with a .NET-shop hiring narrative in New Jersey.
- **A free-tier-only deployment (Azure free tier / F1 App Service).**
  Rejected as the primary target — F1's resource limits (60 CPU
  minutes/day shared quota, no custom domain/SSL on some historical free
  tiers) make for a demo that could visibly degrade during an interview
  live-demo moment, which is a worse outcome than a ~$15-20/month tier
  that behaves reliably. Free tier is noted as a fallback if budget needs
  to drop to $0 without a full pause.

## Consequences

- A live URL exists to put in the README and on the resume, at a real but
  small monthly cost, consistent with NFR-17.
- Azure pricing pages did not render exact current numbers via automated
  fetch during this research pass (confirmed while researching
  [04-architecture.md](../04-architecture.md) — static pages show
  placeholders, requiring the interactive calculator). The cost estimate
  above is built from general, well-established Azure pricing knowledge
  and is **re-verified against the Azure Pricing Calculator immediately
  before first deployment** — that verification step is itself in
  [08-build-plan.md](../08-build-plan.md), not assumed away.
- A pause runbook is a required deliverable (Demonstrated tier,
  [06-component-inventory.md §2.7](../06-component-inventory.md)), not
  optional — the whole point of the budget decision was "pausable after
  landing a job," so the mechanism to actually do that has to exist and be
  documented, not just be theoretically possible.
