# ISP Example

## Before — ISP Violation

```typescript
// UserPort.ts — fat interface
interface UserPort {
  getById(id: string): Promise<User>;
  getAll(): Promise<User[]>;
  create(user: CreateUserDTO): Promise<User>;
  update(id: string, user: UpdateUserDTO): Promise<User>;
  delete(id: string): Promise<void>;
  getByEmail(email: string): Promise<User | null>;
  search(query: string): Promise<User[]>;
  exportToCsv(): Promise<string>;
  importFromCsv(csv: string): Promise<void>;
}

// ReadOnlyUserRepository.ts — forced to stub methods it doesn't need
class ReadOnlyUserRepository implements UserPort {
  async getById(id: string): Promise<User> { /* ... */ }
  async getAll(): Promise<User[]> { /* ... */ }
  async getByEmail(email: string): Promise<User | null> { /* ... */ }
  async search(query: string): Promise<User[]> { /* ... */ }

  // ISP violations — methods that don't belong
  async create(): Promise<User> { throw new Error('Read-only'); }
  async update(): Promise<User> { throw new Error('Read-only'); }
  async delete(): Promise<void> { throw new Error('Read-only'); }
  async exportToCsv(): Promise<string> { throw new Error('Not supported'); }
  async importFromCsv(): Promise<void> { throw new Error('Not supported'); }
}
```

## After — ISP Applied

```typescript
// UserReadPort.ts
interface UserReadPort {
  getById(id: string): Promise<User>;
  getByEmail(email: string): Promise<User | null>;
}

// UserWritePort.ts
interface UserWritePort {
  create(user: CreateUserDTO): Promise<User>;
  update(id: string, user: UpdateUserDTO): Promise<User>;
  delete(id: string): Promise<void>;
}

// UserSearchPort.ts
interface UserSearchPort {
  search(query: string): Promise<User[]>;
  getAll(): Promise<User[]>;
}

// UserExportPort.ts
interface UserExportPort {
  exportToCsv(): Promise<string>;
  importFromCsv(csv: string): Promise<void>;
}

// ReadOnlyUserRepository.ts — implements only what it needs
class ReadOnlyUserRepository implements UserReadPort, UserSearchPort {
  async getById(id: string): Promise<User> { /* ... */ }
  async getByEmail(email: string): Promise<User | null> { /* ... */ }
  async search(query: string): Promise<User[]> { /* ... */ }
  async getAll(): Promise<User[]> { /* ... */ }
}

// GetUserUseCase.ts — depends only on UserReadPort
class GetUserUseCase {
  constructor(private readonly userReader: UserReadPort) {}
  // Easy to test: only mock getById and getByEmail
}
```
