---
name: least-astonishment
description: Principle of Least Astonishment (POLA) for TypeScript code review and refactoring. Use when detecting surprising behavior, misleading names, unexpected side effects, or inconsistent APIs. Helps improve code predictability and developer experience.
---

# Principle of Least Astonishment

Code should behave as users and developers expect. Minimize surprise. When in doubt, do the obvious thing.

## Also Known As

- Principle of Least Surprise
- POLA (different from Principle of Least Authority)

## Violation Signs

- Function names that don't match behavior
- Methods with unexpected side effects
- Inconsistent return types across similar functions
- Parameters that change meaning based on context
- Default values that produce surprising results

## Example

**Before — Astonishing Behavior:**

```typescript
// UserService.ts — full of surprises
class UserService {
  // Surprise: "get" also updates last access time
  async getUser(id: string): Promise<User> {
    const user = await this.db.findById(id);
    user.lastAccessedAt = new Date();      // Side effect!
    await this.db.save(user);              // Write in a "get"!
    return user;
  }

  // Surprise: returns different types
  async findUsers(query: string): Promise<User[] | User | null> {
    const users = await this.db.search(query);
    if (users.length === 0) return null;     // null for empty
    if (users.length === 1) return users[0]; // single User
    return users;                             // array of Users
  }

  // Surprise: mutates the input
  formatUser(user: User): string {
    user.name = user.name.trim();           // Mutates input!
    user.email = user.email.toLowerCase();  // Mutates input!
    return `${user.name} <${user.email}>`;
  }

  // Surprise: "delete" doesn't delete
  async deleteUser(id: string): Promise<void> {
    const user = await this.db.findById(id);
    user.status = 'deleted';                 // Soft delete
    await this.db.save(user);                // Still exists!
  }
}
```

**After — Predictable Behavior:**

```typescript
// UserService.ts — does what it says
class UserService {
  // Query is pure — no side effects
  async getUser(id: string): Promise<User> {
    return this.db.findById(id);
  }

  // Separate method for tracking access
  async recordUserAccess(id: string): Promise<void> {
    await this.db.updateLastAccess(id, new Date());
  }

  // Consistent return type
  async findUsers(query: string): Promise<User[]> {
    return this.db.search(query);
    // Empty array for no results — not null
    // Always array — not sometimes single
  }

  // Pure function, no mutation
  formatUser(user: User): string {
    const name = user.name.trim();
    const email = user.email.toLowerCase();
    return `${name} <${email}>`;
  }

  // Name reflects actual behavior
  async softDeleteUser(id: string): Promise<void> {
    const user = await this.db.findById(id);
    user.status = 'deleted';
    await this.db.save(user);
  }

  // Or provide actual delete
  async permanentlyDeleteUser(id: string): Promise<void> {
    await this.db.delete(id);
  }
}
```

## Naming Conventions That Prevent Surprise

```typescript
// Prefixes that set expectations:

// === QUERIES (return value, no side effects) ===

// get* — synchronous, returns value
getFullName(): string

// fetch* / load* — async, network/IO read
fetchUser(id: string): Promise<User>
loadSettings(): Promise<Settings>

// find* — may return null/undefined
findUserByEmail(email: string): Promise<User | null>

// *OrThrow — throws on failure instead of returning null
getOrderOrThrow(id: string): Promise<Order>

// *OrDefault — returns default instead of null
getThemeOrDefault(): Theme

// build* / make* — pure factory, creates in-memory instance
buildOrder(items: Item[]): Order
makeViewModel(data: Data): ViewModel

// parse* / tryParse* — transforms input, no side effects
parseDate(value: string): Date
tryParseInt(value: string): number | null

// === COMMANDS (modify state, return void) ===

// save* / persist* — stores to database
saveOrder(order: Order): Promise<void>

// update* / set* — modifies state
updateEmail(newEmail: string): Promise<void>

// delete* / remove* — permanently removes
deleteAccount(): Promise<void>

// soft* prefix for non-destructive variants
softDeleteUser(id: string): Promise<void>

// send* / notify* — triggers external action
sendEmail(to: string, content: EmailContent): Promise<void>
notifyUser(userId: string, message: string): Promise<void>
```

## Anti-Pattern: Boolean Trap

```typescript
// ❌ What does `true` mean?
processOrder(order, true);
formatDate(date, false, true);
createUser(data, true, false, true);

// ✅ Named parameters or enums
processOrder(order, { immediate: true });

formatDate(date, {
  includeTime: false,
  useRelative: true,
});

createUser(data, {
  sendWelcomeEmail: true,
  requireVerification: false,
  isAdmin: true,
});

// Or use distinct functions
processOrderImmediately(order);
processOrderQueued(order);
```

## React / React Native

```typescript
// ❌ Component with surprising behavior
interface ButtonProps {
  onClick: () => void;
  children: React.ReactNode;
  disabled?: boolean;
}

function Button({ onClick, children, disabled }: ButtonProps) {
  return (
    <Pressable
      onPress={() => {
        onClick();                        // Surprise: fires even when disabled!
        analytics.track('button_click');  // Surprise: hidden analytics!
      }}
      style={disabled ? styles.disabled : styles.normal}
    >
      <Text>{children}</Text>
    </Pressable>
  );
}

// ✅ Predictable component
interface ButtonProps {
  onPress: () => void;
  children: React.ReactNode;
  disabled?: boolean;
  trackingId?: string; // Explicit if tracking needed
}

function Button({ onPress, children, disabled, trackingId }: ButtonProps) {
  const handlePress = () => {
    if (disabled) return; // Expected: disabled means no action
    
    onPress();
    
    if (trackingId) {
      analytics.track('button_press', { id: trackingId });
    }
  };

  return (
    <Pressable
      onPress={handlePress}
      disabled={disabled}
      style={disabled ? styles.disabled : styles.normal}
    >
      <Text>{children}</Text>
    </Pressable>
  );
}
```

## Consistent Error Handling

```typescript
// ❌ Inconsistent error patterns
class Api {
  getUser(id: string): Promise<User>        // throws on error
  findUsers(q: string): Promise<User[]>     // returns [] on error
  deleteUser(id: string): Promise<boolean>  // returns false on error
  updateUser(u: User): Promise<User | null> // returns null on error
}

// ✅ Consistent pattern across all methods
class Api {
  // All methods throw on error
  getUser(id: string): Promise<User>
  findUsers(query: string): Promise<User[]>
  deleteUser(id: string): Promise<void>
  updateUser(user: User): Promise<User>
}

// Or all return Result types
class Api {
  getUser(id: string): Promise<Result<User, ApiError>>
  findUsers(query: string): Promise<Result<User[], ApiError>>
  deleteUser(id: string): Promise<Result<void, ApiError>>
  updateUser(user: User): Promise<Result<User, ApiError>>
}
```

## When Surprise is Acceptable

- **Performance optimizations** — Caching may be surprising but valuable (document it)
- **Framework conventions** — Follow established patterns even if initially surprising
- **Security measures** — Rate limiting, sanitization may surprise but protect

The key question: "Would a developer using this code be surprised by its behavior?"
