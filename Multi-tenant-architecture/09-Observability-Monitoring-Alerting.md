# Module 9 — Observability, Monitoring & Alerting

## Learning Objectives

- Tag all telemetry (logs, metrics, traces) with tenant context
- Build per-tenant dashboards in Grafana
- Detect and alert on per-tenant SLA breaches

## The Golden Rule of Multi-Tenant Observability

> Every log line, metric data point, and trace span MUST carry a `tenant_id` label. Without this, you cannot distinguish a platform-wide outage from a single-tenant issue.

## OpenTelemetry Tenant Enrichment

```typescript
// otel-tenant-instrumentation.ts
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

export function withTenantSpan<T>(
  spanName: string,
  tenantId: string,
  fn: () => Promise<T>
): Promise<T> {
  const tracer = trace.getTracer('platform');
  return tracer.startActiveSpan(spanName, async (span) => {
    // Tag every span with tenant context
    span.setAttribute('tenant.id', tenantId);
    span.setAttribute('tenant.tier', getCurrentTenant().tier);
    try {
      const result = await fn();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      span.recordException(err);
      throw err;
    } finally {
      span.end();
    }
  });
}
```

## Key Metrics to Track Per Tenant

| Metric                                 | Why It Matters                  | Alert Threshold |
| -------------------------------------- | ------------------------------- | --------------- |
| `http_request_duration_p99{tenant_id}` | Detect slow tenants             | > 2s for 5 min  |
| `db_query_duration_p95{tenant_id}`     | Slow query detection            | > 500ms         |
| `error_rate{tenant_id}`                | Detect tenant-specific failures | > 1% for 2 min  |
| `active_connections{tenant_id}`        | Connection pool exhaustion      | > 80% of limit  |
| `storage_used_gb{tenant_id}`           | Quota enforcement               | > 90% of limit  |
| `api_requests_per_minute{tenant_id}`   | Rate limit enforcement          | > quota         |

## Grafana Dashboard Structure

```
Platform Overview (cross-tenant)
├── Cluster health (CPU, memory, pods)
├── Top 10 tenants by request volume
├── Error rate by tenant (heat map)
└── P99 latency by tenant

Per-Tenant Drill-Down (parameterized by tenant_id)
├── Request rate (last 1h, 24h, 7d)
├── Error breakdown (4xx vs 5xx)
├── DB query latency percentiles
├── Active users
└── Storage usage vs quota
```
