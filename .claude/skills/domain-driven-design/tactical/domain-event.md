---
type: skill
name: domain-event
category: ddd/tactical
---

# Domain Event

## Detect

Scan for these patterns indicating missing or broken Domain Event:

| Pattern | Example |
|---------|---------|
| Present/future tense name | `PlaceOrder`, `OrderPlacing` instead of `OrderPlaced` |
| Mutable event | Event with setters or methods that modify state |
| Missing data | Event with only ID, handler must query for details |
| Event before state change | `addEvent()` called before validation passes |
| Technical event | `DatabaseRowInserted`, `EmailSent` |
| Synchronous coupling | Aggregate directly calls services instead of raising events |

## Fix

Valid Domain Event structure:

```typescript
abstract class DomainEvent {
  readonly occurredAt: Date = new Date();
  abstract get eventName(): string;
}

class OrderPlaced extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    readonly items: ReadonlyArray<OrderItemSnapshot>,
    readonly totalAmount: Money
  ) {
    super();
  }

  get eventName(): string {
    return 'OrderPlaced';
  }
}

// Aggregate raises events
class Order {
  private domainEvents: DomainEvent[] = [];

  static create(id: OrderId, customerId: CustomerId, items: OrderItem[]): Order {
    const order = new Order(id, customerId, items, OrderStatus.Placed);
    order.addDomainEvent(new OrderPlaced(id, customerId, items.map(i => i.toSnapshot()), order.totalAmount));
    return order;
  }

  pullDomainEvents(): DomainEvent[] {
    const events = [...this.domainEvents];
    this.domainEvents = [];
    return events;
  }
}
```

Checklist:
- ✅ Past tense naming (`OrderPlaced`, not `PlaceOrder`)
- ✅ Immutable (all `readonly` properties)
- ✅ Contains data handlers need
- ✅ Raised after successful state change
- ✅ Represents domain concept, not technical action

## Violations

### Present or future tense naming
```typescript
// ❌ Detect: event name not past tense
class PlaceOrder extends DomainEvent {}
class OrderPlacing extends DomainEvent {}
class OrderToBeShipped extends DomainEvent {}

// ✅ Fix: past tense
class OrderPlaced extends DomainEvent {}
class OrderShipped extends DomainEvent {}
class PaymentReceived extends DomainEvent {}
```

### Mutable event
```typescript
// ❌ Detect: public properties or mutation methods
class UserRegistered extends DomainEvent {
  public userId: string;
  public email: string;
  
  markAsProcessed(): void {
    this.processed = true;
  }
}

// ✅ Fix: all readonly
class UserRegistered extends DomainEvent {
  constructor(
    readonly userId: UserId,
    readonly email: Email,
    readonly registeredAt: Date
  ) {
    super();
  }
}
```

### Missing essential data
```typescript
// ❌ Detect: event with only ID, handler queries for rest
class OrderPlaced extends DomainEvent {
  constructor(readonly orderId: OrderId) {
    super();
  }
}

// ✅ Fix: include what handlers need
class OrderPlaced extends DomainEvent {
  constructor(
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    readonly items: ReadonlyArray<OrderItemSnapshot>,
    readonly totalAmount: Money
  ) {
    super();
  }
}
```

### Event raised before state change
```typescript
// ❌ Detect: addEvent before validation/mutation
class Account {
  withdraw(amount: Money): void {
    this.addDomainEvent(new MoneyWithdrawn(this.id, amount));
    
    if (this.balance.isLessThan(amount)) {
      throw new InsufficientFundsError();
    }
    this.balance = this.balance.subtract(amount);
  }
}

// ✅ Fix: raise event after success
class Account {
  withdraw(amount: Money): void {
    if (this.balance.isLessThan(amount)) {
      throw new InsufficientFundsError();
    }
    this.balance = this.balance.subtract(amount);
    
    this.addDomainEvent(new MoneyWithdrawn(this.id, amount));
  }
}
```

### Technical event instead of domain event
```typescript
// ❌ Detect: infrastructure/technical naming
class DatabaseRowInserted extends DomainEvent {}
class EmailSent extends DomainEvent {}
class CacheInvalidated extends DomainEvent {}

// ✅ Fix: business meaning
class CustomerRegistered extends DomainEvent {}
class OrderFulfilled extends DomainEvent {}
```

### Synchronous handler coupling
```typescript
// ❌ Detect: aggregate calls services directly
class Order {
  place(): void {
    this.status = OrderStatus.Placed;
    inventoryService.reserveStock(this.items);
    emailService.sendConfirmation(this.customerId);
  }
}

// ✅ Fix: raise events, handlers react
class Order {
  place(): void {
    this.status = OrderStatus.Placed;
    this.addDomainEvent(new OrderPlaced(/*...*/));
  }
}
```

## Skip when

- Internal state change — not every setter needs an event
- Query operations — events are for writes
- No subscribers — don't create events nobody listens to
