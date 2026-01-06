---
type: skill
name: ubiquitous-language
category: ddd/strategic
---

# Ubiquitous Language

## Detect

Scan for these patterns indicating broken Ubiquitous Language:

| Pattern | Example |
|---------|---------|
| Technical jargon | `OrderEntity`, `processOrderTransaction()`, `getOrderDTO()` |
| Generic CRUD naming | `create()`, `update()`, `delete()` instead of domain verbs |
| Same term different meanings | "Product" with different fields in different modules |
| Abbreviations | `poid`, `qty`, `calcTTV()` without clear meaning |
| Synonyms in code | `client`, `customer`, `buyer` for same concept |
| Technical boolean names | `isActive`, `isDeleted` instead of domain predicates |

## Fix

Code should mirror how domain experts describe the business:

```typescript
// Domain expert says:
// "A guest makes a reservation for a room. 
//  They can check in early if the room is available.
//  If they don't show up, we mark it as a no-show."

// ✅ Code mirrors this exactly
class Reservation {
  private constructor(
    private readonly id: ReservationId,
    private readonly guest: GuestId,
    private readonly room: RoomNumber,
    private readonly period: StayPeriod,
    private status: ReservationStatus
  ) {}

  static make(id: ReservationId, guest: GuestId, room: RoomNumber, period: StayPeriod): Reservation {
    return new Reservation(id, guest, room, period, ReservationStatus.Confirmed);
  }

  checkInEarly(currentTime: Date): void {
    if (!this.canCheckInEarly(currentTime)) {
      throw new EarlyCheckInNotAllowedError(this.id);
    }
    this.status = ReservationStatus.CheckedIn;
  }

  markAsNoShow(): void {
    if (this.status !== ReservationStatus.Confirmed) {
      throw new InvalidNoShowError(this.id);
    }
    this.status = ReservationStatus.NoShow;
  }
}
```

Checklist:
- ✅ Class/method names match domain expert vocabulary
- ✅ Domain verbs instead of CRUD (`register`, `activate`, not `create`, `update`)
- ✅ One term per concept, used consistently
- ✅ No unexplained abbreviations
- ✅ Predicates reflect business meaning

## Violations

### Technical jargon instead of domain terms
```typescript
// ❌ Detect: developer-speak
class OrderEntity {
  private orderStatusEnum: number;
  
  updateOrderStatus(statusCode: number): void {
    this.orderStatusEnum = statusCode;
  }
  
  processOrderTransaction(): void { }
  getOrderDTO(): OrderDataTransferObject { }
}

// ✅ Fix: domain language
class Order {
  private status: OrderStatus;

  confirm(): void {
    this.status = OrderStatus.Confirmed;
  }

  ship(carrier: Carrier, trackingNumber: TrackingNumber): void {
    this.status = OrderStatus.Shipped;
  }
}
```

### Generic CRUD naming
```typescript
// ❌ Detect: generic operations
class UserService {
  create(data: UserData): User { }
  update(id: string, data: Partial<UserData>): User { }
  delete(id: string): void { }
}

// ✅ Fix: domain-meaningful operations
class Customer {
  static register(email: Email, name: CustomerName): Customer { }
  changeAddress(newAddress: Address): void { }
  deactivate(reason: DeactivationReason): void { }
  reinstate(): void { }
}
```

### Same term, different meanings
```typescript
// ❌ Detect: "Product" means different things
class Product {
  description: string;      // Catalog concern
  price: Money;             // Pricing concern
  stockQuantity: number;    // Inventory concern
  weight: number;           // Shipping concern
}

// ✅ Fix: different terms per context
// Catalog: Product
// Inventory: StockItem
// Shipping: Parcel
```

### Abbreviations without definition
```typescript
// ❌ Detect: unclear abbreviations
class Order {
  poid: string;
  qty: number;
  calcTTV(): Money { }
}

// ✅ Fix: full domain terms
class Order {
  purchaseOrderId: PurchaseOrderId;
  quantity: Quantity;
  calculateTotalTransactionValue(): Money { }
}
```

### Synonyms in code
```typescript
// ❌ Detect: multiple terms for same concept
class OrderService {
  getClient(id: string): Customer { }
  findCustomer(id: string): Customer { }
  loadBuyer(id: string): Customer { }
}

// ✅ Fix: one term, used consistently
// Pick "Customer" and use it everywhere
```

### Boolean naming that doesn't match domain
```typescript
// ❌ Detect: technical boolean names
class Subscription {
  isActive: boolean;
  hasPaymentMethod: boolean;
  isDeleted: boolean;
}

if (subscription.isActive && !subscription.isDeleted) { }

// ✅ Fix: domain predicates
class Subscription {
  isInGoodStanding(): boolean {
    return this.status === SubscriptionStatus.Active
      && this.hasValidPaymentMethod();
  }

  isCancelled(): boolean {
    return this.status === SubscriptionStatus.Cancelled;
  }
}

if (subscription.isInGoodStanding()) { }
```

## Skip when

- Pure technical code — infrastructure, utilities
- Standard patterns — Repository, Factory are OK as technical terms
- External APIs — third-party conventions, use ACL to translate
