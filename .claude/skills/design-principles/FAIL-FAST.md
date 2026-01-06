---
name: fail-fast
description: Fail Fast principle for TypeScript code review and refactoring. Use when detecting delayed error handling, silent failures, defensive programming that hides bugs, or late validation. Helps improve error detection and debugging.
---

# Fail Fast

Detect and report errors as early as possible. The closer a failure is to its cause, the easier it is to debug.

## Violation Signs

- `catch` blocks that swallow errors silently
- Default values that hide invalid input
- Validation at the end of a process instead of the beginning
- Defensive null checks deep in call chains
- Error messages that don't identify the root cause

## Example

**Before — Fail Slow (Hidden Errors):**

```typescript
// UserService.ts — errors are hidden or delayed
class UserService {
  async createUser(input: unknown): Promise<User | null> {
    try {
      // No input validation — proceeds with bad data
      const data = input as CreateUserDTO;
      
      // Defaults hide missing data
      const email = data.email || 'unknown@example.com';
      const name = data.name || 'Anonymous';
      const age = data.age ?? 0;

      // Invalid data reaches the database
      const user = await this.userRepository.create({
        email,
        name,
        age,
        createdAt: new Date(),
      });

      // Error happens later, far from cause
      await this.emailService.sendWelcome(user.email);
      // "Failed to send email to unknown@example.com"
      // — WHERE did this bad email come from?

      return user;
    } catch (error) {
      // Swallowed! Caller has no idea what went wrong
      console.log('Something went wrong');
      return null;
    }
  }
}

// Consumer can't distinguish success from failure
const user = await userService.createUser(badInput);
if (!user) {
  // Was it validation? Database? Email? Network?
  // No way to know, can't give user meaningful feedback
}
```

**After — Fail Fast (Early Detection):**

```typescript
// UserService.ts — validate early, fail loudly
class UserService {
  async createUser(input: unknown): Promise<User> {
    // Fail fast: validate at entry point
    const validatedInput = this.validateInput(input);

    // Each step can assume valid data
    const user = await this.userRepository.create({
      email: validatedInput.email,
      name: validatedInput.name,
      age: validatedInput.age,
      createdAt: new Date(),
    });

    await this.emailService.sendWelcome(user.email);

    return user;
  }

  private validateInput(input: unknown): CreateUserDTO {
    if (!input || typeof input !== 'object') {
      throw new ValidationError('Input must be an object');
    }

    const data = input as Record<string, unknown>;

    if (!data.email || typeof data.email !== 'string') {
      throw new ValidationError('Email is required and must be a string');
    }

    if (!this.isValidEmail(data.email)) {
      throw new ValidationError(`Invalid email format: ${data.email}`);
    }

    if (!data.name || typeof data.name !== 'string') {
      throw new ValidationError('Name is required and must be a string');
    }

    if (data.age !== undefined && (typeof data.age !== 'number' || data.age < 0)) {
      throw new ValidationError('Age must be a positive number');
    }

    return {
      email: data.email,
      name: data.name,
      age: data.age as number | undefined,
    };
  }
}

// Consumer gets actionable errors
try {
  const user = await userService.createUser(input);
} catch (error) {
  if (error instanceof ValidationError) {
    // "Invalid email format: not-an-email"
    // Clear, actionable, points to the cause
    showFieldError(error.message);
  }
  throw error;
}
```

## Anti-Pattern: Defensive Null Absorption

```typescript
// ❌ Silently absorbs errors, bug hides for months
function processOrder(order: Order | null | undefined) {
  const items = order?.items ?? [];
  const total = items.reduce((sum, i) => sum + (i?.price ?? 0) * (i?.qty ?? 0), 0);
  const address = order?.shipping?.address ?? 'No address';
  
  // Processes empty order with 0 total and 'No address'
  // No one notices until customer complains
  return { total, address };
}

// ✅ Fail immediately on invalid state
function processOrder(order: Order): OrderResult {
  if (!order.items.length) {
    throw new InvalidOrderError('Order has no items');
  }

  if (!order.shipping?.address) {
    throw new InvalidOrderError('Shipping address is required');
  }

  const total = order.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );

  return { total, address: order.shipping.address };
}
```

## Constructor Validation

```typescript
// ❌ Invalid object can exist
class Product {
  constructor(
    public id: string,
    public name: string,
    public price: number
  ) {}
}

const product = new Product('', '', -50); // Invalid but created

// ✅ Fail fast in constructor
class Product {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly price: number
  ) {
    if (!id) throw new Error('Product ID is required');
    if (!name) throw new Error('Product name is required');
    if (price < 0) throw new Error('Price cannot be negative');
  }
}

const product = new Product('', '', -50); // Throws immediately
```

## When NOT to Fail Fast

- **Batch operations** — Collect all errors, report at end
- **User input** — Show all validation errors, not just the first
- **Graceful degradation** — Feature flags, fallbacks for non-critical features
- **Retry-able operations** — Network requests may deserve retries

The key question: "If this fails silently, how long until someone notices?"
