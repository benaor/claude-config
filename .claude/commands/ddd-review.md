---
type: command
name: ddd-review
description: Analyze code for DDD tactical pattern violations
agent: ddd-reviewer
---

# /ddd-review

Invoke the DDD reviewer agent to analyze code for tactical pattern violations.

## Usage

```bash
# Default: analyze current git changes
/ddd-review

# Analyze specific file
/ddd-review src/domain/Order.ts

# Analyze multiple files
/ddd-review src/domain/Order.ts src/domain/Customer.ts

# Analyze directory
/ddd-review src/domain/

# Analyze branch diff
/ddd-review --branch feature/checkout
```

## Options

| Option | Description |
|--------|-------------|
| `--branch <name>` | Compare specified branch to main/master |
| `--staged` | Only analyze staged changes |
| `--all` | Include low-severity issues (hidden by default if > 10 violations) |

## Output

Produces a prioritized report:
- 🔴 **High**: Structural DDD violations
- 🟠 **Medium**: Pattern violations
- 🟡 **Low**: Improvement opportunities

## Next steps

After review, use `/ddd-refactor` to apply fixes to specific files.
