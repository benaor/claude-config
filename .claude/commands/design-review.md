# Design Review

Review code for design principle violations. Analyzes targeted scope (branch diff, uncommitted changes, or specific files) and generates a detailed report with refactoring plan.

## Usage

```bash
# Auto-detect context (will prompt for confirmation)
/design-review

# Compare current branch to main/master
/design-review --branch

# Compare to specific branch
/design-review --branch develop

# Review uncommitted changes (staged + unstaged)
/design-review --uncommitted

# Review specific file
/design-review --file src/services/UserService.ts

# Review specific directory
/design-review --file src/domain/

# Filter by principles
/design-review --principles solid,dry,kiss

# Combined
/design-review --branch feature/auth --principles solid
```

## Arguments

| Flag | Description |
|------|-------------|
| `--branch [name]` | Compare to branch (default: main/master) |
| `--uncommitted` | Review staged + unstaged changes |
| `--file <path>` | Review specific file or directory |
| `--principles <list>` | Comma-separated list of principles to check |

## Available Principles

`solid`, `yagni`, `kiss`, `dry`, `wet`, `law-of-demeter`, `tell-dont-ask`, `cqs`, `cqrs`, `fail-fast`, `pola`, `least-astonishment`, `composition-over-inheritance`, `separation-of-concerns`

## Output

Generates a structured report with:
- Summary (files analyzed, violations by severity)
- Violations grouped by principle
- Detailed refactoring plan
- Option to launch refactor agent

## Agent

Invokes `design-reviewer` agent with Opus model.
