---
type: skill
name: aggregate
category: ddd/tactical
---

# Aggregate

## Detect

Scan for these patterns indicating missing or broken Aggregate:

| Pattern | Example |
|---------|---------|
| Internal entity exposed | `getItems(): OrderItem[]` returns mutable array |
| Direct reference to other aggregate | `private customer: Customer` instead of `customerId` |
| Transaction spans aggregates | One DB transaction saves multiple aggregates |
| No invariant protection | Public properties allow invalid state |
| Aggregate too large | Loading requires many tables/queries |
| Bypassing root | `order.items[0].changePrice()` directly |

## Fix

Valid Aggregate structure:

```typescript
class ShoppingCart {
  private constructor(
    private readonly id: CartId,
    private readonly customerId: CustomerId,
    private items: CartItem[],
    private status: CartStatus
  ) {}

  static create(id: CartId, customerId: CustomerId): ShoppingCart {
    return new ShoppingCart(id, customerId, [], CartStatus.Active);
  }

  addItem(productId: ProductId, quantity: Quantity, unitPrice: Money): void {
    this.assertActive();
    const existing = this.items.find(i => i.productId.equals(productId));
    if (existing) {
      existing.increaseQuantity(quantity);
    } else {
      this.items.push(CartItem.create(productId, quantity, unitPrice));
    }
  }

  checkout(): void {
    this.assertActive();
    if (this.items.length === 0) {
      throw new EmptyCartError(this.id);
    }
    this.status = CartStatus.CheckedOut;
  }

  private assertActive(): void {
    if (this.status !== CartStatus.Active) {
      throw new CartNotActiveError(this.id);
    }
  }
}
```

Checklist:
- ✅ Single entry point (all changes through root)
- ✅ Reference other aggregates by ID only
- ✅ Internal entities never exposed directly
- ✅ Invariants enforced at all times
- ✅ Small and focused

## Violations

### Exposing internal entities
```typescript
// ❌ Detect: getter returns internal collection
class Order {
  getItems(): OrderItem[] {
    return this.items;
  }
}

order.getItems().push(newItem); // Bypasses invariants

// ✅ Fix: controlled access only
class Order {
  addItem(item: OrderItem): void {
    this.assertCanAddItem();
    this.items.push(item);
  }

  get itemsSummary(): ReadonlyArray<ItemSummary> {
    return this.items.map(i => i.toSummary());
  }
}
```

### Reference by object instead of ID
```typescript
// ❌ Detect: aggregate holds reference to another aggregate
class Order {
  constructor(
    private readonly id: OrderId,
    private readonly customer: Customer
  ) {}
}

// ✅ Fix: reference by ID only
class Order {
  constructor(
    private readonly id: OrderId,
    private readonly customerId: CustomerId
  ) {}
}
```

### Transaction spanning multiple aggregates
```typescript
// ❌ Detect: single transaction modifies multiple aggregates
async function transferMoney(from: Account, to: Account, amount: Money) {
  await db.transaction(async (tx) => {
    from.withdraw(amount);
    to.deposit(amount);
    await tx.save(from);
    await tx.save(to);
  });
}

// ✅ Fix: eventual consistency via domain events
class Account {
  withdraw(amount: Money): void {
    this.balance = this.balance.subtract(amount);
    this.addDomainEvent(new MoneyWithdrawn(this.id, amount));
  }
}
```

### Aggregate too large
```typescript
// ❌ Detect: aggregate with many collections/entities
class Company {
  private employees: Employee[];
  private departments: Department[];
  private projects: Project[];
  private invoices: Invoice[];
}

// ✅ Fix: separate aggregates with ID references
class Employee {
  private readonly companyId: CompanyId;
  private departmentId: DepartmentId;
}
```

### No invariant protection
```typescript
// ❌ Detect: public properties allow invalid state
class Booking {
  public startDate: Date;
  public endDate: Date;
}

// ✅ Fix: enforce invariants
class Booking {
  private constructor(
    private readonly id: BookingId,
    private readonly period: DateRange
  ) {}

  reschedule(newPeriod: DateRange): void {
    if (this.status !== BookingStatus.Pending) {
      throw new BookingNotReschedulableError(this.id);
    }
    this.period = newPeriod;
  }
}
```

## Skip when

- Simple CRUD — no invariants to protect
- Read models — queries can span aggregates
- Reporting — aggregates are for writes, not complex reads
