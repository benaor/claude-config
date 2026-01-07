---
name: react-test-coverage
description: Identify missing test coverage in a module
arguments:
  - name: path
    description: Path to module/folder (optional, detects current context if omitted)
    required: false
  - name: options
    description: Options like --high-only, --generate, --json
    required: false
---

# /react-test-coverage

Identify missing test coverage using the `react-test-coverage-gap` agent.

## Usage

```
/react-test-coverage [path] [options]
```

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `path` | No | Path to module/folder. If omitted, uses current module or prompts. |
| `options` | No | `--high-only`, `--generate`, `--json` |

## Behavior

### 1. Resolve target path

```
IF path argument provided:
  IF path == "." OR path == "current":
    → Detect module from currently discussed file
    → Example: file in src/modules/auth/core/... → use src/modules/auth
  ELSE:
    → Use provided path
ELSE:
  → Check if a file is currently being discussed
  → If yes: derive module path from that file
  → If no: prompt user "Enter module path:"
```

### 2. Validate path

- Path must exist
- Path must be a directory
- Path must contain `.ts` or `.tsx` files

### 3. Invoke agent

```
Load and execute agent: .claude/agents/react-test-coverage-gap.md
Pass: resolved path + options
```

## Examples

```bash
# Scan specific module
/react-test-coverage src/modules/events

# Current module (derived from current file)
/react-test-coverage .

# Only show critical gaps
/react-test-coverage src/modules/auth --high-only

# Generate tests for all gaps
/react-test-coverage src/modules/favorites --generate

# JSON output for CI
/react-test-coverage src/modules/events --json

# No argument — will prompt or detect from current file
/react-test-coverage
```

## Options

| Option | Description |
|--------|-------------|
| `--high-only` | Show only HIGH priority gaps |
| `--generate` | Auto-generate tests for gaps (calls react-test-writer) |
| `--json` | Output as JSON for CI integration |

## Output

Standard report:
```
📊 Coverage Gap Report: <module>

🔴 HIGH PRIORITY:
  - <file> — <reason>

🟡 MEDIUM PRIORITY:
  - <file> — <reason>

📈 Pyramid Balance:
  Unit: X tests (Y%)
  Integration: X tests (Y%)
  E2E: X tests (Y%)
```

With `--generate`:
```
🔧 Generating tests...

✅ Created: <test-file> (X tests)
✅ Added: X tests to <existing-test-file>

Total: X new tests generated
```

With `--json`:
```json
{
  "module": "...",
  "summary": { ... },
  "gaps": [ ... ],
  "pyramid": { ... }
}
```

## Errors

| Error | Message |
|-------|---------|
| No path provided | `❌ No module specified. Usage: /react-test-coverage <path>` |
| Path not found | `❌ Module not found: <path>` |
| Not a directory | `⚠️ Path is not a directory: <path>` |
| No source files | `⚠️ No .ts/.tsx files found in <path>` |
