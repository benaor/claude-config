---
type: skill
name: specification
category: ddd/tactical
---

# Specification

## Detect

Scan for these patterns indicating missing or broken Specification:

| Pattern | Example |
|---------|---------|
| Technical naming | `CustomerHasTotalGreaterThan1000` instead of domain term |
| Specification with side effects | Spec that modifies state or logs |
| God specification | One spec checking 20+ conditions |
| Duplicated condition logic | Same `if` repeated in multiple files |
| Not using composition | Multiple specs with duplicated checks |
| Inline conditions | Complex business rule as inline `if` |

## Fix

Valid Specification structure:

```typescript
interface Specification<T> {
  isSatisfiedBy(candidate: T): boolean;
  and(other: Specification<T>): Specification<T>;
  or(other: Specification<T>): Specification<T>;
  not(): Specification<T>;
}

abstract class CompositeSpecification<T> implements Specification<T> {
  abstract isSatisfiedBy(candidate: T): boolean;

  and(other: Specification<T>): Specification<T> {
    return new AndSpecification(this, other);
  }

  or(other: Specification<T>): Specification<T> {
    return new OrSpecification(this, other);
  }

  not(): Specification<T> {
    return new NotSpecification(this);
  }
}

// Concrete specification
class OrderIsShippable extends CompositeSpecification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return order.isPaid() 
      && order.hasValidShippingAddress()
      && !order.hasBackorderedItems();
  }
}

// Composed usage
const eligibleForDiscount = new CustomerIsEligibleForDiscount();
const isVip = new CustomerIsVip();
const eligibleVip = eligibleForDiscount.and(isVip);

if (eligibleVip.isSatisfiedBy(customer)) {
  // apply discount
}
```

Checklist:
- ✅ Named after business concept
- ✅ Pure predicate (no side effects)
- ✅ Composable (and, or, not)
- ✅ Single responsibility (one rule per spec)
- ✅ Reusable across codebase

## Violations

### Technical naming
```typescript
// ❌ Detect: named after implementation
class CustomerHasTotalPurchasesGreaterThan1000 extends Specification<Customer> {}
class OrderStatusEqualsShipped extends Specification<Order> {}

// ✅ Fix: named after business concept
class CustomerIsEligibleForDiscount extends Specification<Customer> {}
class OrderIsReadyForShipment extends Specification<Order> {}
```

### Specification with side effects
```typescript
// ❌ Detect: modifies state in isSatisfiedBy
class OrderIsValid extends Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    if (order.items.length === 0) {
      order.addError('No items');
      return false;
    }
    order.markAsValidated();
    return true;
  }
}

// ✅ Fix: pure predicate
class OrderIsValid extends Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return order.items.length > 0 
      && order.hasValidCustomer()
      && order.totalAmount.isPositive();
  }
}
```

### God specification
```typescript
// ❌ Detect: one spec checking everything
class OrderIsProcessable extends Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return order.items.length > 0
      && order.customer !== null
      && order.customer.isActive
      && order.shippingAddress !== null
      && order.isPaid
      // ... 15 more conditions
  }
}

// ✅ Fix: compose from focused specs
class OrderIsProcessable extends CompositeSpecification<Order> {
  private readonly spec = new OrderHasItems()
    .and(new OrderHasActiveCustomer())
    .and(new OrderHasValidShippingAddress())
    .and(new OrderIsPaid());

  isSatisfiedBy(order: Order): boolean {
    return this.spec.isSatisfiedBy(order);
  }
}
```

### Duplicated condition logic
```typescript
// ❌ Detect: same rule in multiple places
class OrderService {
  canShip(order: Order): boolean {
    return order.status === 'paid' && order.items.every(i => i.inStock);
  }
}

class OrderController {
  ship(orderId: string): void {
    const order = this.repository.findById(orderId);
    if (order.status === 'paid' && order.items.every(i => i.inStock)) {
      // duplicate!
    }
  }
}

// ✅ Fix: single specification, reused
const shippable = new OrderIsShippable();

class OrderService {
  canShip(order: Order): boolean {
    return shippable.isSatisfiedBy(order);
  }
}

class OrderController {
  ship(orderId: string): void {
    const order = this.repository.findById(orderId);
    if (shippable.isSatisfiedBy(order)) { /**/ }
  }
}
```

### Not using composition
```typescript
// ❌ Detect: separate specs with duplicated logic
class PremiumCustomerDiscount extends Specification<Customer> {
  isSatisfiedBy(c: Customer): boolean {
    return c.isPremium() && c.totalPurchases.isGreaterThan(Money.of(500));
  }
}

class PremiumCustomerFreeShipping extends Specification<Customer> {
  isSatisfiedBy(c: Customer): boolean {
    return c.isPremium() && c.totalPurchases.isGreaterThan(Money.of(500));
  }
}

// ✅ Fix: compose from atomic specs
const isPremium = new CustomerIsPremium();
const hasHighPurchases = new CustomerHasMinimumPurchases(Money.of(500));
const premiumHighValue = isPremium.and(hasHighPurchases);
```

## Skip when

- Simple condition — `if (order.isPaid())` doesn't need a spec
- One-time use — if rule is used only once, inline is fine
- Non-domain logic — technical validations don't need specs
