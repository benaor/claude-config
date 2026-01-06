---
type: skill
name: entity
category: ddd/tactical
---

# Entity

## Detect

Scan for these patterns indicating missing or broken Entity:

| Pattern | Example |
|---------|---------|
| Primitive ID | `id: string` instead of typed `UserId` |
| Public mutable ID | `public id: string` that can change |
| Equality by attributes | `equals()` comparing name, email instead of ID |
| No lifecycle methods | Only setters, no domain verbs |
| Public setters everywhere | `public status: string` |
| ID on value concept | `Address` with `id` field (should be VO) |
| Business logic outside | `userService.activate(user)` instead of `user.activate()` |

## Fix

Valid Entity structure:

```typescript
class Order {
  private constructor(
    private readonly id: OrderId,
    private status: OrderStatus,
    private items: OrderItem[],
    private readonly createdAt: Date
  ) {}

  static create(id: OrderId, items: OrderItem[]): Order {
    if (items.length === 0) {
      throw new EmptyOrderError();
    }
    return new Order(id, OrderStatus.Pending, items, new Date());
  }

  confirm(): void {
    this.assertModifiable();
    this.status = OrderStatus.Confirmed;
  }

  private assertModifiable(): void {
    if (this.status !== OrderStatus.Pending) {
      throw new OrderNotModifiableError(this.id);
    }
  }

  equals(other: Order): boolean {
    return this.id.equals(other.id);
  }
}
```

Checklist:
- ✅ Typed immutable ID (`OrderId`, not `string`)
- ✅ `equals()` compares ID only
- ✅ State changes via domain methods (`confirm()`, not `setStatus()`)
- ✅ Invariants enforced internally
- ✅ Uses Value Objects for attributes

## Violations

### Equality by attributes
```typescript
// ❌ Detect: equals() comparing properties instead of ID
equals(other: User): boolean {
  return this.name === other.name && this.email === other.email;
}

// ✅ Fix: compare identity only
equals(other: User): boolean {
  return this.id.equals(other.id);
}
```

### Anemic Entity
```typescript
// ❌ Detect: public setters + logic in external service
class User {
  public name: string;
  public email: string;
  public status: string;
}

userService.activate(user);
userService.changeName(user, newName);

// ✅ Fix: encapsulate behavior
class User {
  private constructor(
    private readonly id: UserId,
    private name: UserName,
    private status: UserStatus
  ) {}

  activate(): void {
    if (this.status !== UserStatus.Pending) {
      throw new InvalidActivationError(this.id);
    }
    this.status = UserStatus.Active;
  }

  rename(newName: UserName): void {
    this.name = newName;
  }
}
```

### Mutable or missing ID
```typescript
// ❌ Detect: public id or id assigned after construction
class Product {
  public id: string;
  constructor(public name: string) {
    this.id = generateId();
  }
}

// ✅ Fix: readonly ID from construction
class Product {
  private constructor(
    private readonly id: ProductId,
    private name: ProductName
  ) {}

  static create(id: ProductId, name: ProductName): Product {
    return new Product(id, name);
  }
}
```

### Primitive ID
```typescript
// ❌ Detect: string/number ID allows type confusion
function getOrder(id: string): Order
getOrder(customerId); // Compiles but wrong!

// ✅ Fix: typed ID prevents mixups
function getOrder(id: OrderId): Order
getOrder(customerId); // Compile error
```

### Entity instead of Value Object
```typescript
// ❌ Detect: ID on concept with no lifecycle
class Address {
  constructor(
    readonly id: string,
    readonly street: string,
    readonly city: string
  ) {}
}

// ✅ Fix: remove ID, make it Value Object
class Address {
  private constructor(
    readonly street: Street,
    readonly city: City
  ) {}
}
```

## Skip when

- No lifecycle — if it never changes, use Value Object
- Identity doesn't matter — if same values = same thing, use Value Object
- Cross-aggregate reference — store ID only, not full entity
