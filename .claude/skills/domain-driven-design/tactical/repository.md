---
type: skill
name: repository
category: ddd/tactical
---

# Repository

## Detect

Scan for these patterns indicating missing or broken Repository:

| Pattern | Example |
|---------|---------|
| Repository for internal entity | `OrderItemRepository` instead of accessing via `OrderRepository` |
| Persistence leaking to domain | Interface exposes SQL, joins, connections |
| Partial aggregate returned | `findById` loads order but not items |
| Generic repository | `Repository<T>` with same methods for all types |
| Business logic in repository | Repository calls domain methods, orchestrates |
| Returns DTOs/primitives | `findTotalsByCustomer(): { total: number }` |

## Fix

Valid Repository structure:

```typescript
// Domain layer - interface (port)
interface OrderRepository {
  nextId(): OrderId;
  findById(id: OrderId): Promise<Order | null>;
  findByCustomer(customerId: CustomerId): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(order: Order): Promise<void>;
}

// Infrastructure layer - implementation (adapter)
class PostgresOrderRepository implements OrderRepository {
  constructor(private readonly db: Database) {}

  nextId(): OrderId {
    return OrderId.create(crypto.randomUUID());
  }

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.query('SELECT * FROM orders WHERE id = $1', [id.value]);
    if (!row) return null;
    return this.toDomain(row);
  }

  async save(order: Order): Promise<void> {
    const data = this.toPersistence(order);
    await this.db.upsert('orders', data);
  }

  private toDomain(row: OrderRow): Order { /* reconstitute */ }
  private toPersistence(order: Order): OrderRow { /* map */ }
}
```

Checklist:
- ✅ One repository per aggregate root
- ✅ Interface in domain, implementation in infrastructure
- ✅ Collection-like semantics (`find`, `save`, `delete`)
- ✅ Returns complete, valid aggregates
- ✅ Domain-specific query methods

## Violations

### Repository for non-root entity
```typescript
// ❌ Detect: repository for internal entity
interface OrderItemRepository {
  findById(id: OrderItemId): Promise<OrderItem>;
  save(item: OrderItem): Promise<void>;
}

// Bypasses aggregate invariants!
const item = await orderItemRepository.findById(itemId);
item.changePrice(newPrice);

// ✅ Fix: access through aggregate root only
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
}

const order = await orderRepository.findById(orderId);
order.updateItemPrice(itemId, newPrice);
```

### Leaking persistence concerns
```typescript
// ❌ Detect: domain interface exposes DB concepts
interface OrderRepository {
  findBySQL(query: string): Promise<Order[]>;
  findWithJoin(tables: string[]): Promise<Order[]>;
  getConnection(): DbConnection;
}

// ✅ Fix: domain-focused interface
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findPending(): Promise<Order[]>;
  findByCustomerAndStatus(customerId: CustomerId, status: OrderStatus): Promise<Order[]>;
}
```

### Returning partial aggregates
```typescript
// ❌ Detect: aggregate returned without children loaded
class OrderRepository {
  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
    return Order.reconstitute(row.id, row.status, []);  // Items missing!
  }
}

// ✅ Fix: return complete aggregate
class OrderRepository {
  async findById(id: OrderId): Promise<Order | null> {
    const orderRow = await this.db.query('SELECT * FROM orders WHERE id = $1', [id]);
    const itemRows = await this.db.query('SELECT * FROM order_items WHERE order_id = $1', [id]);
    
    return Order.reconstitute(orderRow.id, orderRow.status, itemRows.map(this.toOrderItem));
  }
}
```

### Generic repository
```typescript
// ❌ Detect: same interface for all aggregates
interface Repository<T> {
  findById(id: string): Promise<T>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<void>;
  delete(id: string): Promise<void>;
}

// ✅ Fix: specific repository per aggregate
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findPendingOlderThan(date: Date): Promise<Order[]>;
  save(order: Order): Promise<void>;
}
```

### Business logic in repository
```typescript
// ❌ Detect: repository contains domain logic
class OrderRepository {
  async cancelExpiredOrders(): Promise<void> {
    const orders = await this.findExpired();
    for (const order of orders) {
      order.status = 'cancelled';
      order.cancelledAt = new Date();
      await this.save(order);
    }
  }
}

// ✅ Fix: domain service orchestrates
class OrderExpirationService {
  async cancelExpiredOrders(): Promise<void> {
    const orders = await this.orderRepository.findExpired();
    for (const order of orders) {
      order.cancel();
      await this.orderRepository.save(order);
    }
  }
}
```

### Query methods returning wrong types
```typescript
// ❌ Detect: repository returns DTOs or primitives
interface OrderRepository {
  findTotalsByCustomer(customerId: CustomerId): Promise<{ total: number }>;
  findOrderStatuses(): Promise<string[]>;
}

// ✅ Fix: repositories return aggregates, separate read model for queries
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findByCustomer(customerId: CustomerId): Promise<Order[]>;
}

interface OrderReadModel {
  getTotalsByCustomer(customerId: string): Promise<CustomerTotalsDto>;
}
```

## Skip when

- Read models — queries can bypass repositories
- Simple value lookups — reference data might not need full pattern
- Cross-aggregate reports — use query services
