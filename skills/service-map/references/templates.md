# Templates

Both docs are `MARKDOWN`, scope `PROJECT`. Keep the section headings exactly as written: other
runs find their way around a doc by them, and "Open questions" is deliberately not called
"Open gaps" (see "Names and search" in SKILL.md).

## Service doc

- `name`: `<service-name> (service map)`
- `projectKey`: the repo folder name
- `summary` (max 1024 chars, never the word "gap"): what the service does, its set, and its
  main neighbours. Search and recall show only this, so an agent decides from it whether to
  open the doc.
  Example: `order-service: owns the order lifecycle for the web shop (create, pay, cancel,
  ship). Set: company-platform. Calls billing-service and inventory-service; produces topic
  order-events; consumes payment.completed.v1.`

### Sections

```markdown
# <service-name>

## Purpose
What the service does on its own, in 2-4 sentences. What it owns (data, decisions), not how.

## Identification
- Set: <set-key>
- Repo / projectKey: <repo folder>
- Service name: <name callers use> (<where it comes from>); aliases: <others or none>
- groupId: <...> · artifactId: <...> · base package: <...>

## Inbound
### REST
One line per resource, not per endpoint. List an endpoint on its own line only when its
callers differ from the rest of the resource or it has an open question.
- <METHODS> <base path>[, <sub-paths>]: <purpose>. Callers: <service (how)>, ... | unknown.
### Messaging
- <Kafka topic | Rabbit queue | ...> <name> (group <consumer group>): <what the service does with it>. Producer: <service> | unknown.
### gRPC
- <package.Service>/<Method or *>: <purpose>. Callers: ... | unknown.
### Scheduled
- <job>: <what and when>. (Omit the subsection if there are none; same for the others.)

## Outbound
- <service> via <Feign client X | WebClient | RestTemplate | RestClient | gRPC stub>: <METHOD path>, ... Target does: <what> | unknown.
- Produces <topic> (<event types>): Consumers: <service: what it does>, ... | unknown.
- Database: <engine, schema/db name, migrations tool>. Cache: <...>.
- External: <system: what for>. (No questions for external systems; just name them.)

## Cross-cutting
Only what a developer must not break: tenant isolation, auth, transaction boundaries,
idempotency, outbox, retries, caching, ordering, locking. One line each, naming the class or
config that enforces it.

## Code conventions
Only where this service differs from the index doc's shared conventions, or conventions no
other mapped service shares yet. "Same as set." when nothing differs.

## Tests
Same rule: only what differs from the index doc. Frameworks, fixtures, test infra
(Testcontainers, WireMock, embedded Kafka), base classes.

## Open questions
- GAP: <question with exact names>?
- none  (when there are none)

## Last analyzed
<YYYY-MM-DD>, commit <short sha>[, with uncommitted changes]
```

### Example (abridged)

```markdown
# order-service

## Purpose
Owns orders for the web shop: creation, payment state, cancellation and hand-off to
shipping. It is the source of truth for order status; other services learn about changes
from topic order-events.

## Identification
- Set: company-platform
- Repo / projectKey: order-service
- Service name: order-service (spring.application.name); aliases: orders (k8s Service)
- groupId: com.company.orders · artifactId: order-service · base package: com.company.orders

## Inbound
### REST
- GET, POST /api/v1/orders, GET /api/v1/orders/{}: create and read orders. Callers: web-bff (Feign client OrderClient).
- POST /api/v1/orders/{}/cancel: cancel an unpaid order. Callers: unknown.
### Messaging
- Kafka topic payment.completed.v1 (group order-service): marks the order paid. Producer: unknown.

## Outbound
- billing-service via Feign client BillingClient: POST /api/v1/invoices. Target does: creates the invoice and charges the card.
- inventory-service via WebClient: POST /api/v1/reservations. Target does: unknown.
- Produces order-events (OrderCreated, OrderCancelled): Consumers: billing-service: opens a draft invoice; notification-service: unknown.
- Database: PostgreSQL, schema orders, Liquibase.

## Cross-cutting
- Tenant isolation: every query filters by tenant_id from TenantContext (TenantFilter); repositories must not bypass it.
- Order status changes and the order-events record are written in one transaction (outbox table, OutboxPublisher).

## Code conventions
Same as set, except: no MapStruct; mapping is hand-written in *Assembler classes.

## Tests
Same as set.

## Open questions
- GAP: who calls POST /api/v1/orders/{}/cancel?
- GAP: who produces topic payment.completed.v1?
- GAP: what does inventory-service do with POST /api/v1/reservations?
- GAP: what does notification-service do with topic order-events?

## Last analyzed
2026-10-15, commit 3f2a91c
```

## Index doc

- `name`: `<set-key> (service map index)`
- `projectKey`: the set key
- `summary`: `Service map index of the <set-key> set: one line per mapped service and the
  conventions two or more of them share.` Add a few words on what the system is once known.
  Never the word "gap".

```markdown
# <set-key>

## Services
- <service-name>: <one-line purpose>. Repo: <projectKey>.

## Set evidence
How a run recognises a member: parent POM/BOM, groupId and package prefix, shared libraries,
deployment files.

## Shared conventions
Each line names the services that follow it; a convention needs at least two.
- <convention> (<service>, <service>)

## Shared test conventions
- <convention> (<service>, <service>)

## Last updated
<YYYY-MM-DD> by the service-map run on <service-name>
```

When a convention drops to one follower (a service changed), move it back into that
service's doc.
