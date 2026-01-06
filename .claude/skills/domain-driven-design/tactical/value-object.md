---
type: skill
name: value-object
category: ddd/tactical
---

# Value Object

## Detect

Scan for these patterns indicating missing or broken Value Object:

| Pattern | Example |
|---------|---------|
| Primitive for domain concept | `email: string`, `price: number`, `currency: string` |
| Repeated validation | Same regex/check in multiple files |
| Public mutable property | `public amount: number` |
| Method mutates this | `this.amount += x` with `void` return |
| Missing equals() | Class with value semantics, no equality method |
| External validation | `EmailValidator.validate(str)` separate from class |
| No behavior | Class with only constructor and getters |

## Fix

Valid Value Object structure:

```typescript
class Email {
  private constructor(private readonly value: string) {}

  static create(input: string): Email {
    const normalized = input.trim().toLowerCase();
    if (!Email.isValid(normalized)) {
      throw new InvalidEmailError(input);
    }
    return new Email(normalized);
  }

  private static isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }

  get domain(): string {
    return this.value.split('@')[1];
  }
}
```

Checklist:
- ✅ Private constructor
- ✅ Static factory with validation
- ✅ All properties `readonly`
- ✅ `equals()` based on values
- ✅ Behavior encapsulated

## Violations

### Primitive Obsession
```typescript
// ❌ Detect: primitive type + validation logic nearby
function createUser(email: string) {
  if (!email.includes('@')) throw new Error('Invalid');
}

// ✅ Fix: extract Value Object
function createUser(email: Email) {}
```

### Mutation
```typescript
// ❌ Detect: void return + this mutation
add(other: Money): void {
  this.amount += other.amount;
}

// ✅ Fix: return new instance
add(other: Money): Money {
  return new Money(this.amount + other.amount, this.currency);
}
```

### External validation
```typescript
// ❌ Detect: public constructor + separate validator
class Email {
  constructor(public readonly value: string) {}
}

// ✅ Fix: private constructor + factory
class Email {
  private constructor(private readonly value: string) {}
  static create(input: string): Email { /* validates here */ }
}
```

### Reference equality
```typescript
// ❌ Detect: no equals() method on value class
// ✅ Fix: implement equals() comparing all properties
```

### Anemic Value Object
```typescript
// ❌ Detect: logic outside, class is just data
function isInRange(date: Date, range: DateRange): boolean

// ✅ Fix: move logic inside
class DateRange {
  contains(date: Date): boolean
  overlaps(other: DateRange): boolean
}
```

## Skip when

- DTOs — transfer objects between layers
- Simple boolean — `isActive: boolean` doesn't need VO
- Infrastructure code — not domain layer
