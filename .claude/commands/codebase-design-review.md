# Codebase Design Review

Broad review of entire codebase for design principle violations. Uses Pareto approach — focuses on the 20% of issues causing 80% of problems.

## Usage

```bash
# Full codebase review
/codebase-design-review

# Review specific directory
/codebase-design-review src/domain/

# Filter by principles
/codebase-design-review --principles solid,separation-of-concerns
```

## Arguments

| Flag | Description |
|------|-------------|
| `[path]` | Optional directory to scope review |
| `--principles <list>` | Comma-separated list of principles to check |

## Available Principles

`solid`, `yagni`, `kiss`, `dry`, `wet`, `law-of-demeter`, `tell-dont-ask`, `cqs`, `cqrs`, `fail-fast`, `pola`, `least-astonishment`, `composition-over-inheritance`, `separation-of-concerns`

## Sampling Strategy

Analyzes strategically, not exhaustively:
- All files in `domain/`, `core/`, `entities/`
- Largest 10 files (likely problem areas)
- 10% random sample of remaining files
- Entry points and DI setup

## Output

Generates executive report with:
- Health Score (X/10)
- Top 3 critical issues
- Architecture assessment (strengths/concerns)
- Critical issues with systemic patterns
- Refactoring roadmap (phased by week)
- Files requiring attention (prioritized table)
- Option to launch codebase refactor agent

## Agent

Invokes `codebase-design-review` agent with Opus model.
