# /codebase-ca-react-review

Broad architecture review of entire codebase for Clean Architecture compliance.

## Usage

```bash
# Full codebase review
/codebase-ca-react-review

# Specific directory
/codebase-ca-react-review src/modules/

# Filter checks
/codebase-ca-react-review --check layers,patterns
```

## Arguments

| Flag | Description |
|------|-------------|
| `[path]` | Directory to scope review (default: src/) |
| `--check <list>` | Comma-separated checks to run (default: all) |

## Checks

| Check | Description |
|-------|-------------|
| `layers` | Core/Infrastructure/UI separation across modules |
| `patterns` | Result pattern adoption, Ports/Adapters consistency |
| `naming` | File extension conventions across codebase |
| `react-query` | Query keys factory usage, mutation patterns |
| `di` | Dependency injection setup, registration completeness |

All checks run by default. Use `--check` to filter.

## Behavior

1. **Discovery**
   - Map project structure
   - Identify bounded contexts in `modules/`
   - Count files by layer

2. **Sampling strategy**
   - **All files** in `modules/*/core/` (business critical)
   - **Largest 10 files** elsewhere (likely problem areas)
   - **Entry points**: `App.tsx`, navigation, DI setup
   - **10% random sample** of remaining files

3. **Analysis**
   - Run checks on sampled files
   - Track violation patterns (systemic vs isolated)

4. **Health Score**
   - Calculate: `10 - (critical × 2) - (major × 1) - (minor × 0.25)`
   - Minimum: 0, Maximum: 10

5. **Generate roadmap**
   - Phase 1: Critical (layer violations)
   - Phase 2: Major (pattern violations)
   - Phase 3: Polish (conventions)

6. **Output**
   - Executive summary
   - Health score with breakdown
   - Prioritized roadmap for refactor agent

## Output

```markdown
## Codebase Architecture Review

### Health Score: 6.5/10 🟠

### Executive Summary

The codebase follows Clean Architecture structure but has inconsistent pattern adoption. Core layer is mostly clean with 2 critical import violations. ViewModels frequently contain business logic that should be in UseCases. Query key factories are missing in 4/7 modules.

### By Category

#### Layers (8/10)
✅ Clear module boundaries with bounded contexts
✅ Infrastructure properly isolated
⚠️ 2 Core files import from Infrastructure
⚠️ 1 Core file imports React

#### Patterns (5/10)
✅ Result pattern used in 60% of UseCases
⚠️ 5 ViewModels contain business logic
⚠️ 3 UseCases missing Result return type
⚠️ 2 Adapters contain business logic

#### Naming (7/10)
✅ Component naming consistent (PascalCase)
⚠️ 12 entity files missing .entity.ts extension
⚠️ 4 port files missing .port.ts extension

#### React Query (6/10)
✅ Mutations properly invalidate queries
⚠️ 4/7 modules missing query key factories
⚠️ 3 inline query keys in hooks

#### DI (9/10)
✅ useDependencies() used consistently
✅ All adapters registered
⚠️ 2 UseCase instantiations inside function body

### Critical Issues

1. **Core importing Infrastructure** (2 files)
   - `modules/auth/core/usecases/Login.usecase.ts`
   - `modules/events/core/usecases/CreateEvent.usecase.ts`

2. **Business logic in ViewModels** (5 files)
   - Validation logic should be in UseCases

3. **Missing query key factories** (4 modules)
   - auth, events, profile, settings

### Refactoring Roadmap

**Phase 1 — Critical (Week 1)**
- Fix Core layer imports (2 files)
- Extract business logic from ViewModels (5 files)

**Phase 2 — Major (Week 2-3)**
- Implement Result pattern in remaining UseCases (3 files)
- Move business logic from Adapters (2 files)
- Create query key factories (4 modules)

**Phase 3 — Polish (Week 4)**
- Rename entity files (12 files)
- Rename port files (4 files)
- Fix UseCase instantiation (2 files)

---
Run `/codebase-ca-react-refactor` to apply fixes phase by phase.
```

## Agent

Invokes `ca-react-reviewer` agent with codebase scope.
