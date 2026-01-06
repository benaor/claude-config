---
name: kiss
description: KISS (Keep It Simple, Stupid) principle for TypeScript code review and refactoring. Use when detecting over-complicated solutions, unnecessary abstractions, convoluted logic, or code that's hard to understand. Helps simplify implementations.
---

# KISS — Keep It Simple, Stupid

Prefer simple solutions over clever ones. Code is read far more often than written.

## Violation Signs

- Nested ternaries or complex one-liners
- Multiple levels of indirection to perform a simple task
- "Clever" solutions that require comments to explain
- Generic abstractions where a concrete solution would suffice
- Callback chains or deeply nested promises

## Example

**Before — KISS Violation:**

```typescript
// utils.ts — "clever" but unreadable
const processUserData = <T extends Record<string, unknown>>(
  data: T,
  transforms: Array<(input: Partial<T>) => Partial<T>>,
  validators: Array<(input: Partial<T>) => boolean>,
  options?: { stopOnFirstError?: boolean; mergeStrategy?: 'shallow' | 'deep' }
) =>
  transforms.reduce(
    (acc, transform) =>
      validators.every((v) => v(acc)) || options?.stopOnFirstError
        ? options?.mergeStrategy === 'deep'
          ? deepMerge(acc, transform(acc))
          : { ...acc, ...transform(acc) }
        : acc,
    data as Partial<T>
  );

// Usage requires mental gymnastics
const result = processUserData(
  user,
  [(u) => ({ ...u, name: u.name?.trim() }), (u) => ({ ...u, age: Number(u.age) })],
  [(u) => !!u.name, (u) => !isNaN(u.age as number)],
  { stopOnFirstError: true, mergeStrategy: 'shallow' }
);
```

**After — KISS Applied:**

```typescript
// utils.ts — readable and debuggable
interface UserInput {
  name?: string;
  age?: string | number;
}

interface ValidatedUser {
  name: string;
  age: number;
}

function validateAndTransformUser(input: UserInput): ValidatedUser | null {
  const name = input.name?.trim();
  if (!name) {
    return null;
  }

  const age = Number(input.age);
  if (isNaN(age)) {
    return null;
  }

  return { name, age };
}

// Usage is obvious
const result = validateAndTransformUser(user);
if (!result) {
  handleValidationError();
}
```

## Anti-Pattern: Abstraction Astronaut

Creating layers of abstraction that obscure simple operations:

```typescript
// ❌ Over-abstracted
interface StringProcessor {
  process(input: string): string;
}

class TrimProcessor implements StringProcessor {
  process(input: string): string {
    return input.trim();
  }
}

class LowercaseProcessor implements StringProcessor {
  process(input: string): string {
    return input.toLowerCase();
  }
}

class ProcessorChain {
  constructor(private processors: StringProcessor[]) {}
  
  execute(input: string): string {
    return this.processors.reduce((acc, p) => p.process(acc), input);
  }
}

const result = new ProcessorChain([
  new TrimProcessor(),
  new LowercaseProcessor()
]).execute(email);

// ✅ Just write the code
const normalizedEmail = email.trim().toLowerCase();
```

## When NOT to Apply

Simplicity doesn't mean dumbing down. Acceptable complexity includes:

- **Domain complexity** — If the business rules are complex, the code will reflect that
- **Performance-critical paths** — Sometimes clever solutions are necessary for performance
- **Type safety** — Complex generic types can prevent runtime errors
- **Well-known patterns** — Design patterns may add indirection but improve maintainability

The key question: "Can a new team member understand this in under 2 minutes?"
