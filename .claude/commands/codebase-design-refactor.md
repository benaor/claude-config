# Codebase Design Refactor

Large-scale refactoring across entire codebase. Plans strategically with Opus, executes by phases with Sonnet, provides escape hatches at every step.

## Usage

```bash
# Full codebase refactor
/codebase-design-refactor

# Execute specific phase from roadmap
/codebase-design-refactor --phase 1

# Refactor specific directory
/codebase-design-refactor src/services/

# Filter by principles
/codebase-design-refactor --principles solid
```

## Arguments

| Flag | Description |
|------|-------------|
| `[path]` | Optional directory to scope refactor |
| `--phase <n>` | Execute specific phase from previous roadmap |
| `--principles <list>` | Comma-separated list of principles to apply |

## Available Principles

`solid`, `yagni`, `kiss`, `dry`, `wet`, `law-of-demeter`, `tell-dont-ask`, `cqs`, `cqrs`, `fail-fast`, `pola`, `least-astonishment`, `composition-over-inheritance`, `separation-of-concerns`

## Workflow

1. **Assess**: Quick analysis, identify critical issues (Opus)
2. **Plan**: Generate phased roadmap
3. **Confirm**: Present strategy, safety checklist
4. **Execute**: Phase by phase, file by file (Sonnet)
5. **Verify**: Suggest test runs between phases
6. **Summary**: List all changes, remaining issues, suggested commits

## Controls

At each step:
- `y` — Apply change, continue
- `n` — Skip this file, continue
- `skip phase` — Skip to next phase
- `abort` — Stop refactoring
- `commit first` — Pause to commit before continuing

## Safety

- Recommends creating branch before starting
- Never modifies test files (flags for manual review)
- Each phase leaves codebase in working state
- Suggests verification commands between phases

## Agent

Invokes `codebase-design-refactor` agent.
