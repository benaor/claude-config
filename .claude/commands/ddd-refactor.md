---
type: command
name: ddd-refactor
description: Apply DDD tactical pattern fixes to code
agent: ddd-refactor
---

# /ddd-refactor

Invoke the DDD refactor agent to fix tactical pattern violations.

## Usage

```bash
# Refactor single file
/ddd-refactor src/domain/Order.ts

# Refactor multiple files
/ddd-refactor src/domain/Order.ts src/domain/Customer.ts

# Use output from previous ddd-review
/ddd-refactor --from-review

# Fix specific violation type only
/ddd-refactor src/domain/Order.ts --fix primitive-obsession
/ddd-refactor src/domain/Order.ts --fix anemic-entity
```

## Options

| Option | Description |
|--------|-------------|
| `--from-review` | Use violations from last `ddd-review` output |
| `--fix <type>` | Target specific violation type |
| `--dry-run` | Preview only, don't prompt for apply |

## Available fix types

| Type | Description |
|------|-------------|
| `primitive-obsession` | Extract Value Objects from primitives |
| `anemic-entity` | Move logic into entity methods |
| `exposed-aggregate` | Protect aggregate internals |
| `missing-factory` | Add factory methods |
| `mutable-vo` | Make Value Object immutable |
| `wrong-equality` | Fix equals() implementation |
| `missing-domain-event` | Add domain event dispatch |
| `stateful-service` | Make domain service stateless |

## Workflow

1. **Analysis** — Detects violations (Opus)
2. **Preview** — Shows all planned changes
3. **Confirm** — User approves or aborts
4. **Apply** — Executes changes (Sonnet)
5. **Report** — Summary of changes made

## Examples

```bash
# After running ddd-review
/ddd-review src/domain/
# Output shows: ddd-refactor src/domain/Order.ts — 2 high, 1 medium

/ddd-refactor src/domain/Order.ts
# Shows preview, apply on confirm
```

## See also

- `/ddd-review` — Analyze code for violations (run first)
- `/codebase-ddd-review` — Architectural DDD analysis
