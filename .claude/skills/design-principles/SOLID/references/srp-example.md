# SRP Example

## Before — SRP Violation

```typescript
// UserService.ts — handles too many concerns
class UserService {
  private cache: Map<string, User> = new Map();

  async getUser(id: string): Promise<User> {
    // Caching logic
    if (this.cache.has(id)) {
      return this.cache.get(id)!;
    }

    // HTTP logic
    const response = await fetch(`/api/users/${id}`, {
      headers: { 'Authorization': `Bearer ${this.getToken()}` }
    });
    const user = await response.json();

    // Validation logic
    if (!user.email || !user.name) {
      throw new Error('Invalid user data');
    }

    // Analytics logic
    this.trackUserFetch(user.id);

    this.cache.set(id, user);
    return user;
  }

  private getToken(): string { /* ... */ }
  private trackUserFetch(id: string): void { /* ... */ }
}
```

## After — SRP Applied

```typescript
// UserRepository.ts — only data access
class UserRepository {
  constructor(
    private readonly httpClient: HttpClient,
    private readonly cache: CachePort
  ) {}

  async getById(id: string): Promise<User> {
    const cached = await this.cache.get<User>(`user:${id}`);
    if (cached) return cached;

    const user = await this.httpClient.get<User>(`/users/${id}`);
    await this.cache.set(`user:${id}`, user);
    return user;
  }
}

// UserValidator.ts — only validation
class UserValidator {
  validate(user: User): ValidationResult {
    const errors: string[] = [];
    if (!user.email) errors.push('Email required');
    if (!user.name) errors.push('Name required');
    return { isValid: errors.length === 0, errors };
  }
}

// GetUserUseCase.ts — orchestration only
class GetUserUseCase {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly validator: UserValidator,
    private readonly analytics: AnalyticsPort
  ) {}

  async execute(id: string): Promise<User> {
    const user = await this.userRepository.getById(id);
    const validation = this.validator.validate(user);
    if (!validation.isValid) {
      throw new ValidationError(validation.errors);
    }
    this.analytics.track('user_fetched', { userId: id });
    return user;
  }
}
```
