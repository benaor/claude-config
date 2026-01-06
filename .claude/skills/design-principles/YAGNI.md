---
name: yagni
description: YAGNI (You Ain't Gonna Need It) principle for TypeScript code review and refactoring. Use when detecting premature abstractions, unused flexibility, over-engineered solutions, or speculative features. Helps identify code that should be simplified or removed.
---

# YAGNI — You Ain't Gonna Need It

Do not implement functionality until it is actually needed. Every line of code has a maintenance cost.

## Violation Signs

- Generic solutions for a single concrete use case
- Configuration options that are never used
- Abstract base classes with only one implementation
- Comments like `// for future use` or `// in case we need`
- Parameters or props that are always passed the same value

## Example

**Before — YAGNI Violation:**

```typescript
// UserRepository.ts — over-engineered for "future" needs
interface QueryOptions {
  fields?: string[];
  include?: string[];
  cache?: boolean;
  cacheTTL?: number;
  retryCount?: number;
  timeout?: number;
  transform?: (user: User) => User;
}

class UserRepository {
  constructor(
    private readonly httpClient: HttpClient,
    private readonly cache: CachePort,
    private readonly logger: LoggerPort,
    private readonly metrics: MetricsPort // "we might need metrics later"
  ) {}

  async getById(id: string, options: QueryOptions = {}): Promise<User> {
    const {
      fields = ['*'],
      include = [],
      cache = true,
      cacheTTL = 3600,
      retryCount = 3,
      timeout = 5000,
      transform = (u) => u,
    } = options;

    // Complex caching logic for hypothetical needs
    if (cache) {
      const cacheKey = this.buildCacheKey(id, fields, include);
      const cached = await this.cache.get<User>(cacheKey);
      if (cached) {
        this.metrics.increment('cache_hit');
        return transform(cached);
      }
      this.metrics.increment('cache_miss');
    }

    // Retry logic "in case API is flaky"
    let lastError: Error | null = null;
    for (let attempt = 0; attempt < retryCount; attempt++) {
      try {
        const user = await this.httpClient.get<User>(`/users/${id}`, {
          timeout,
          params: { fields: fields.join(','), include: include.join(',') }
        });
        // ... more unused flexibility
      } catch (error) {
        lastError = error as Error;
        this.logger.warn(`Attempt ${attempt + 1} failed`);
      }
    }
    throw lastError;
  }

  private buildCacheKey(id: string, fields: string[], include: string[]): string {
    return `user:${id}:${fields.join(',')}:${include.join(',')}`;
  }
}

// Actual usage everywhere in the codebase:
const user = await userRepository.getById(userId); // No options ever passed
```

**After — YAGNI Applied:**

```typescript
// UserRepository.ts — implements only what's actually used
class UserRepository {
  constructor(private readonly httpClient: HttpClient) {}

  async getById(id: string): Promise<User> {
    return this.httpClient.get<User>(`/users/${id}`);
  }
}

// When caching is ACTUALLY needed, add it then:
// - You'll know the real requirements
// - You'll know the actual cache key structure needed
// - You won't guess wrong about TTL, retry logic, etc.
```

## Anti-Pattern: Speculative Generality

Building abstractions for hypothetical future requirements:

```typescript
// ❌ Abstract factory for one implementation
interface NotificationFactory {
  createNotification(type: string): Notification;
}

class NotificationFactoryImpl implements NotificationFactory {
  createNotification(type: string): Notification {
    if (type === 'push') return new PushNotification();
    throw new Error('Unknown type');
  }
}

// ✅ Just use what you need
class PushNotification {
  send(message: string): Promise<void> { /* ... */ }
}
```

## When NOT to Apply

YAGNI doesn't mean ignoring good architecture. Apply thoughtful design for:

- **Known requirements** — If the spec says "support 3 payment methods", design for it
- **Proven patterns** — Dependency injection isn't speculative, it's testability
- **Security/compliance** — Don't defer authentication because "we only have 2 users now"
- **Breaking changes** — Public APIs need more upfront design to avoid breaking consumers

The key question: "Do I have evidence this will be needed, or am I guessing?"
