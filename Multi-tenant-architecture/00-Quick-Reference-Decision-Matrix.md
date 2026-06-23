# Quick Reference — Decision Matrix

## Which Database Isolation Should I Use?

```
Start here →
│
├── Are you early-stage or cost-constrained?
│   └── YES → Shared Schema + RLS. Start here. Migrate later.
│
├── Do you have enterprise customers requiring compliance (HIPAA, SOC2)?
│   └── YES → Database-per-tenant for those customers. Use tiered model.
│
├── Will you have > 10,000 tenants?
│   └── YES → Shared Schema or Branch-per-tenant (Neon/Turso). Schema-per-tenant breaks at scale.
│
└── Is data residency (EU data in EU) a hard requirement?
    └── YES → Database-per-tenant with region routing.
```

## Which Tenant Identification Should I Use?

```
B2B SaaS with branded experience?     → Subdomain (acme.yourapp.com)
API-first product?                     → Header (X-Tenant-ID) or JWT claim
Internal tooling / intranet?           → Path-based (/org/acme/...)
Mobile / native app?                   → JWT claim (no subdomain access)
```
