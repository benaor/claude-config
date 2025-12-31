# Code Review Checklist

## 🏗️ Architecture & Structure

- [ ] Core NEVER imports from Infrastructure or UI
- [ ] File is in the correct folder with the correct extension
- [ ] Bounded context is respected (no direct imports between modules)
- [ ] Imports between modules go through `shared/` if needed

## 🎯 Core Domain

- [ ] Use cases have a single responsibility
- [ ] Use cases return `Result<T, E>`
- [ ] Entities are pure types/interfaces (no React logic)
- [ ] Ports are interfaces, not implementations
- [ ] No dependency on infrastructure or UI

## 🔌 Infrastructure

- [ ] Adapters correctly implement their ports
- [ ] Adapters handle errors and return `Result`
- [ ] No business logic in adapters (just mapping/transformation)
- [ ] API calls are typed (request and response)
- [ ] Mapping snake_case → camelCase if needed

## 🖼️ UI Layer

- [ ] ViewModels expose `{ state, handlers }` only
- [ ] No business logic in viewModels (delegate to use cases)
- [ ] Components are at the correct atomic design level
- [ ] Screens use viewModels, no inline logic
- [ ] Dependencies are injected via `useDependencies()`
- [ ] Use case instantiated outside the calling function

## 🔄 React Query

- [ ] Query keys use the factory pattern (`entityKeys.detail(id)`)
- [ ] Mutations invalidate the correct queries
- [ ] Proper handling of loading/error states
- [ ] Simple CRUD → direct adapter
- [ ] Business logic → use case

## 📝 Conventions

- [ ] File naming: `.entity.ts`, `.port.ts`, `.usecase.ts`, `.adapter.ts`, `.viewModel.tsx`, `.store.ts`, `.query.ts`, `.mutation.ts`
- [ ] Components: `PascalCase.tsx`
- [ ] No explicit or implicit `any`
- [ ] Types are exported from the correct files

## ⚠️ Red Flags

- [ ] ❌ `useEffect` with complex business logic
- [ ] ❌ State management in components instead of viewModels
- [ ] ❌ Direct API calls in components
- [ ] ❌ Circular imports between modules
- [ ] ❌ Duplicated logic that should be in `shared/`
- [ ] ❌ Pass-through use case without logic (use direct adapter)
- [ ] ❌ `new UseCase()` inside a function body called multiple times
