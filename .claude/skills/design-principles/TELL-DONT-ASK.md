---
name: tell-dont-ask
description: Tell Don't Ask principle for TypeScript code review and refactoring. Use when detecting procedural code that extracts data from objects to make decisions, anemic domain models, or logic that should be encapsulated. Helps improve encapsulation and cohesion.
---

# Tell, Don't Ask

Tell objects what to do; don't ask them for data and make decisions for them. Behavior should live with the data it operates on.

## Violation Signs

- Getting data from an object, then making decisions based on it
- External code checking object state before calling methods
- Getters used primarily for decision-making elsewhere
- Business logic scattered outside domain objects
- Feature envy: methods more interested in another class's data

## Example

**Before — Ask (Violation):**

```typescript
// OrderService.ts — asks Order for data, makes decisions externally
class OrderService {
  applyDiscount(order: Order, discountCode: string): void {
    // ASK: Get data from order
    const items = order.getItems();
    const customer = order.getCustomer();
    const currentTotal = order.getTotal();

    // External decision-making with order's data
    if (items.length < 3) {
      throw new Error('Minimum 3 items for discount');
    }

    if (customer.getMembershipLevel() !== 'premium') {
      throw new Error('Premium members only');
    }

    if (currentTotal < 100) {
      throw new Error('Minimum order $100');
    }

    // Calculate discount externally
    const discount = this.discountRepository.getDiscount(discountCode);
    const discountedTotal = currentTotal * (1 - discount.percentage);

    // ASK again, then mutate
    order.setTotal(discountedTotal);
    order.setDiscountCode(discountCode);
  }
}

// What's wrong:
// - OrderService knows too much about Order's rules
// - Order is just a data bag (anemic)
// - Discount rules are spread across services
```

**After — Tell (Correct):**

```typescript
// Order.ts — encapsulates its own rules and behavior
class Order {
  private discountCode: string | null = null;

  constructor(
    private readonly items: OrderItem[],
    private readonly customer: Customer,
    private total: number
  ) {}

  applyDiscount(discount: Discount): void {
    // TELL: Order decides if discount is applicable
    this.validateDiscountEligibility(discount);
    
    // TELL: Order updates itself
    this.total = this.total * (1 - discount.percentage);
    this.discountCode = discount.code;
  }

  private validateDiscountEligibility(discount: Discount): void {
    if (discount.minimumItems && this.items.length < discount.minimumItems) {
      throw new DiscountError(`Minimum ${discount.minimumItems} items required`);
    }

    if (discount.requiredMembership && !this.customer.hasMembership(discount.requiredMembership)) {
      throw new DiscountError(`${discount.requiredMembership} membership required`);
    }

    if (discount.minimumTotal && this.total < discount.minimumTotal) {
      throw new DiscountError(`Minimum order $${discount.minimumTotal} required`);
    }
  }

  // Queries for display purposes are fine
  getTotal(): number {
    return this.total;
  }
}

// Customer.ts — encapsulates membership logic
class Customer {
  hasMembership(level: MembershipLevel): boolean {
    return this.membershipLevel === level || this.isHigherTier(level);
  }
}

// OrderService.ts — coordinates, doesn't decide
class OrderService {
  async applyDiscount(orderId: string, discountCode: string): Promise<void> {
    const order = await this.orderRepository.getById(orderId);
    const discount = await this.discountRepository.getByCode(discountCode);

    // TELL the order to apply discount — order decides if it can
    order.applyDiscount(discount);

    await this.orderRepository.save(order);
  }
}
```

## Anti-Pattern: Anemic Domain Model

Objects that are just data containers with getters/setters:

```typescript
// ❌ Anemic: just a data bag
class User {
  private email: string;
  private isVerified: boolean;
  private verificationToken: string | null;

  getEmail(): string { return this.email; }
  setEmail(email: string): void { this.email = email; }
  getIsVerified(): boolean { return this.isVerified; }
  setIsVerified(v: boolean): void { this.isVerified = v; }
  getVerificationToken(): string | null { return this.verificationToken; }
  setVerificationToken(t: string | null): void { this.verificationToken = t; }
}

// Logic lives in services
class UserService {
  verifyEmail(user: User, token: string): void {
    if (user.getVerificationToken() !== token) {
      throw new Error('Invalid token');
    }
    user.setIsVerified(true);
    user.setVerificationToken(null);
  }
}

// ✅ Rich domain model: behavior with data
class User {
  private email: string;
  private isVerified: boolean = false;
  private verificationToken: string | null = null;

  verifyEmail(token: string): void {
    if (this.verificationToken !== token) {
      throw new InvalidTokenError();
    }
    this.isVerified = true;
    this.verificationToken = null;
  }

  requestVerification(): string {
    if (this.isVerified) {
      throw new AlreadyVerifiedError();
    }
    this.verificationToken = generateToken();
    return this.verificationToken;
  }

  // Query method for UI is fine
  canAccessPremiumContent(): boolean {
    return this.isVerified;
  }
}
```

## React / React Native

```typescript
// ❌ Component asks for data, makes decisions
function CartButton({ cart }: { cart: Cart }) {
  const isEmpty = cart.getItems().length === 0;
  const total = cart.getItems().reduce((sum, i) => sum + i.price * i.qty, 0);
  const hasMinimum = total >= 50;

  return (
    <Button
      disabled={isEmpty || !hasMinimum}
      title={isEmpty ? 'Cart Empty' : `Checkout $${total}`}
    />
  );
}

// ✅ Domain object provides computed state
class Cart {
  canCheckout(): boolean {
    return !this.isEmpty() && this.meetsMinimumOrder();
  }

  getCheckoutButtonLabel(): string {
    if (this.isEmpty()) return 'Cart Empty';
    return `Checkout ${this.getFormattedTotal()}`;
  }

  private isEmpty(): boolean { /* ... */ }
  private meetsMinimumOrder(): boolean { /* ... */ }
}

// Component just tells what to render
function CartButton({ cart }: { cart: Cart }) {
  return (
    <Button
      disabled={!cart.canCheckout()}
      title={cart.getCheckoutButtonLabel()}
    />
  );
}
```

## When NOT to Apply

- **DTOs and ViewModels** — Pure data objects meant for transfer/display
- **Reporting/Analytics** — Sometimes you need to aggregate data from multiple objects
- **Serialization** — JSON export needs to access internal data
- **Testing assertions** — Tests legitimately query state to verify behavior

The key question: "Is this logic about this object's responsibility?"
