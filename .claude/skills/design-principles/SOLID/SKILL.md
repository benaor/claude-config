---
name: solid-principles
description: SOLID principles for TypeScript code review and refactoring. Use when reviewing code architecture, detecting design violations, refactoring classes/modules/hooks, or improving code maintainability. Covers SRP, OCP, LSP, ISP, DIP with concrete violation detection and fix patterns.
---

# SOLID Principles

Guidelines for reviewing and refactoring TypeScript code.

## Single Responsibility Principle (SRP)

A module/class/function should have only one reason to change.

### Violation Signs
- Class or module handling multiple domains (e.g., user data + API calls + caching)
- Function with multiple `// Step X` comments doing unrelated things
- Hook managing unrelated state pieces
- File imports from many unrelated domains

### When NOT to Apply
Avoid over-splitting trivial logic. A simple utility function doing 2-3 closely related operations is fine.

→ See [srp-example.md](references/srp-example.md) for before/after code.

---

## Open/Closed Principle (OCP)

Modules should be open for extension, closed for modification.

### Violation Signs
- Switch/if-else chains on type discriminators that grow with new features
- Modifying existing functions when adding new variants
- `type === 'X'` checks scattered across codebase
- Adding new feature requires touching many existing files

### When NOT to Apply
Don't create abstraction layers for 2-3 stable variants that rarely change. OCP adds indirection — apply when you genuinely expect extension.

→ See [ocp-example.md](references/ocp-example.md) for before/after code.

---

## Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without altering correctness.

### Violation Signs
- Subclass throws exceptions the parent doesn't declare
- Overridden method ignores or reinterprets parent's parameters
- Consumer code checks `instanceof` before calling methods
- Subclass with empty/no-op method implementations

### When NOT to Apply
LSP applies to inheritance hierarchies. For simple data objects or composition-based designs, focus on interface contracts instead.

→ See [lsp-example.md](references/lsp-example.md) for before/after code.

---

## Interface Segregation Principle (ISP)

Clients should not depend on interfaces they don't use.

### Violation Signs
- Implementing class has methods throwing `NotImplemented`
- Interface with 10+ methods where most implementers use only a few
- Props interface with many optional fields, most unused per component
- Mocking pain: test needs to mock methods unrelated to test case

### When NOT to Apply
Don't split interfaces that are naturally cohesive and always used together. 2-3 tightly coupled methods belong in one interface.

→ See [isp-example.md](references/isp-example.md) for before/after code.

---

## Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions.

### Violation Signs
- Direct instantiation of dependencies (`new ConcreteService()`)
- Importing concrete implementations in domain/usecase layer
- Hard to test without real database/API/filesystem
- Framework imports in business logic

### When NOT to Apply
Don't abstract stable, unlikely-to-change utilities (e.g., date formatting). DIP adds indirection — apply for infrastructure boundaries (DB, API, storage, analytics).

→ See [dip-example.md](references/dip-example.md) for before/after code.

---

## Review Checklist

| Principle | Quick Check |
|-----------|-------------|
| **SRP** | Does this module have multiple reasons to change? |
| **OCP** | Will adding a variant require modifying existing code? |
| **LSP** | Can all subtypes replace the base without breaking behavior? |
| **ISP** | Are there unused methods or `NotImplemented` stubs? |
| **DIP** | Does business logic import concrete infrastructure? |

### Refactoring Priority

1. **DIP violations** — hardest to test, fix first
2. **SRP violations** — cascading impact, fix early
3. **ISP violations** — testing pain, medium priority
4. **OCP violations** — refactor when adding new variants
5. **LSP violations** — fix when inheritance hierarchy grows
