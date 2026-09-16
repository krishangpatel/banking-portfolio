# ADR-0005: Authentication and authorization

## Status

Accepted

## Context

FR-6 requires three distinct roles (Customer, SupportOps, ComplianceOfficer)
with real segregation of duties (a customer cannot approve their own
flagged transfer; only a ComplianceOfficer can resolve a case). Every
money-moving and PII-touching endpoint must be authenticated (NFR-9).

## Decision

**ASP.NET Core Identity** for user/credential management, issuing **JWT
bearer tokens** for API authentication, with **policy-based role
authorization** (`[Authorize(Policy = "ComplianceOnly")]`, etc.) enforced
per endpoint and covered by an explicit "should be denied" test per
endpoint per role that shouldn't have access.

## Alternatives considered

- **A third-party identity provider (Azure AD B2C / Entra External ID,
  Auth0).** The more "enterprise-realistic" choice for a system with real
  external customers, and genuinely tempting to include to demonstrate
  OAuth2/OIDC federation knowledge. Rejected for the Core build: it adds
  an external dependency and configuration surface (tenant setup, app
  registrations) that consumes build-plan time without adding to the
  system's *own* engineering story — the interesting part of this
  project's auth story is the authorization/segregation-of-duties model,
  not token-issuance plumbing. Documented in
  [06-component-inventory.md](../06-component-inventory.md) as a
  reasonable production evolution (Acknowledged) rather than built.
- **Cookie-based session auth.** A reasonable choice for a server-rendered
  app; rejected because this is an API-only system (see
  [ADR-0008](0008-no-ui-api-only.md)) consumed via Swagger/Postman/scripts,
  where bearer tokens are the more natural, more commonly expected fit.
- **API keys instead of per-user tokens.** Would work for service-to-
  service calls but doesn't model *who* (which customer, which employee)
  performed an action, which is required for the audit trail (FR-7.1) and
  for role-based segregation of duties — a hard requirement here, not a
  nice-to-have.

## Consequences

- Local development needs a realistic-but-fake user/credential seeding
  story (the seed script in
  [06-component-inventory.md §2.6](../06-component-inventory.md) creates
  one user per role) so a reviewer can exercise all three roles without
  registering accounts by hand.
- JWT signing key management is a real secret to protect — handled per
  NFR-11/ADR-0003 (User Secrets locally, Key Vault in the cloud), and its
  rotation is named explicitly as Acknowledged, not built
  ([06-component-inventory.md §2.3](../06-component-inventory.md)).
- The authorization model (three roles, policy-based) is intentionally
  simple relative to what a real bank would need (fine-grained,
  attribute-based access control across many more roles and data
  sensitivity levels) — sufficient to demonstrate the segregation-of-duties
  *pattern* correctly without building a policy engine a solo project
  doesn't need.
