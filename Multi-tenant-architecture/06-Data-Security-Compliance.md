# Module 6 — Data Security & Compliance

## Learning Objectives

- Apply encryption at rest and in transit per tenant
- Understand GDPR "right to erasure" in multi-tenant systems
- Implement audit logging

## Encryption Strategies

**At Rest:**

- Use envelope encryption: each tenant's sensitive data is encrypted with a **tenant-specific Data Encryption Key (DEK)**, which is itself encrypted with a Key Encryption Key (KEK) stored in AWS KMS / Azure Key Vault / GCP Cloud KMS.
- Rotating a tenant's keys does not require re-encrypting all other tenants' data.

```
Tenant Data → encrypted with DEK (per-tenant)
DEK → encrypted with KEK (stored in KMS)
KEK → managed by cloud KMS (hardware-backed)
```

**In Transit:**

- TLS 1.3 on all connections (browser ↔ API, service ↔ service, service ↔ DB)
- mTLS between internal microservices (Istio / Linkerd service mesh)

## GDPR Right to Erasure

The implementation varies dramatically by isolation model:

| Isolation Model     | Erasure Implementation                                                    |
| ------------------- | ------------------------------------------------------------------------- |
| Shared schema       | `DELETE FROM all_tables WHERE tenant_id = ?` — cascading deletes, complex |
| Schema-per-tenant   | `DROP SCHEMA tenant_abc CASCADE` — clean and atomic                       |
| Database-per-tenant | `DROP DATABASE tenant_abc` — cleanest possible                            |

## Audit Logging

Every data-modifying action must be logged with tenant context:

```typescript
interface AuditEvent {
  id: string;
  tenantId: string;
  actorId: string;         // who did it
  actorType: 'user' | 'super_admin' | 'system';
  action: string;          // 'order.created', 'user.deleted'
  resourceType: string;
  resourceId: string;
  previousState?: object;  // snapshot before change
  nextState?: object;      // snapshot after change
  ipAddress: string;
  userAgent: string;
  timestamp: Date;
}
```

Store audit logs in a **write-once** store (separate from main DB) that even super-admins cannot modify.
