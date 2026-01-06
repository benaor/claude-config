---
type: skill
name: factory
category: ddd/tactical
---

# Factory

## Detect

Scan for these patterns indicating missing or broken Factory:

| Pattern | Example |
|---------|---------|
| Constructor doing too much | Constructor with validation, side effects, queries |
| Factory returns invalid | Factory doesn't validate, accepts anything |
| Missing reconstitute method | No way to rebuild aggregate from persistence |
| Factory with infrastructure | Factory calls DB, generates IDs from sequence |
| Stateful factory | Factory with instance variables |
| Public constructor on aggregate | `new Order()` instead of `Order.create()` |

## Fix

Valid Factory structure:

```typescript
// Factory as static method (simple cases)
class Order {
  private constructor(
    private readonly id: OrderId,
    private readonly customerId: CustomerId,
    private items: OrderItem[],
    private status: OrderStatus
  ) {}

  // Creation factory - validates
  static place(id: OrderId, customerId: CustomerId, items: OrderItem[]): Order {
    if (items.length === 0) {
      throw new EmptyOrderError();
    }
    return new Order(id, customerId, items, OrderStatus.Placed);
  }

  // Reconstitution factory - trusts data
  static reconstitute(
    id: OrderId,
    customerId: CustomerId,
    items: OrderItem[],
    status: OrderStatus
  ): Order {
    return new Order(id, customerId, items, status);
  }
}

// Separate Factory class (complex cases)
class LoanApplicationFactory {
  constructor(
    private readonly creditScoreService: CreditScoreService,
    private readonly riskPolicy: RiskAssessmentPolicy
  ) {}

  create(applicantId: ApplicantId, amount: Money, term: LoanTerm): LoanApplication {
    const creditScore = this.creditScoreService.calculate(applicantId);
    const riskLevel = this.riskPolicy.assess(creditScore, amount);

    if (riskLevel === RiskLevel.Unacceptable) {
      throw new LoanApplicationRejectedError(applicantId);
    }

    return LoanApplication.create(applicantId, amount, term, riskLevel);
  }
}
```

Checklist:
- ✅ Private constructor + factory method
- ✅ Factory validates and fails fast
- ✅ Separate `reconstitute()` for persistence
- ✅ Stateless (no instance state between creations)
- ✅ No infrastructure calls

## Violations

### Constructor doing too much
```typescript
// ❌ Detect: constructor with logic, side effects
class Order {
  constructor(customerId: string, items: Array<{ productId: string; qty: number }>) {
    this.id = generateUUID();
    this.customerId = CustomerId.create(customerId);
    this.items = items.map(i => {
      const product = productCatalog.find(i.productId);
      return OrderItem.create(i.productId, i.qty, product.price);
    });
    emailService.sendConfirmation(this);
  }
}

// ✅ Fix: private constructor + factory method
class Order {
  private constructor(
    private readonly id: OrderId,
    private readonly customerId: CustomerId,
    private items: OrderItem[]
  ) {}

  static place(id: OrderId, customerId: CustomerId, items: OrderItem[]): Order {
    if (items.length === 0) {
      throw new EmptyOrderError();
    }
    return new Order(id, customerId, items);
  }
}
```

### Factory returns invalid object
```typescript
// ❌ Detect: factory accepts anything
class UserFactory {
  create(data: UserData): User {
    return new User(data.id, data.email, data.name);
  }
}

// ✅ Fix: factory validates
class UserFactory {
  create(data: UserData): User {
    const email = Email.create(data.email);
    const name = UserName.create(data.name);
    return User.register(UserId.create(data.id), email, name);
  }
}
```

### Missing reconstitution method
```typescript
// ❌ Detect: no way to rebuild from persistence
class Order {
  static place(id: OrderId, items: OrderItem[]): Order {
    return new Order(id, items, OrderStatus.Placed, new Date());
  }
  // How to rebuild a Shipped order from DB?
}

// ✅ Fix: separate reconstitute method
class Order {
  static place(id: OrderId, items: OrderItem[]): Order {
    if (items.length === 0) throw new EmptyOrderError();
    return new Order(id, items, OrderStatus.Placed, new Date());
  }

  static reconstitute(
    id: OrderId,
    items: OrderItem[],
    status: OrderStatus,
    placedAt: Date
  ): Order {
    return new Order(id, items, status, placedAt);
  }
}
```

### Factory with infrastructure dependencies
```typescript
// ❌ Detect: factory calls database
class OrderFactory {
  constructor(private readonly db: Database) {}

  async create(customerId: CustomerId): Promise<Order> {
    const id = await this.db.nextSequence('orders');
    const customer = await this.db.findCustomer(customerId);
    return Order.place(id, customer);
  }
}

// ✅ Fix: factory receives what it needs
class OrderFactory {
  create(id: OrderId, customerId: CustomerId, items: OrderItem[]): Order {
    return Order.place(id, customerId, items);
  }
}

// Application layer handles infrastructure
class PlaceOrderUseCase {
  async execute(command: PlaceOrderCommand): Promise<void> {
    const id = this.orderRepository.nextId();
    const items = this.buildItems(command.items);
    const order = Order.place(id, command.customerId, items);
    await this.orderRepository.save(order);
  }
}
```

### Stateful factory
```typescript
// ❌ Detect: factory with instance state
class InvoiceFactory {
  private lastNumber: number = 0;

  create(order: Order): Invoice {
    this.lastNumber++;
    return Invoice.create(InvoiceNumber.of(this.lastNumber), order);
  }
}

// ✅ Fix: stateless, sequence from repository
class InvoiceFactory {
  create(invoiceNumber: InvoiceNumber, order: Order): Invoice {
    return Invoice.create(invoiceNumber, order);
  }
}
```

## Skip when

- Simple construction — if just assigning values, constructor is fine
- Value Objects — usually `create()` static method is enough
- No invariants — if anything goes, factory adds no value
