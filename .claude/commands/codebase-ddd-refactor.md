---
type: command
name: codebase-ddd-refactor
description: Apply architectural DDD fixes across codebase
agent: codebase-ddd-refactor
---

# /codebase-ddd-refactor

Invoke the codebase DDD refactor agent for architectural fixes.

## Usage

```bash
# From previous review (recommended)
/codebase-ddd-refactor --from-review

# Full analysis and refactor
/codebase-ddd-refactor

# Specific architectural fix
/codebase-ddd-refactor --fix context-boundaries
/codebase-ddd-refactor --fix add-acl stripe
/codebase-ddd-refactor --fix split-aggregate Order
```

## Options

| Option | Description |
|--------|-------------|
| `--from-review` | Use last `codebase-ddd-review` output |
| `--fix <type>` | Apply specific architectural fix |
| `--dry-run` | Preview only, don't prompt for apply |

## Available fix types

| Type | Description |
|------|-------------|
| `context-boundaries` | Enforce bounded context separation |
| `add-acl <system>` | Create Anti-Corruption Layer |
| `split-aggregate <name>` | Break oversized aggregate |
| `extract-context <name>` | Extract new bounded context |
| `add-domain-events` | Add event dispatch to aggregates |
| `unify-language <term>` | Rename to consistent ubiquitous term |

## Workflow

1. **Analysis** — Plan architectural changes (Opus)
2. **Preview** — Show all changes with diffs
3. **Confirm** — User approves or selects subset
4. **Apply** — Execute changes (Sonnet)
5. **Report** — Summary + manual steps required

## Example workflow

```bash
# 1. Review codebase
/codebase-ddd-review

# 2. Apply recommended fixes
/codebase-ddd-refactor --from-review

# 3. Verify improvements
/codebase-ddd-review
```

## See also

- `/codebase-ddd-review` — Architectural analysis (run first)
- `/ddd-refactor` — Per-file tactical refactoring
