---
type: command
name: codebase-ddd-review
description: Architectural DDD analysis applying Pareto principle
agent: codebase-ddd-review
---

# /codebase-ddd-review

Invoke the codebase DDD review agent for architectural analysis.

## Usage

```bash
# Analyze entire codebase
/codebase-ddd-review

# Specify domain location
/codebase-ddd-review --domain src/core/

# Focus on specific contexts
/codebase-ddd-review --contexts sales,billing
```

## Options

| Option | Description |
|--------|-------------|
| `--domain <path>` | Specify domain layer location |
| `--contexts <list>` | Comma-separated list of contexts to analyze |

## What it analyzes

**Strategic patterns (Priority 1-4):**
- Bounded context violations (cross-imports, shared DB)
- Aggregate boundary violations (multi-aggregate transactions)
- Ubiquitous language inconsistencies
- Missing Anti-Corruption Layers

**Tactical patterns (Priority 5-6):**
- Repository violations
- Anemic domain model

## Output

- Executive summary
- Visual architecture map
- Prioritized critical/serious/moderate issues
- Health score per bounded context
- Action plan (immediate/short-term/medium-term)

## Examples

```bash
# Full codebase review
/codebase-ddd-review

# Then drill down
/ddd-review src/domain/sales/

# Or apply architectural fixes
/codebase-ddd-refactor
```

## See also

- `/ddd-review` — Detailed per-file analysis
- `/codebase-ddd-refactor` — Apply architectural fixes
