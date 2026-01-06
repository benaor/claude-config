# /codebase-ca-react-refactor

Large-scale refactoring across entire codebase toward Clean Architecture.

## Usage

```bash
# Full codebase refactor (uses roadmap from previous review)
/codebase-ca-react-refactor

# Execute specific phase
/codebase-ca-react-refactor --phase 1

# Specific directory
/codebase-ca-react-refactor src/modules/auth/

# Filter checks
/codebase-ca-react-refactor --check layers,patterns
```

## Arguments

| Flag | Description |
|------|-------------|
| `[path]` | Directory to scope refactor |
| `--phase <n>` | Execute specific phase from previous roadmap |
| `--check <list>` | Comma-separated checks to apply (default: all) |

## Checks

| Check | Description |
|-------|-------------|
| `layers` | Fix layer violations, correct imports |
| `patterns` | Implement Result pattern, extract Ports/Adapters, restructure ViewModels |
| `naming` | Rename files to correct extensions |
| `react-query` | Extract query key factories, add invalidations |
| `di` | Fix dependency injection, UseCase instantiation |

All checks run by default. Use `--check` to filter.

## Behavior

1. **Load roadmap**
   - From previous `/codebase-ca-react-review` if available
   - Otherwise: run quick analysis to generate roadmap

2. **Plan phases**
   - Phase 1: Critical (layer violations)
   - Phase 2: Major (pattern violations)
   - Phase 3: Polish (conventions)

3. **For each phase:**
   - Present phase scope and files
   - Ask: `Continue / Skip phase / Abort`
   - If continue:
     - Apply all changes in phase
     - Run type check
     - Report results
   - If skip: move to next phase
   - If abort: stop completely

4. **Summary**
   - List all applied changes
   - Suggest commit messages per phase

## Output

### Phase Execution

```markdown
## Codebase Refactoring

### Phase 1: Critical (Layer Violations)

**Files to modify (3):**
- `modules/auth/core/usecases/Login.usecase.ts` — Remove Infrastructure import
- `modules/events/core/usecases/CreateEvent.usecase.ts` — Remove Infrastructure import
- `modules/auth/core/entities/User.entity.ts` — Remove React import

**Changes:**
- Replace concrete Adapter imports with Port interfaces
- Remove React dependencies from Core

---
**Continue / Skip phase / Abort?** 
```

### After Phase Completion

```markdown
### Phase 1: Complete ✅

**Applied:**
- Modified: 3 files
- Type check: ✅ Passed

**Suggested commit:**
```
fix(core): remove Infrastructure and React imports from Core layer
```

---

### Phase 2: Major (Pattern Violations)

**Files to modify (10):**
- `modules/auth/ui/viewModels/useLogin.viewModel.tsx` — Extract validation to UseCase
- `modules/events/ui/viewModels/useCreateEvent.viewModel.tsx` — Extract validation
- ... (8 more)

**Files to create (5):**
- `modules/auth/core/usecases/ValidateLogin.usecase.ts`
- ... (4 more)

---
**Continue / Skip phase / Abort?**
```

### Final Summary

```markdown
## Codebase Refactoring Complete ✅

### Summary

| Phase | Status | Files Modified | Files Created |
|-------|--------|----------------|---------------|
| 1. Critical | ✅ Applied | 3 | 0 |
| 2. Major | ✅ Applied | 10 | 5 |
| 3. Polish | ⏭️ Skipped | - | - |

### Type check: ✅ Passed

### Suggested commits

```
fix(core): remove Infrastructure and React imports from Core layer
refactor(auth): extract validation logic to UseCases
refactor(events): implement Result pattern in UseCases
```

### Remaining work (Phase 3)

- Rename 12 entity files to .entity.ts
- Rename 4 port files to .port.ts
- Fix 2 UseCase instantiations

Run `/codebase-ca-react-refactor --phase 3` to complete.
```

## Escape Hatches

At any phase prompt:

| Input | Action |
|-------|--------|
| `continue` or `y` | Apply phase, proceed to next |
| `skip` or `s` | Skip phase, proceed to next |
| `abort` or `q` | Stop refactoring entirely |

## Agent

Invokes `ca-react-refactorer` agent with codebase scope.
