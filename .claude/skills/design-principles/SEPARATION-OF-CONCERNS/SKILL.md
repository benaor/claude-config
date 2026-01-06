---
name: separation-of-concerns
description: Separation of Concerns principle for TypeScript code review and refactoring. Use when detecting mixed responsibilities, business logic in UI components, infrastructure code in domain layer, or tightly coupled modules. Helps improve modularity and maintainability. Related to Clean Architecture layers.
---

# Separation of Concerns

Divide a system into distinct sections, each addressing a separate concern. A concern is a set of information that affects the code.

## Common Concerns to Separate

- **UI/Presentation**: How things look
- **Business logic/Domain**: What things do
- **Data access**: Where things are stored
- **Navigation**: How to move between screens
- **State management**: How data flows
- **External services**: Third-party integrations

## Violation Signs

- Components fetching data AND rendering AND handling business rules
- Business logic importing React or framework-specific code
- Database queries mixed with validation logic
- UI components aware of API endpoint URLs
- Navigation logic inside business rules

→ See [layers-example.md](references/layers-example.md) for complete before/after refactoring.

## Layer Dependencies

```
┌─────────────────────────────────────────┐
│           Presentation Layer            │
│  (Screens, Components, Hooks, ViewModels)│
└─────────────────────┬───────────────────┘
                      │ depends on
                      ▼
┌─────────────────────────────────────────┐
│             Domain Layer                │
│    (Entities, UseCases, Ports)          │
└─────────────────────┬───────────────────┘
                      │ depends on
                      ▼
┌─────────────────────────────────────────┐
│          Infrastructure Layer           │
│   (Adapters, Repositories, Services)    │
└─────────────────────────────────────────┘

Domain has NO dependencies on Presentation or Infrastructure
```

## Quick Checks by Layer

| Layer | Should NOT contain |
|-------|-------------------|
| **Domain** | React imports, HTTP clients, AsyncStorage, navigation |
| **Presentation** | Direct API calls, SQL queries, business rules |
| **Infrastructure** | UI components, business decisions, routing |

## When Concerns Can Stay Together

- **Prototypes**: Speed over architecture for throwaway code
- **Simple utilities**: Pure functions without dependencies
- **Trivial components**: A button that just styles and calls onPress
- **Co-location benefit**: Sometimes keeping related code together aids understanding

The key question: "If I change this concern, how many unrelated things break?"
