# Module 13 — Real-World Case Studies

## Salesforce: The Original Multi-Tenant Pioneer

Salesforce built their "Shared Everything" architecture in 2000 where all 150,000+ customers share the same Oracle tables. Their key innovations:

- **`OrgID` as the universal tenant key** — every table has it, every query filters by it
- **Custom metadata tables** — tenants can add custom fields without schema changes (stored as key-value pairs in a metadata table, not as actual columns)
- **Pod architecture** — they split their SIMT into "pods" (groups of \~10,000 tenants per pod) to limit blast radius

## Shopify: Scaling Schema-Per-Tenant

Shopify uses a database-sharding approach where each "shop" (tenant) is assigned to a shard. They pioneered the use of **Vitess** (MySQL sharding middleware) to manage tens of thousands of database shards.

Key insight: Shopify does not use schema-per-tenant on a single database. They use **pod-based sharding** where each shard cluster serves \~1,000 shops, providing physical-level isolation between shards while still sharing infrastructure within a shard.

## GitHub: Multi-Tenant at the Infrastructure Layer

GitHub (enterprise server) allows organizations to run their own GitHub instances but GitHub.com itself is a multi-tenant service where every organization is a tenant. They heavily use **read replicas per region**, **database-level sharding by user ID ranges**, and **Kafka for cross-shard event propagation**.

## Stripe: API-First Multi-Tenancy

Stripe's core design: every API call is authenticated by an API key that is scoped to an **Account** (their term for tenant). Their entire data model is partitioned by Account ID from the database level up.

Key pattern: **Idempotency keys are scoped per account** — the same idempotency key can be reused across different accounts without conflict.

## Notion: Workspace as Tenant

Notion treats each **Workspace** as a logical tenant within their shared infrastructure. They use Postgres with careful RLS policies and workspace-scoped permissions. Their challenge: collaborative workspaces where users belong to multiple workspaces require switching tenant context within a single user session.
