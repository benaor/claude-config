# Design Refactor

Refactor code to fix design principle violations. Plans with Opus, executes with Sonnet, confirms each file modification.

## Usage

```bash
# Refactor specific file
/design-refactor src/services/UserService.ts

# Refactor directory
/design-refactor src/services/

# From branch diff
/design-refactor --branch
/design-refactor --branch develop

# From uncommitted changes
/design-refactor --uncommitted

# Filter by principles
/design-refactor src/services/ --principles solid,dry
```

## Arguments

| Flag | Description |
|------|-------------|
| `<path>` | File or directory to refactor |
| `--branch [name]` | Refactor branch diff (default: main/master) |
| `--uncommitted` | Refactor uncommitted changes |
| `--principles <list>` | Comma-separated list of principles to apply |

## Available Principles

`solid`, `yagni`, `kiss`, `dry`, `wet`, `law-of-demeter`, `tell-dont-ask`, `cqs`, `cqrs`, `fail-fast`, `pola`, `least-astonishment`, `composition-over-inheritance`, `separation-of-concerns`

## Workflow

1. **Plan**: Analyzes code, generates detailed refactoring plan (Opus)
2. **Confirm**: Presents plan, asks for confirmation
3. **Execute**: Applies changes one file at a time (Sonnet)
4. **Verify**: Shows before/after, asks confirmation per file
5. **Summary**: Lists all changes, suggests commit messages

## Controls

At each file modification:
- `y` — Apply change, continue
- `n` — Skip this file, continue
- `abort` — Stop refactoring

## Agent

Invokes `design-refactor` agent.
