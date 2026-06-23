# Module 14 — Implementation Roadmap (NestJS + Postgres)

## Learning Objectives

- Have a concrete step-by-step path to build a production-ready multi-tenant API

## Phase 1: Foundation (Week 1–2)

- [ ] Set up NestJS monorepo (or modular monolith)
- [ ] Create `tenants` and `users` tables in a central registry DB
- [ ] Implement `TenantMiddleware` (subdomain-based resolution)
- [ ] Implement `AsyncLocalStorage` tenant context
- [ ] Set up Redis caching for tenant resolution

## Phase 2: Data Isolation (Week 3–4)

- [ ] Choose isolation model (shared schema + RLS recommended for start)
- [ ] Implement `TenantAwareRepository` base class that auto-filters by `tenant_id`
- [ ] Enable Postgres RLS policies on all tenant-scoped tables
- [ ] Write integration tests that assert cross-tenant data isolation (Two tenants, one of them must not see the other's data)

## Phase 3: Auth & RBAC (Week 5)

- [ ] JWT auth with `tenant_id` claim
- [ ] Tenant-scoped user registration and login
- [ ] RBAC: `roles` + `permissions` tables scoped per tenant
- [ ] `PermissionsGuard` NestJS decorator

## Phase 4: Onboarding Pipeline (Week 6)

- [ ] Async tenant provisioning via queue (BullMQ)
- [ ] Schema creation + migration on provisioning
- [ ] Tenant configuration service (feature flags, limits)

## Phase 5: Observability (Week 7)

- [ ] OpenTelemetry instrumentation with `tenant_id` on all spans
- [ ] Structured JSON logs with `tenantId` field
- [ ] Grafana + Prometheus dashboards (per-tenant panels)

## Phase 6: Billing (Week 8)

- [ ] Usage metering events (Kafka or BullMQ)
- [ ] Metering aggregator service
- [ ] Stripe integration for usage-based billing
- [ ] Quota guard middleware

## Phase 7: Production Hardening (Week 9–10)

- [ ] Rate limiting per tenant (throttle guard)
- [ ] Circuit breakers around tenant registry
- [ ] Migration runner with canary rollout support
- [ ] Load test with 100 simulated tenants (k6 or Artillery)
- [ ] Penetration test specifically targeting cross-tenant isolation
