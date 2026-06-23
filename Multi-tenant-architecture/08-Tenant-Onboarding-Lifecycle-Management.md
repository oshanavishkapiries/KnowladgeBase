# Module 8 — Tenant Onboarding & Lifecycle Management

## Learning Objectives

- Design automated tenant provisioning workflows
- Handle tenant suspension, deletion, and data export
- Understand tenant configuration management

## Tenant Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Pending: Signup
    Pending --> Provisioning: Email verified
    Provisioning --> Active: Infra + DB ready
    Active --> Suspended: Payment failed\nor policy violation
    Suspended --> Active: Issue resolved
    Suspended --> Deleting: Churn / account closed
    Active --> Deleting: Churn / GDPR request
    Deleting --> [*]: All data purged\nAudit log retained
```

## Automated Provisioning Flow

A new tenant signup should trigger a provisioning pipeline:

```mermaid
sequenceDiagram
    participant U as User (Signup Form)
    participant API as Platform API
    participant PQ as Provisioning Queue
    participant PW as Provisioning Worker
    participant DB as Tenant Registry DB
    participant R as Redis Cache
    participant TDB as Tenant DB / Schema

    U->>API: POST /signup { name, email, org }
    API->>DB: INSERT tenant { slug, tier='free', status='pending' }
    API->>PQ: Enqueue ProvisionTenant { tenantId }
    API-->>U: 202 Accepted (provisioning async)

    PW->>PQ: Dequeue
    PW->>TDB: CREATE SCHEMA tenant_{id}
    PW->>TDB: Run migrations on new schema
    PW->>DB: UPDATE tenant SET status='active'
    PW->>R: Invalidate cache for tenant slug
    PW->>U: Send "Your account is ready" email
```

**Why async provisioning?** Schema creation and migration can take seconds or even minutes. Returning a 202 immediately, then notifying via email or webhook, gives much better UX.

## Tenant Configuration Service

Each tenant typically has configuration that diverges from defaults:

```typescript
interface TenantConfiguration {
  tenantId: string;
  
  // Branding
  logoUrl: string;
  primaryColor: string;
  customDomain?: string;       // app.acme.com → your platform

  // Feature flags (per tenant)
  features: {
    advancedReports: boolean;
    apiAccess: boolean;
    ssoEnabled: boolean;
  };

  // Limits
  maxUsers: number;
  maxApiRequestsPerMinute: number;
  storageLimitGb: number;
}
```

Store this in the tenant registry (fast Redis cache) — it is read on every single request to determine what a tenant can do.
