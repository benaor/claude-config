---
type: skill
name: context-mapping
category: ddd/strategic
---

# Context Mapping

## Detect

Scan for these patterns indicating missing or broken Context Mapping:

| Pattern | Example |
|---------|---------|
| Missing ACL for external | External model (API, legacy) used directly in domain |
| Conformist when ACL needed | Adopting bad upstream model without translation |
| Oversized shared kernel | 50+ classes in shared-kernel folder |
| No translation layer | External DTO types in domain code |
| Unclear relationship | No documentation of upstream/downstream dependencies |

## Relationship patterns

| Pattern | Description | When to use |
|---------|-------------|-------------|
| **Customer-Supplier** | Downstream needs drive upstream | Teams can collaborate |
| **Conformist** | Downstream adopts upstream as-is | No leverage to influence |
| **Anti-Corruption Layer** | Downstream translates | Protect from legacy/external |
| **Shared Kernel** | Small shared model | Stable, minimal concepts |
| **Open Host Service** | Public API | Many consumers |

## Fix

### Anti-Corruption Layer (ACL)

```typescript
// External legacy model (ugly)
interface LegacyOrderResponse {
  ord_id: string;
  cust_num: string;
  ord_stat: 'N' | 'P' | 'S' | 'C';
}

// Our clean domain model
class Order {
  constructor(
    readonly id: OrderId,
    readonly customerId: CustomerId,
    readonly status: OrderStatus
  ) {}
}

// ACL translates between worlds
class LegacyOrderAdapter {
  async findOrder(id: OrderId): Promise<Order | null> {
    const legacy = await this.legacyClient.getOrder(id.value);
    if (!legacy) return null;
    return this.translate(legacy);
  }

  private translate(legacy: LegacyOrderResponse): Order {
    return Order.reconstitute(
      OrderId.create(legacy.ord_id),
      CustomerId.create(legacy.cust_num),
      this.translateStatus(legacy.ord_stat)
    );
  }

  private translateStatus(status: string): OrderStatus {
    const mapping: Record<string, OrderStatus> = {
      'N': OrderStatus.New,
      'P': OrderStatus.Processing,
      'S': OrderStatus.Shipped,
      'C': OrderStatus.Cancelled
    };
    return mapping[status] ?? OrderStatus.Unknown;
  }
}
```

### Shared Kernel (minimal)

```typescript
// shared-kernel/ — co-owned, changes require agreement
// ✅ Keep minimal
shared-kernel/
├── Money.ts
├── Currency.ts
└── DateRange.ts

// ❌ Too much
shared-kernel/
├── Customer.ts
├── Order.ts
├── Product.ts
└── ... 50 more
```

## Violations

### Missing ACL for external systems
```typescript
// ❌ Detect: external model leaks into domain
class OrderService {
  async importOrder(legacyData: LegacyOrderResponse): Promise<void> {
    const order = new Order(
      legacyData.ord_id,
      legacyData.cust_num,
      legacyData.ord_stat  // 'N', 'P', 'S' in our domain!
    );
  }
}

// ✅ Fix: ACL protects domain
class OrderService {
  constructor(private readonly legacyAdapter: LegacyOrderAdapter) {}

  async importOrder(legacyOrderId: string): Promise<void> {
    const order = await this.legacyAdapter.findOrder(
      OrderId.create(legacyOrderId)
    );
    // Clean domain model
  }
}
```

### Conformist when ACL needed
```typescript
// ❌ Detect: adopting bad upstream model directly
interface PaymentProviderResponse {
  txn_id: string;
  stat: 0 | 1 | 2 | -1;
  amt_cents: number;
}

class Payment {
  constructor(
    public txn_id: string,
    public stat: number,
    public amt_cents: number
  ) {}
}

// ✅ Fix: translate to our model
class Payment {
  constructor(
    readonly id: PaymentId,
    readonly status: PaymentStatus,
    readonly amount: Money
  ) {}
}

class PaymentProviderAdapter {
  translate(response: PaymentProviderResponse): Payment {
    return Payment.create(
      PaymentId.create(response.txn_id),
      this.mapStatus(response.stat),
      Money.ofCents(response.amt_cents, Currency.EUR)
    );
  }
}
```

### Oversized shared kernel
```typescript
// ❌ Detect: too much in shared kernel
shared-kernel/
├── Money.ts
├── Customer.ts
├── Order.ts
├── Product.ts
└── ... 50 more files

// ✅ Fix: minimal shared kernel
shared-kernel/
├── Money.ts
├── Currency.ts
├── DateRange.ts
└── Percentage.ts
```

## Skip when

- Single context — no map needed
- Early stage — map emerges as system grows
- No external integration — clean internal system
