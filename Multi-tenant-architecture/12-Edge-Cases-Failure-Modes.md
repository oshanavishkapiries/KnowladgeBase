# Module 12 — Edge Cases & Failure Modes

## Learning Objectives

- Identify the most dangerous failure modes in multi-tenant systems
- Implement mitigations for each

## Critical Failure Modes

### 1. Cross-Tenant Data Leak

The most severe failure. Causes: missing `tenant_id` filter in a query, a buggy JOIN, or a middleware that fails open.

**Mitigations:**

- Postgres Row-Level Security as a hard backstop (even if app code is wrong)
- Integration tests that explicitly assert tenant isolation with two tenants
- Automated query auditing that fails CI if a query on a tenant-scoped table lacks a `tenant_id` filter

### 2. Noisy Neighbor (Resource Starvation)

One tenant generates a spike (bulk import, runaway job) that degrades all other tenants.

**Mitigations:**

- Per-tenant rate limiting at the API Gateway level
- Per-tenant database connection pool limits
- Background job queues scoped per tenant with fair scheduling
- Kubernetes ResourceQuotas (see Module 7)

### 3. Tenant Registry Outage

If the service that resolves tenant slugs → tenant config goes down, no request can be served.

**Mitigations:**

- Redis cache with a long TTL (5–30 min) — most requests are served from cache
- Circuit breaker: if registry is down, serve from stale cache rather than fail
- The registry should be a read-heavy, highly available service (not the same DB as tenant data)

### 4. Migration Failure on Subset of Tenants

A migration runs successfully on 9,990 tenants and fails on 10. Those 10 tenants are now on an older schema version.

**Mitigations:**

- Track migration status per tenant in a `tenant_migrations` table
- Deploy code that can handle both old and new schemas during the migration window (forward-compatible code)
- Alert immediately when tenant migration fails; retry with exponential backoff

### 5. Stale Tenant Cache Serving Deleted/Suspended Tenant

A tenant is suspended for non-payment. The cache still holds their active config. They keep making requests for 5 more minutes until TTL expires.

**Mitigations:**

- Publish a `tenant.suspended` event to a message bus; cache service subscribes and immediately invalidates
- Shorter TTL for tenant status (30s) vs. tenant config (5 min)

### 6. Tenant Impersonation / JWT Confusion

A malicious actor modifies their JWT to claim a different `tenant_id`.

**Mitigations:**

- Always verify JWT signature (never trust `tenant_id` from unsigned request body)
- Double-check that JWT `tenant_id` matches the tenant resolved from the subdomain/route — they must agree
