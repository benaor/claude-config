# /ca-react-review

Review code for Clean Architecture compliance.

## Usage

```bash
# Branch diff (default: main/master)
/ca-react-review

# Specific branch
/ca-react-review --branch develop

# Uncommitted changes
/ca-react-review --uncommitted

# Staged only
/ca-react-review --staged

# Specific path
/ca-react-review src/modules/auth/

# Filter checks
/ca-react-review --check layers,patterns
```

## Arguments

| Flag | Description |
|------|-------------|
| `[path]` | File or directory to review |
| `--branch [name]` | Review branch diff (default: main/master) |
| `--uncommitted` | Review uncommitted changes |
| `--staged` | Review staged changes only |
| `--check <list>` | Comma-separated checks to run (default: all) |

## Checks

| Check | Description |
|-------|-------------|
| `layers` | Core/Infrastructure/UI separation, import directions |
| `patterns` | Result pattern, Ports/Adapters, ViewModel structure |
| `naming` | File extensions, casing conventions |
| `react-query` | Query keys factory, mutation invalidations |
| `di` | Dependency injection, UseCase instantiation |

All checks run by default. Use `--check` to filter.

## Behavior

1. **Resolve scope**
   - Default: diff against main/master
   - With `--branch`: diff against specified branch
   - With `--uncommitted`: all uncommitted changes
   - With `--staged`: staged changes only
   - With path: specific file/directory

2. **Load agent**
   - Invoke `ca-react-reviewer` agent

3. **Execute review**
   - Run specified checks (or all)
   - Categorize violations by severity

4. **Output**
   - List violations grouped by severity
   - Summary with counts
   - Suggest `/ca-react-refactor` if violations found

## Output

```markdown
## Review: [scope]

### 🔴 Critical (2)

**src/modules/auth/core/usecases/Login.usecase.ts** (line 3)
- Violation: Core imports from Infrastructure
- Fix: Use Port interface instead of concrete Adapter

### 🟠 Major (3)
...

### Summary
- Files reviewed: 12
- Violations: 2 critical, 3 major, 5 minor

---
Run `/ca-react-refactor` to fix these issues.
```

## Agent

Invokes `ca-react-reviewer` agent.
