# Confeito Gestão — Architecture

Architecture notes from a **multi-tenant SaaS for bakery management**, built end-to-end: product, backend, frontend, and deployment.

> **About this repository** — this is an architecture and design-decision write-up of a commercial product. The product source code is private. Everything here describes real decisions, real trade-offs, and real numbers from the production codebase — no illustrative pseudo-code.

[![Go](https://img.shields.io/badge/Go-1.25-00ADD8?logo=go&logoColor=white)](#)
[![Fiber](https://img.shields.io/badge/Fiber-v2.52-00ADD8)](#)
[![Next.js](https://img.shields.io/badge/Next.js-15-000000?logo=nextdotjs)](#)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)](#)
[![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white)](#)

---

## The product problem

Bakeries price by **recipe**, not by product. A cake's cost depends on its preparations, which depend on ingredients, whose prices change weekly. When butter goes up, every recipe that uses it is suddenly mispriced — silently.

Most generic ERPs handle this badly: they treat costing as a spreadsheet the owner maintains by hand. Confeito treats it as **a graph with event-driven propagation**.

Three tenant-facing capabilities drive the architecture:

1. **Technical costing sheets** — ingredient → preparation → recipe → catalog product, with cost cascading through every level.
2. **Online store** — a public catalog per tenant, orders landing directly in WhatsApp.
3. **Order workflow** — quote → production → delivered, for made-to-order events.

---

## System shape

```
┌──────────────────────────┐        HTTP / REST        ┌──────────────────────────┐
│      apps/web            │  ◄────────────────────►   │       apps/api           │
│  Next.js 15 App Router   │                           │   Go 1.25 + Fiber v2     │
│  Tailwind + shadcn/ui    │                           │   GORM + pgx             │
│  TypeScript              │                           │   JWT (access + refresh) │
└──────────────────────────┘                           └────────────┬─────────────┘
                                                                    │
                                                     ┌──────────────┴──────────────┐
                                                     │                             │
                                              ┌──────┴──────┐              ┌───────┴───────┐
                                              │ PostgreSQL  │              │    Redis 7    │
                                              │     16      │              │  tokens +     │
                                              │  GORM/pgx   │              │  Streams      │
                                              └─────────────┘              └───────────────┘
```

**Layering inside the API** — a strict one-way dependency chain:

```
handlers  →  services  →  repository  →  models
   │            │             │            │
 HTTP +      business      data          GORM
 DTOs        rules         access        entities
```

`handlers` never touch the database. `services` never touch `fiber.Ctx`. This is what makes the service layer unit-testable with an in-memory SQLite driver while production runs PostgreSQL.

---

## Two decisions worth writing down

### 1. Cost cascading via Redis Streams

**The problem.** An ingredient price change has to propagate:

```
ingredient ──► preparation ──► recipe ──► catalog product
```

Doing this synchronously inside the HTTP request means the user waits on a graph traversal whose depth they can't see. Doing it in a background table-scan job means stale prices for minutes.

**The approach.** The ingredient update publishes to a Redis Stream. A consumer group recalculates the affected subtree asynchronously. The write path stays fast; the graph settles in the background.

**The part that actually took work.** Consumer groups across multiple replicas have a failure mode that only appears under real deployment: when a worker dies mid-message, that message sits in the **Pending Entries List (PEL)** and — with a naive `XREADGROUP` loop — is never reclaimed. Prices silently stop updating for whatever subtree that message covered.

The fix was a per-instance consumer name plus `XAUTOCLAIM` on startup:

```go
// Consumer único por instância: com workers em várias réplicas, cada uma
// consome uma parte do grupo; mensagens de consumers mortos são retomadas
// via XAUTOCLAIM na inicialização (nunca ficam órfãs no PEL).
consumer := fmt.Sprintf("worker-%s", uuid.New().String()[:8])
```

Each replica consumes a slice of the group, and messages orphaned by a dead consumer are reclaimed on boot rather than lost.

**Trade-off accepted:** at-least-once delivery means recalculation must be **idempotent**. Recomputing a subtree twice is wasteful but correct; recomputing zero times is a wrong price shown to a customer.

→ Read the full write-up: [`docs/cost-cascading.md`](docs/cost-cascading.md)

### 2. Multi-tenant isolation in the middleware layer

Every tenant's data lives in shared tables, scoped by `tenant_id`. The tenant ID comes from the JWT and is placed in the request context once; repositories filter on it.

```go
func ValidateTenantAccess(c *fiber.Ctx, resourceTenantID string) error {
	currentTenantID := GetTenantID(c)
	if currentTenantID == "" {
		return fiber.NewError(fiber.StatusUnauthorized, "Tenant context not found")
	}
	if resourceTenantID != "" && currentTenantID != resourceTenantID {
		return fiber.NewError(fiber.StatusForbidden, "Access denied to this tenant resource")
	}
	return nil
}
```

**Why this shape.** A single choke point is auditable. A `WHERE tenant_id = ?` scattered across 40 query builders is not — one forgotten clause is a cross-tenant data leak, and that class of bug is invisible in code review.

**The honest trade-off.** Shared tables mean no physical isolation: a bad query can still leak across tenants. Schema-per-tenant would prevent that but multiplies migration cost by tenant count. For a product targeting independent bakers, the operational simplicity won. Above a few thousand tenants, this decision would deserve revisiting — and that's a documented trigger, not a hand-wave.

→ Read the full write-up: [`docs/multi-tenancy.md`](docs/multi-tenancy.md)

---

## Testing strategy

**43 test files** across the API, testing behavior rather than implementation:

| Layer | Files | What's covered |
|---|---|---|
| `services` | 18 | Business rules, costing math, billing flows, auth roles, quote logic |
| `handlers` | 17 | HTTP contracts, status codes, conflict handling, privacy endpoints |
| `middleware` | 8 | Auth, cookies, CORS, rate limiting, tenant isolation, trial expiry |

Two conventions that paid off:

- **In-memory SQLite per test** — services run against a real database engine, not mocks, so GORM query semantics are exercised. Tests stay fast because the DB is ephemeral.
- **Tests assert on invariants, not snapshots.** For example, privilege-escalation guards are tested from both directions:

```go
// Registro nunca eleva privilégio: qualquer email (inclusive o antigo
// admin) entra como role padrão e não herda permissões anteriores.
```

Naming the *rule* in the test comment means a future reader knows what breaking it would mean — the test documents intent, not just expectations.

---

## Delivery / infrastructure

- **Docker Compose** for local: `postgres:16-alpine` + `redis:7-alpine` with healthchecks gating API startup.
- **Monorepo** with npm workspaces: `apps/api`, `apps/web`, `packages/shared-types` — types generated once, consumed by both sides, so an API contract change breaks the frontend build rather than production.
- **Local dev via `Makefile`** — one target brings up infrastructure and runs both apps.
- **Seed data** — demo tenant with a populated catalog, so the storefront is explorable without manual data entry.

---

## Scale of the codebase

| | |
|---|---|
| Go source files | ~150 |
| Total LOC (API + web + shared types) | ~62,000 |
| Test files | 43 |
| Backend dependencies | Fiber, GORM, pgx, go-redis, golang-jwt |

---

## What I'd do differently

An architecture document that only lists successes is marketing, not engineering.

- **`tenant_id` as a repository constructor argument.** It's currently passed per call. Binding it to the repository instance at construction makes the forgotten-clause bug structurally impossible. That's the change I'd make first.
- **Cost recalculation should be batched.** Today each ingredient change enqueues its own message. Fifty price updates from one supplier invoice produce fifty recalculation events over the same subtree. A debounce window would collapse them.
- **Shared types should be generated from the Go models**, not hand-maintained alongside them. The current approach works but drifts at the edges.

---

## About

**William Macarini** — Senior Software Engineer, Go & .NET, distributed systems and fiscal platforms at scale.

- 🌐 [williammacarini.com.br](https://www.williammacarini.com.br)
- 💼 [LinkedIn](https://www.linkedin.com/in/william-macarini-288aa5189)
- 🐙 [github.com/williammacarini](https://github.com/williammacarini)

Happy to go deeper on any of this in a conversation — including the parts that didn't work.
