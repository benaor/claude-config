---
name: react-test-refactor
description: Improve test quality by detecting and fixing test smells
arguments:
  - name: path
    description: Path to test file or folder (optional, detects current context if omitted)
    required: false
  - name: options
    description: Options like --check-only, --auto-fix, --category
    required: false
---

# /react-test-refactor

Improve test quality using the `react-test-refactor` agent.

## Usage

```
/react-test-refactor [path] [options]
```

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `path` | No | Path to test file or folder. If omitted, uses current file or prompts. |
| `options` | No | `--check-only`, `--auto-fix`, `--category <cat>` |

## Behavior

### 1. Resolve target path

```
IF path argument provided:
  IF path == "." OR path == "current":
    → Use currently open/discussed test file
    → OR derive test folder from current source file
  ELSE:
    → Use provided path (file or folder)
ELSE:
  → Check if a test file is currently being discussed
  → If yes: use that file
  → If no: prompt user "Enter test file or folder path:"
```

### 2. Validate path

- Path must exist
- If file: must be a test file (`.test.ts` or `.test.tsx`)
- If folder: must contain test files

### 3. Invoke agent

```
Load and execute agent: .claude/agents/react-test-refactor.md
Pass: resolved path + options
```

## Examples

```bash
# Analyze specific test file
/react-test-refactor src/modules/auth/useAuth.test.ts

# Analyze all tests in a folder
/react-test-refactor src/modules/auth/__tests__/

# Current file/context shortcut
/react-test-refactor .

# Check only, no fixes
/react-test-refactor src/modules/events --check-only

# Auto-fix safe issues
/react-test-refactor src/modules/auth --auto-fix

# Focus on flaky tests only
/react-test-refactor src/modules/auth --category flaky

# Focus on slow tests
/react-test-refactor src/modules/events --category slow

# No argument — will prompt or use current file
/react-test-refactor
```

## Options

| Option | Description |
|--------|-------------|
| `--check-only` | Report issues without fixing |
| `--auto-fix` | Apply safe fixes automatically |
| `--category <cat>` | Focus on specific smell category |

### Categories

| Category | Description |
|----------|-------------|
| `flaky` | Timing-dependent, random, shared state |
| `slow` | Real timers, network, heavy setup |
| `fragile` | Tests implementation details |
| `obscure` | Hard to understand, missing AAA |
| `coupled` | Tests depend on each other |
| `over-mocked` | Too many mocks (>3) |

## Output

Summary report:
```
🔍 Test Quality Report: <path>

Issues found: X

  🎲 FLAKY:       X (HIGH)
  🐢 SLOW:        X (MEDIUM)
  🔨 FRAGILE:     X (MEDIUM)
  🌫️ OBSCURE:     X (LOW)
  🔗 COUPLED:     X (HIGH)
  🎭 OVER-MOCKED: X (MEDIUM)
```

Per-issue detail:
```
Issue #1: FLAKY — <file>:<line>

❌ Before:
  <code snippet>

✅ After:
  <fixed code>

Why: <explanation>

Apply fix? [Y/n]:
```

After fixes:
```
✅ Test Refactoring Complete

Applied: X fixes
Skipped: X
Files modified: <list>

Verification: ✅ All tests passing
```

## Errors

| Error | Message |
|-------|---------|
| No path provided | `❌ No path specified. Usage: /react-test-refactor <path>` |
| Path not found | `❌ Path not found: <path>` |
| No test files | `⚠️ No test files found in <path>` |
| Invalid category | `⚠️ Unknown category: <cat>. Valid: flaky, slow, fragile, obscure, coupled, over-mocked` |
| Fix breaks tests | `⚠️ Fix introduced failures. Rolling back...` |
