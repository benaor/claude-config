---
name: cqs
description: CQS (Command Query Separation) principle for TypeScript code review and refactoring. Use when detecting methods that both modify state and return values, side effects in getters, or mixed responsibilities in functions. Helps improve predictability and testability. See also CQRS for architectural application.
---

# CQS — Command Query Separation

A method should either be a **command** (changes state, returns nothing) or a **query** (returns data, changes nothing). Never both.

## Definitions

- **Command**: Performs an action, has side effects, returns `void`
- **Query**: Returns data, has no side effects, idempotent

## Violation Signs

- Methods that return a value AND modify state
- Getters with side effects (logging, caching that affects behavior)
- Functions named `getX` that also `setY`
- Difficulty testing because calling a method twice gives different results

## Example

**Before — CQS Violation:**

```typescript
// CartService.ts — mixed command/query
class CartService {
  private cart: Cart;

  // Violation: command + query combined
  addItemAndGetTotal(item: CartItem): number {
    this.cart.items.push(item);  // Command: modifies state
    return this.calculateTotal(); // Query: returns value
  }

  // Violation: query with side effect
  getNextOrderNumber(): string {
    this.lastOrderNumber++;       // Side effect!
    return `ORD-${this.lastOrderNumber}`;
  }

  // Violation: getter that modifies
  getOrCreateCart(userId: string): Cart {
    if (!this.carts.has(userId)) {
      this.carts.set(userId, new Cart()); // Side effect!
    }
    return this.carts.get(userId)!;
  }
}

// Usage is unpredictable
const total = cartService.addItemAndGetTotal(item);
// Did I just want the total? Now I've modified the cart!

const orderNum1 = cartService.getNextOrderNumber();
const orderNum2 = cartService.getNextOrderNumber();
// orderNum1 !== orderNum2 — "get" implies no change!
```

**After — CQS Applied:**

```typescript
// CartService.ts — separated commands and queries
class CartService {
  private cart: Cart;

  // Command: modifies state, returns nothing
  addItem(item: CartItem): void {
    this.cart.items.push(item);
  }

  // Command: modifies state, returns nothing
  removeItem(itemId: string): void {
    this.cart.items = this.cart.items.filter(i => i.id !== itemId);
  }

  // Query: returns value, no side effects
  getTotal(): number {
    return this.cart.items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    );
  }

  // Query: pure, idempotent
  getItemCount(): number {
    return this.cart.items.length;
  }

  // Query: returns data, doesn't modify
  getCart(): Cart {
    return this.cart;
  }
}

// OrderNumberService.ts — if you need unique IDs
class OrderNumberService {
  // Command: allocates and persists
  allocateOrderNumber(): void {
    this.nextNumber++;
    this.persistence.save(this.nextNumber);
  }

  // Query: returns current (already allocated)
  getCurrentOrderNumber(): string {
    return `ORD-${this.nextNumber}`;
  }
}

// Clear, predictable usage
cartService.addItem(item);           // I know this modifies
const total = cartService.getTotal(); // I know this is safe
```

## Anti-Pattern: Pop Operations

Classic example from stack/queue operations:

```typescript
// ❌ Stack.pop() violates CQS
class Stack<T> {
  pop(): T {                    // Returns value AND modifies
    return this.items.splice(-1)[0];
  }
}

// ✅ CQS-compliant stack
class Stack<T> {
  // Query: look at top
  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  // Command: remove top
  remove(): void {
    this.items.pop();
  }

  // Usage
  const top = stack.peek();
  stack.remove();
}

// Or accept the pragmatic violation for well-known operations
// but document it clearly
```

## When CQS Can Be Relaxed

- **Well-known idioms**: `array.pop()`, `map.set()` returning old value
- **Fluent builders**: `builder.setX(v)` returning `this` for chaining
- **Atomic operations**: `counter.incrementAndGet()` where separation would cause race conditions
- **Performance-critical**: When separation would require two round-trips

Document these exceptions clearly. The key question: "Will callers be surprised by the side effect?"
