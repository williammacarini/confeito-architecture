# Cost cascading with Redis Streams

How ingredient price changes propagate through preparations, recipes, and catalog products — and the consumer-group failure mode that only shows up in production.

---

## The domain model

Costing in a bakery is a **directed acyclic graph**:

```
ingredient ──► preparation ──► recipe ──► catalog product
   (R$/g)        (cost/unit)    (cost/portion)   (price)
```

- An **ingredient** has a price and a base unit (gram, ml, unit).
- A **preparation** (e.g. *brigadeiro base*) consumes ingredients with quantities.
- A **recipe** (e.g. *cake*) consumes preparations and/or ingredients.
- A **catalog product** is what the customer buys; it references a recipe and applies margin.

Because each level composes the one above it, a single leaf change invalidates everything upstream. A 12% increase in butter doesn't just change the butter line — it changes every preparation using butter, every recipe using those preparations, and the suggested price of every product built on them.

**The requirement:** when the owner updates a price, the whole subtree must reflect it — without the owner waiting, and without a stale price ever being displayed as current.

---

## Why not the obvious approaches

**Synchronous recalculation in the request.** The owner edits one ingredient and the HTTP request walks the graph. Problem: depth is data-dependent, not bounded. An ingredient used by 30 preparations, each in 15 recipes, is a 450-node traversal inside a request whose latency the user feels. It also couples the *write* to work that is logically *derived* from it — an ingredient update should succeed even if the recalculation never runs.

**Recalculate on read.** Compute cost lazily whenever a recipe is displayed. Problem: converts a write-time problem into a read-time one, and the expensive query now runs on the hot path. Also makes "what is my product's cost right now" a question with no answer outside a request context.

**Periodic full-table sweep.** A cron job recomputes everything every N minutes. Simple, and wrong: prices are stale for up to N minutes, and on a tiny bakery's dataset you're burning full-table work to service a change that touched three rows.

---

## The approach: events with a consumer group

The write path publishes. A consumer group recalculates.

```
        ┌───────────────────────────────────────────────┐
        │  PUT /ingredients/:id                         │
        │  → update row                                 │
        │  → PublishIngredientChanged(tenant, ingredient)│
        │  → 200 OK                       (fast path)   │
        └───────────────────┬───────────────────────────┘
                            │ XADD
                            ▼
                 ┌──────────────────────┐
                 │   Redis Stream       │
                 │  confeito:cost:recalc│
                 └──────────┬───────────┘
                            │ XREADGROUP (consumer group)
                            ▼
        ┌───────────────────────────────────────────────┐
        │  CostRecalculationService                     │
        │  → resolve affected preparations              │
        │  → resolve affected recipes                   │
        │  → recompute costs bottom-up                  │
        │  → update catalog product costs               │
        │  → XACK                                       │
        └───────────────────────────────────────────────┘
```

The message carries the minimum needed to resolve the subtree:

```go
type RecalcMessage struct {
	EntityType string // "ingredient" | "preparation" | "recipe"
	EntityID   string
	TenantID   string
}
```

`TenantID` is carried explicitly rather than looked up, so a consumer never has to query to know which tenant's data it is about to touch — and can't accidentally recompute across a tenant boundary.

**Why Streams over Pub/Sub:** Pub/Sub is fire-and-forget. If the API publishes while no consumer is running, the message is gone and prices stay silently wrong forever. A Stream persists, supports consumer groups, and tracks acknowledgement. For a correctness-critical propagation, at-least-once with acknowledgement is the floor.

---

## The failure mode that took real work

This is the part that looks correct in development and breaks in production.

With a consumer group, each message is delivered to one consumer and must be `XACK`ed. If a consumer dies after reading a message but before acking it, that message **stays in the Pending Entries List (PEL)** for that consumer. It is not redelivered to the group.

With a naive loop — `XREADGROUP` and process — the consequence is severe and quiet: **the prices covered by that message never update.** No error, no alert, no failed request. Just a costing sheet that is wrong.

In development this never appears, because the process doesn't die mid-message. It appears in production on a redeploy, an OOM kill, or a node eviction.

### The fix

Two parts, both in `NewCostRecalculationService`:

```go
// Consumer único por instância: com workers em várias réplicas, cada uma
// consome uma parte do grupo; mensagens de consumers mortos são retomadas
// via XAUTOCLAIM na inicialização (nunca ficam órfãs no PEL).
consumer := fmt.Sprintf("worker-%s", uuid.New().String()[:8])
```

**1. Unique consumer name per instance.** With a fixed name like `worker`, every replica shares one identity in the group. Redis then can't tell a message delivered to a *dead* replica from one delivered to a *live* one. A per-instance UUID makes each replica's pending set distinct, so orphaned entries are attributable to a specific dead worker.

**2. `XAUTOCLAIM` on startup.** On boot, each instance claims the pending messages of the dead consumers it is taking over from, rather than waiting for them to come back (they won't). Messages orphaned by a crash are recovered on the next deploy.

Together these turn the failure mode from *silent permanent staleness* into *delayed recalculation* — the graph settles a few seconds later than usual, and nobody sees a wrong price.

### Verifying it

The behavior is covered by tests using `redismock`, which assert publish payloads and consumer-group semantics. The boundary conditions that matter:

- Recalculating the same subtree twice produces the same result (idempotency).
- An unacked message is reclaimed rather than lost.
- A tenant's recalculation never touches another tenant's rows.

---

## Trade-offs, stated plainly

**At-least-once means idempotent recalculation is mandatory.** The consumer may process the same message more than once. Recalculation therefore *computes* the correct cost from current data rather than *adjusting* a stored value by a delta. Recomputing twice is wasted work; recomputing zero times is a wrong price. The design chooses waste over wrongness — the correct trade for a numeric field a business prices from.

**Eventual consistency is visible to the user.** Between publish and processing, a recipe shows the old cost. This window is milliseconds in practice, but it is real and it is a design property, not an accident. Because the recalculation is derived data, this is acceptable; if a price were used to *charge* a customer at that instant, it would not be.

**Redis becomes a durability dependency.** With Pub/Sub, Redis going down costs you deliveries. With Streams and AOF persistence, a Redis restart doesn't lose pending work — but you now depend on Redis durability, not just its availability. That's a real increase in the operational surface, accepted deliberately.

**What I'd change next:** batch the messages. A supplier invoice updating 50 ingredient prices publishes 50 messages that overlap heavily on the same subtree. A debounce window (or accepting a batch entity list) would collapse them into one pass. The current design is correct but does redundant traversal under bulk updates.

---

## Applying this pattern elsewhere

The shape generalizes to any **derived value with a dependency graph** that needs to stay consistent without blocking writes:

- Pricing with layered discounts or tax rules
- Inventory availability across bundles and kits
- Permission sets derived from role hierarchies
- Search indexes derived from structured records

The pattern's requirements are consistent: a DAG you can traverse, a computation that is **idempotent**, and tolerance for a small consistency window. If the computed value is used to move money at that instant, this is the wrong tool — compute synchronously or hold a lock.

---

**William Macarini** — [williammacarini.com.br](https://www.williammacarini.com.br) · [github.com/williammacarini](https://github.com/williammacarini)
