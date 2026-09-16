# ADR-0004: Messaging technology — Azure Service Bus (with local emulator)

## Status

Accepted

## Context

The transactional outbox (H4) and the fraud-review saga
([05-design-patterns.md](../05-design-patterns.md)) need a real message
broker for async delivery, in both the local (Docker Compose, $0) and
cloud (Azure, ADR-0003) environments. The two environments should use the
same messaging technology and client code — running a different broker
locally than in production is itself a source of "works on my machine"
bugs and undermines the "clone and run locally, deploy the same thing"
story.

## Decision

**Azure Service Bus** everywhere — the real managed service in the cloud
deployment, and Microsoft's official **Service Bus emulator** (a Docker
container, no Azure account required) for local development. One
messaging technology, one client SDK, one set of queue/topic
configuration, in both environments.

## Alternatives considered

- **RabbitMQ**, self-hosted in both environments. A strong, very common
  enterprise choice and seriously considered. Rejected specifically
  because it would mean running a different broker in the cloud than
  RabbitMQ's own hosted offering would require either (a) a self-managed
  RabbitMQ VM/container in Azure — extra operational surface area (patching,
  scaling, HA config) that isn't the point of this portfolio's cloud
  deployment — or (b) a third-party managed RabbitMQ service, which adds
  vendor sprawl and cost outside the Azure-centric budget in ADR-0003. Azure
  Service Bus with a real local emulator avoids both problems by being the
  same technology in both places.
- **Apache Kafka.** Excellent for high-throughput event streaming, but
  it's the wrong tool for this system's actual messaging shape — the
  outbox relay publishes discrete, moderate-volume domain events (transfer
  posted, case opened), not a high-throughput stream needing partitioned
  consumer groups and long retention/replay. Choosing Kafka here would be
  resume-driven ("Kafka is impressive") rather than requirement-driven
  (Hard Rule 5) — a mainstream, right-sized choice is the stronger signal.
- **In-process only (MediatR notifications, no external broker at all).**
  Rejected — this would satisfy in-process module decoupling but not the
  actual requirement: the outbox pattern exists specifically to survive a
  process crash between a DB commit and the side effect being triggered
  (H4), which an in-process-only notification cannot do, because a crash
  before the notification fires loses it.

## Consequences

- One less "how is the demo different from prod" question to answer in an
  interview — the messaging code path is identical in both environments,
  which is itself a small, honest architectural strength worth naming.
- Ties the project to the Azure ecosystem for messaging specifically,
  which is consistent with (not an additional cost beyond) the Azure
  deployment decision in ADR-0003.
- The Service Bus emulator is a relatively recent Microsoft offering; its
  exact feature parity with the real service is verified early in the
  build plan (not assumed), since a local/cloud behavior mismatch
  discovered late would be expensive to work around.
