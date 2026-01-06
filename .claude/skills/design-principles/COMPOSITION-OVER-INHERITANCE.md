---
name: composition-over-inheritance
description: Composition over Inheritance principle for TypeScript code review and refactoring. Use when detecting deep inheritance hierarchies, fragile base classes, or rigid class structures. Helps refactor toward flexible, composable designs using delegation and interfaces.
---

# Composition over Inheritance

Favor object composition over class inheritance. Build complex behavior by combining simple, focused pieces rather than extending base classes.

## Why Composition

- **Flexibility**: Change behavior at runtime by swapping components
- **Testability**: Inject mocks easily
- **Avoiding fragile base class**: Changes to parent don't break children
- **Multiple behaviors**: Combine capabilities without multiple inheritance

## Violation Signs

- Deep inheritance hierarchies (3+ levels)
- Base classes with mixed responsibilities
- Overriding methods just to disable them
- "Is-a" relationships that are really "has-a"
- Abstract classes with concrete methods that children must work around

## Example

**Before — Inheritance Hierarchy:**

```typescript
// Deep hierarchy with fragile base class
abstract class BaseRepository {
  protected db: Database;
  protected cache: Cache;
  protected logger: Logger;

  constructor(db: Database, cache: Cache, logger: Logger) {
    this.db = db;
    this.cache = cache;
    this.logger = logger;
  }

  protected async withCache<T>(key: string, fn: () => Promise<T>): Promise<T> {
    const cached = await this.cache.get<T>(key);
    if (cached) return cached;
    const result = await fn();
    await this.cache.set(key, result);
    return result;
  }

  protected log(message: string): void {
    this.logger.info(`[${this.constructor.name}] ${message}`);
  }
}

abstract class BaseUserRepository extends BaseRepository {
  protected validateEmail(email: string): boolean {
    return email.includes('@');
  }
}

class CachedUserRepository extends BaseUserRepository {
  async getById(id: string): Promise<User> {
    return this.withCache(`user:${id}`, async () => {
      this.log(`Fetching user ${id}`);
      return this.db.findById('users', id);
    });
  }
}

// Problems:
// - Adding audit logging means changing BaseRepository (breaks all children)
// - What if some repo needs cache but not logger?
// - What if we need a UserRepository without cache?
// - Testing requires mocking entire inheritance chain
```

**After — Composition:**

```typescript
// Small, focused interfaces
interface Storage {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T): Promise<void>;
}

interface Logger {
  info(message: string): void;
}

interface UserDataSource {
  findById(id: string): Promise<User | null>;
}

// Behaviors as composable wrappers
class CachingUserDataSource implements UserDataSource {
  constructor(
    private readonly source: UserDataSource,
    private readonly cache: Storage,
    private readonly ttl: number = 3600
  ) {}

  async findById(id: string): Promise<User | null> {
    const cacheKey = `user:${id}`;
    const cached = await this.cache.get<User>(cacheKey);
    if (cached) return cached;

    const user = await this.source.findById(id);
    if (user) {
      await this.cache.set(cacheKey, user);
    }
    return user;
  }
}

class LoggingUserDataSource implements UserDataSource {
  constructor(
    private readonly source: UserDataSource,
    private readonly logger: Logger
  ) {}

  async findById(id: string): Promise<User | null> {
    this.logger.info(`Fetching user ${id}`);
    return this.source.findById(id);
  }
}

// Concrete implementation
class PostgresUserDataSource implements UserDataSource {
  constructor(private readonly db: Database) {}

  async findById(id: string): Promise<User | null> {
    return this.db.query('SELECT * FROM users WHERE id = ?', [id]);
  }
}

// Compose behaviors as needed
const simpleUserSource = new PostgresUserDataSource(db);

const cachedUserSource = new CachingUserDataSource(
  simpleUserSource,
  redisCache
);

const fullUserSource = new LoggingUserDataSource(
  new CachingUserDataSource(
    new PostgresUserDataSource(db),
    redisCache
  ),
  logger
);

// Testing is easy — just test each piece
const mockSource: UserDataSource = { findById: jest.fn() };
const cachedSource = new CachingUserDataSource(mockSource, mockCache);
```

## Delegation Pattern

```typescript
// ❌ Inheritance for code reuse
class TimestampedEntity {
  createdAt: Date = new Date();
  updatedAt: Date = new Date();

  touch(): void {
    this.updatedAt = new Date();
  }
}

class User extends TimestampedEntity {
  constructor(public name: string) {
    super();
  }
}

class Order extends TimestampedEntity {
  constructor(public items: Item[]) {
    super();
  }
}

// ✅ Composition with delegation
interface Timestamps {
  createdAt: Date;
  updatedAt: Date;
}

function createTimestamps(): Timestamps {
  const now = new Date();
  return { createdAt: now, updatedAt: now };
}

function touch(timestamps: Timestamps): Timestamps {
  return { ...timestamps, updatedAt: new Date() };
}

interface User {
  name: string;
  timestamps: Timestamps;
}

interface Order {
  items: Item[];
  timestamps: Timestamps;
}

const user: User = {
  name: 'Alice',
  timestamps: createTimestamps(),
};

const updatedUser = {
  ...user,
  timestamps: touch(user.timestamps),
};
```

## When Inheritance is Acceptable

- **True "is-a" relationships**: `Cat extends Animal` where polymorphism is needed
- **Framework requirements**: React class components (legacy), some ORMs
- **Template method pattern**: Well-defined extension points in abstract class
- **Single level**: One level of inheritance is often fine

```typescript
// Acceptable: one level, true is-a, clear extension point
abstract class PaymentProcessor {
  async process(payment: Payment): Promise<Result> {
    this.validate(payment);
    const result = await this.executePayment(payment);
    await this.notifyComplete(result);
    return result;
  }

  protected abstract executePayment(payment: Payment): Promise<Result>;
  
  protected validate(payment: Payment): void { /* common validation */ }
  protected notifyComplete(result: Result): Promise<void> { /* common notification */ }
}

class StripeProcessor extends PaymentProcessor {
  protected async executePayment(payment: Payment): Promise<Result> {
    // Stripe-specific implementation
  }
}
```

The key question: "Am I using inheritance for code reuse, or for genuine polymorphic behavior?"
