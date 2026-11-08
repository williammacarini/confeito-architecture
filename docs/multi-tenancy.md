# Multi-tenancy: shared tables, one choke point

How tenant isolation works in Confeito Gestão — the model, why it was chosen over the alternatives, and the specific bug class it is designed to make impossible.

---

## The requirement

Every customer of Confeito is an independent bakery. Each has its own ingredients, recipes, customers, orders, and pricing. No bakery may ever see another's data — not through the UI, not through the API, not through a leaked ID.

The system is a **single deployment serving all tenants**, not one instance per customer.

---

## The three standard models

| Model | Isolation | Migration cost | Blast radius of a bug |
|---|---|---|---|
| **Database per tenant** | Strongest | × N tenants | One tenant |
| **Schema per tenant** | Strong | × N tenants | One tenant |
| **Shared tables + `tenant_id`** | Application-enforced | Once | Potentially all tenants |

The first two are physically safer. The third is operationally simpler.

---

## The decision: shared tables + `tenant_id`

Chosen because of who the customer is.

A bakery owner runs one or two stores, has a few thousand rows of data, and needs the software to be cheap and always available. Schema-per-tenant means every migration runs N times — with N in the thousands, a routine column addition becomes a batch job with partial-failure states to reconcile. Database-per-tenant adds connection-pool pressure and per-tenant backup management. Both move real operational cost onto the team in exchange for isolating a blast radius that the application layer can also handle.

For a product with a **small per-tenant data footprint and a large tenant count**, shared tables with rigorous enforcement is the pragmatic answer. The model is not "less safe" — it is safe *by a different mechanism*, and that mechanism has to be built deliberately.

---

## Making the mechanism real

The risk with shared tables is specific and well-known: **one query missing its `WHERE tenant_id = ?` is a cross-tenant data leak**, and that mistake is invisible in code review — it looks exactly like 40 correct queries around it.

So the design doesn't rely on remembering. It relies on a **single choke point**.

### 1. Tenant identity enters once, from the token

The JWT carries `tenant_id`. Auth middleware validates the token and places the tenant in the request context:

```go
func GetTenantID(c *fiber.Ctx) string {
	tenantID, _ := c.Locals("tenantID").(string)
	return tenantID
}
```

The tenant is **never** read from a query parameter, request body, or path. If it came from user input, a user could supply someone else's tenant ID — the entire isolation model would rest on trusting the client.

### 2. One validation function, used everywhere

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

Two distinct failures are distinguished, and the distinction matters operationally:

- **No tenant in context** → `401`. The request never authenticated; something upstream is broken.
- **Tenant mismatch** → `403`. Authentication succeeded but the principal is reaching outside its scope — the signal you want on a dashboard, because it's either a bug or an attack.

Collapsing both into a generic `403` would hide the difference between "misconfigured" and "someone is probing".

### 3. Handlers pass the resource's owner, and repositories filter

Handlers resolve the resource, check ownership, and only then act. Repositories receive `tenant_id` and apply it to every query.

---

## Defense in depth

Middleware alone is a single layer, and single layers fail. Three more:

**Tenant-scoped uniqueness.** Constraints are composite — a tenant can reuse a product name that another tenant already has, but not twice within itself. This also means one tenant's data can never collide with another's.

**Cache keys are tenant-prefixed.** Anything cached in Redis carries the tenant in the key. A shared key is a cross-tenant leak through the cache — a failure path that never touches the database and is therefore invisible to query-focused review.

**Tests assert isolation explicitly.** Cross-tenant access is a tested behavior, not an assumption. Tests confirm that a valid token for tenant A receives `403` — not an empty result, not a silent fallback — when requesting tenant B's resource.

---

## Where this model is weak — honestly

**The database user has no isolation.** Application code connects with credentials that can read every tenant's rows. A SQL injection, a raw query, or a compromised connection reads everything. Physical isolation would bound this; shared tables do not. Mitigations are the usual ones — parameterized queries via GORM, no string-built SQL, restricted credentials — but the *structural* guarantee isn't there.

**The enforcement is a convention, not a constraint.** `ValidateTenantAccess` protects the paths that call it. Nothing in the database prevents a new handler from forgetting. The strongest available fix is structural: bind `tenant_id` to the **repository at construction** rather than passing it per call, so a repository instance physically cannot query outside its tenant.

```go
// Target state — makes the forgotten-clause bug impossible, not just unlikely
repo := repository.NewRecipeRepository(db, tenantID)
```

This is the first change I'd make to this design. Today's correctness depends on discipline; that version depends on the type system.

**Migrations are global.** A bad migration affects every tenant simultaneously. With schema-per-tenant, it would affect one and you'd learn before the rest.

**Noisy neighbors share a database.** One tenant running an expensive query consumes shared resources. Physical isolation bounds this; shared tables require query-level discipline and monitoring.

---

## The documented trigger for revisiting

This decision has a **defined exit condition**, not an open-ended "we'll see":

- **Tenant count in the thousands with a large data footprint** — migration cost per tenant stops being negligible, and the operational argument for shared tables weakens.
- **Enterprise customers requiring contractual isolation** — a compliance requirement overrides the operational preference, and those tenants get a dedicated schema.
- **A cross-tenant leak incident** — a single confirmed leak would invalidate the model's core assumption that application-level enforcement holds. Reassess immediately.

Naming the trigger in advance is what keeps this an engineering decision rather than a default that nobody revisits.

---

## Summary

Shared tables with `tenant_id` is a deliberate trade: **operational simplicity and cheap onboarding, in exchange for isolation enforced by the application rather than the database.**

It is the right call for many small tenants. It is only defensible if enforcement is centralized, defensive layers back it up, the failure modes are understood, and the conditions that would invalidate it are written down.

---

**William Macarini** — [williammacarini.com.br](https://www.williammacarini.com.br) · [github.com/williammacarini](https://github.com/williammacarini)
