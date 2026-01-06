# /ca-react-refactor

Refactor code to comply with Clean Architecture.

## Usage

```bash
# Branch diff (default: main/master)
/ca-react-refactor

# Specific branch
/ca-react-refactor --branch develop

# Uncommitted changes
/ca-react-refactor --uncommitted

# Specific path
/ca-react-refactor src/modules/auth/

# Filter checks
/ca-react-refactor --check layers,patterns

# Preview only
/ca-react-refactor --dry-run
```

## Arguments

| Flag | Description |
|------|-------------|
| `[path]` | File or directory to refactor |
| `--branch [name]` | Refactor branch diff (default: main/master) |
| `--uncommitted` | Refactor uncommitted changes |
| `--check <list>` | Comma-separated checks to apply (default: all) |
| `--dry-run` | Show plan without applying changes |

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

1. **Resolve scope**
   - Default: diff against main/master
   - With `--branch`: diff against specified branch
   - With `--uncommitted`: all uncommitted changes
   - With path: specific file/directory

2. **Load agent**
   - Invoke `ca-react-refactorer` agent

3. **Analyze**
   - Identify violations in scope
   - Generate refactoring plan

4. **Present plan**
   - Show all proposed changes
   - List files to modify/create/rename

5. **If `--dry-run`**
   - Stop after presenting plan

6. **Confirm**
   - Ask user: "Apply all changes? (y/n)"
   - All or nothing (no partial apply)

7. **Execute**
   - Apply all changes
   - Run type check
   - Verify no regressions

8. **Summary**
   - List applied changes
   - Suggest commit messages

## Output

### Plan (shown before confirmation)

```markdown
## Refactoring Plan

### Files to modify (3)

**1. src/modules/auth/ui/viewModels/useLogin.viewModel.tsx**
- Extract validation logic to new UseCase
- Restructure return to {state, handlers}

**2. [NEW] src/modules/auth/core/usecases/Login.usecase.ts**
- Create with extracted validation logic
- Implement Result pattern

**3. src/modules/app/dependencies/Dependencies.type.ts**
- Register new UseCase dependency

### Files to rename (2)

- `User.ts` → `User.entity.ts`
- `loginScreen.tsx` → `LoginScreen.tsx`

---
Apply all changes? (y/n)
```

### After Apply

```markdown
## Refactoring Complete ✅

### Changes applied
- Modified: 3 files
- Created: 1 file
- Renamed: 2 files

### Type check: ✅ Passed

### Suggested commits

```
refactor(auth): extract login validation to UseCase
chore(auth): rename files to correct extensions
```
```

## Agent

Invokes `ca-react-refactorer` agent.
