# Module 10 — Billing, Metering & Usage Quotas

## Learning Objectives

- Implement usage metering as an architectural concern (not an afterthought)
- Understand usage-based vs. seat-based billing models
- Enforce soft and hard usage quotas

## Metering Architecture

Billing is a first-class concern in multi-tenant systems. Meter every meaningful unit of consumption from day one — retrofitting metering is extremely painful.

```mermaid
graph LR
    subgraph "Application Layer"
        A[API Handler]
        W[Background Worker]
    end

    subgraph "Metering Pipeline"
        EQ[Event Queue\nKafka / SQS]
        MA[Metering Aggregator]
        MS[(Metering Store\nTimeSeries DB)]
    end

    subgraph "Billing System"
        BS[Billing Service\nStripe / Lago]
        UD[Usage Dashboard]
    end

    A -->|Emit usage event\n{ tenant_id, metric, value }| EQ
    W -->|Emit usage event| EQ
    EQ --> MA
    MA -->|Aggregate per\ntenant per hour| MS
    MS --> BS
    BS --> UD
```

**Usage event structure:**

```typescript
interface UsageEvent {
  tenantId: string;
  metric: 'api_calls' | 'storage_gb' | 'active_users' | 'compute_minutes';
  value: number;
  timestamp: Date;
  metadata?: Record<string, string>; // e.g., { endpoint: '/api/reports' }
}
```

## Quota Enforcement

Quotas have two enforcement modes:

**Soft Quota:** Warn the tenant but allow continued use. Send email warning at 80% and 95%.

**Hard Quota:** Block requests that exceed the limit.

```typescript
@Injectable()
export class QuotaGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const { tenant } = context.switchToHttp().getRequest();
    
    const currentUsage = await this.meteringService.getCurrentUsage(
      tenant.id,
      'api_calls',
      'current_month'
    );

    if (currentUsage >= tenant.config.maxApiCallsPerMonth) {
      throw new HttpException(
        { error: 'quota_exceeded', limit: tenant.config.maxApiCallsPerMonth },
        HttpStatus.TOO_MANY_REQUESTS
      );
    }

    // Async fire-and-forget — don't slow the request
    this.meteringService.record({
      tenantId: tenant.id,
      metric: 'api_calls',
      value: 1,
      timestamp: new Date(),
    }).catch(err => this.logger.error('Metering failed', err));

    return true;
  }
}
```
