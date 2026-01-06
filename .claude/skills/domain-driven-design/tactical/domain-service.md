---
type: skill
name: domain-service
category: ddd/tactical
---

# Domain Service

## Detect

Scan for these patterns indicating missing or broken Domain Service:

| Pattern | Example |
|---------|---------|
| Stateful service | Service with instance variables between calls |
| Infrastructure in domain service | Service calls DB, HTTP, email directly |
| Logic that belongs in entity | Service does what entity method should do |
| Anemic delegation | Service just calls entity setters |
| Technical naming | `OrderManager`, `OrderHelper`, `OrderUtils` |
| Calculation as service | Stateless calc that should be Value Object |

## Fix

Valid Domain Service structure:

```typescript
// Logic spanning multiple aggregates
class MoneyTransferService {
  transfer(
    source: Account,
    destination: Account,
    amount: Money
  ): TransferResult {
    if (!source.canWithdraw(amount)) {
      return TransferResult.insufficientFunds();
    }

    if (!destination.canReceive(amount)) {
      return TransferResult.destinationRejected();
    }

    source.withdraw(amount);
    destination.deposit(amount);

    return TransferResult.success(source, destination);
  }
}

// Complex business rules spanning entities
class PricingService {
  calculatePrice(
    product: Product,
    customer: Customer,
    quantity: Quantity
  ): Money {
    let price = product.basePrice.multiply(quantity.value);

    if (customer.isVip()) {
      price = price.applyDiscount(Percentage.of(10));
    }

    return price;
  }
}
```

Checklist:
- ✅ Stateless (all data passed as parameters)
- ✅ Pure domain logic (no infrastructure)
- ✅ Named after domain concept
- ✅ Operates on domain objects
- ✅ Used when logic doesn't fit single entity

## Violations

### Stateful service
```typescript
// ❌ Detect: service with instance state
class OrderProcessingService {
  private currentOrder: Order;
  private processingStarted: Date;

  startProcessing(order: Order): void {
    this.currentOrder = order;
    this.processingStarted = new Date();
  }

  complete(): void {
    this.currentOrder.complete();
  }
}

// ✅ Fix: stateless, all data passed in
class OrderProcessingService {
  process(order: Order, processor: Employee): ProcessingResult {
    order.startProcessing(processor.id);
    order.complete();
    return ProcessingResult.success(order);
  }
}
```

### Infrastructure in domain service
```typescript
// ❌ Detect: domain service with DB/HTTP/email dependencies
class InvoiceService {
  constructor(
    private readonly db: Database,
    private readonly emailClient: EmailClient
  ) {}

  generateInvoice(order: Order): Invoice {
    const invoice = Invoice.fromOrder(order);
    await this.db.save(invoice);
    await this.emailClient.send(invoice);
    return invoice;
  }
}

// ✅ Fix: pure domain logic, application layer orchestrates
class InvoiceService {
  generateInvoice(order: Order, invoiceNumber: InvoiceNumber): Invoice {
    return Invoice.fromOrder(order, invoiceNumber);
  }
}

class GenerateInvoiceUseCase {
  async execute(orderId: OrderId): Promise<void> {
    const order = await this.orderRepository.findById(orderId);
    const invoiceNumber = await this.invoiceRepository.nextNumber();
    const invoice = this.invoiceService.generateInvoice(order, invoiceNumber);
    await this.invoiceRepository.save(invoice);
    await this.emailGateway.sendInvoice(invoice);
  }
}
```

### Logic that belongs in Entity
```typescript
// ❌ Detect: service doing entity's job
class OrderService {
  addItem(order: Order, product: Product, quantity: number): void {
    if (order.status !== 'draft') {
      throw new Error('Cannot modify');
    }
    const item = new OrderItem(product.id, quantity, product.price);
    order.items.push(item);
  }
}

// ✅ Fix: move logic to entity
class Order {
  addItem(productId: ProductId, quantity: Quantity, price: Money): void {
    this.assertModifiable();
    this.items.push(OrderItem.create(productId, quantity, price));
  }
}
```

### Anemic service (just delegation)
```typescript
// ❌ Detect: service methods just call setters
class UserService {
  changeName(user: User, name: string): void {
    user.setName(name);
  }

  changeEmail(user: User, email: string): void {
    user.setEmail(email);
  }
}

// ✅ Fix: no service needed, call entity directly
user.rename(newName);
user.changeEmail(newEmail);
```

### Technical naming
```typescript
// ❌ Detect: generic technical names
class OrderHelper {}
class OrderProcessor {}
class OrderManager {}
class OrderUtils {}

// ✅ Fix: domain concept names
class OrderFulfillmentService {}
class PricingService {}
class ShippingCostCalculator {}
```

### Service instead of Value Object
```typescript
// ❌ Detect: stateless calculation as service
class TaxCalculationService {
  calculate(amount: Money, rate: number): Money {
    return amount.multiply(rate);
  }
}

// ✅ Fix: Value Object encapsulates concept
class TaxRate {
  private constructor(private readonly rate: number) {}

  static of(percentage: number): TaxRate {
    return new TaxRate(percentage / 100);
  }

  applyTo(amount: Money): Money {
    return amount.multiply(this.rate);
  }
}
```

## Skip when

- Logic fits in Entity — default to entity methods first
- Logic fits in Value Object — calculations often belong in VOs
- Just CRUD orchestration — that's Application Service
- Infrastructure needed — Domain Services are pure domain
