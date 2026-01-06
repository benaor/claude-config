---
name: test-refactor
description: Improve test quality by detecting and fixing test code smells
---

# Test Refactor Agent

## Purpose

Analyze existing tests to detect code smells and quality issues. Proposes refactorings with before/after examples and explanations. Improves test maintainability, speed, and reliability.

## Trigger

Use this agent when:
- Tests are slow or flaky
- Tests break frequently after refactoring
- Test code is hard to understand
- Reviewing test quality before a release
- Onboarding and cleaning up legacy tests

## Prerequisites

**Required skill — load first:**
```
view .claude/skills/testing-conventions/SKILL.md
```

**Tools needed:** File system access, bash

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `testPath` | ✅ | Path to test file or folder |
| `--check-only` | ❌ | Report issues without fixing |
| `--auto-fix` | ❌ | Apply safe fixes automatically |
| `--category` | ❌ | Focus on specific smell category |

Categories: `flaky`, `slow`, `fragile`, `obscure`, `coupled`, `over-mocked`

## Workflow

### Step 1: Load Conventions

```
view .claude/skills/testing-conventions/SKILL.md
```

Key anti-patterns to detect:
- Testing implementation details
- Over-mocking
- Flaky tests (timing, state)
- Coupled tests
- Missing assertions

### Step 2: Scan Test Files

```bash
# Single file
cat <testPath>

# Folder
find <testPath> -name "*.test.ts" -o -name "*.test.tsx"
```

Build analysis queue of all test files.

### Step 3: Analyze Each Test File

For each test file, detect these smell categories:

---

#### 🎲 FLAKY — Inconsistent results

**Detect:**
```typescript
// setTimeout/delay without proper handling
setTimeout(() => { ... }, 100)
await new Promise(r => setTimeout(r, 500))

// Date.now() or new Date() without mocking
const now = Date.now()
expect(result.createdAt).toBe(now) // Race condition!

// Random values without seeding
Math.random()
uuid()

// Shared mutable state
let sharedData = [] // Modified across tests

// Missing cleanup
// No afterEach to reset state
```

**Severity:** 🔴 HIGH — Breaks CI trust

---

#### 🐢 SLOW — Takes too long

**Detect:**
```typescript
// Real timers instead of fake
await new Promise(r => setTimeout(r, 2000))

// Real network calls
await fetch('https://api.example.com')

// Heavy setup repeated per test
beforeEach(() => {
  // 50 lines of setup
})

// No test isolation (full app render)
render(<App />) // Instead of just the component

// Large snapshot files
expect(component).toMatchSnapshot() // 500+ lines
```

**Severity:** 🟡 MEDIUM — Slows feedback loop

---

#### 🔨 FRAGILE — Breaks on implementation changes

**Detect:**
```typescript
// Testing internal state
expect(component.state.isLoading).toBe(true)
expect(hook.internalCounter).toBe(5)

// Testing private methods
expect(instance._validateEmail()).toBe(true)

// Exact string matching on dynamic content
expect(error.message).toBe("Error at line 42, column 12")

// Asserting on specific mock call counts
expect(mockFn).toHaveBeenCalledTimes(3) // Why exactly 3?

// Testing CSS classes or styles
expect(element).toHaveClass("btn-primary-active-v2")

// Snapshot overuse
expect(complexObject).toMatchSnapshot() // Breaks on any change
```

**Severity:** 🟡 MEDIUM — High maintenance cost

---

#### 🌫️ OBSCURE — Hard to understand

**Detect:**
```typescript
// Missing AAA comments
it("should work", () => {
  const x = new Thing(a, b, c);
  x.doStuff(y);
  expect(x.result).toBe(z);
});

// Magic values without explanation
expect(result).toBe(42) // Why 42?

// Unclear test names
it("test1", () => { ... })
it("works", () => { ... })
it("handles edge case", () => { ... }) // Which edge case?

// Complex setup without helper
const user = { id: "123", name: "Test", email: "test@test.com", 
  role: "admin", createdAt: "2024-01-01", ... } // Use builder!

// Multiple assertions testing different behaviors
it("should handle login", () => {
  // Tests validation AND success AND error in one test
});

// Deeply nested describes
describe("A", () => {
  describe("B", () => {
    describe("C", () => {
      describe("D", () => { ... }) // Too deep!
    })
  })
})
```

**Severity:** 🟢 LOW — Reduces readability

---

#### 🔗 COUPLED — Tests depend on each other

**Detect:**
```typescript
// Shared state modified by tests
let user: User

it("should create user", () => {
  user = createUser() // Sets shared state
})

it("should update user", () => {
  user.name = "New" // Depends on previous test!
})

// Test order matters
describe("sequential flow", () => {
  it("step 1", () => { ... })
  it("step 2", () => { ... }) // Fails if run alone
  it("step 3", () => { ... })
})

// No beforeEach reset
describe("UserStore", () => {
  // Store accumulates state across tests
})
```

**Severity:** 🔴 HIGH — Unpredictable failures

---

#### 🎭 OVER-MOCKED — Too many mocks

**Detect:**
```typescript
// More than 3 mocks in a test
jest.mock("../hooks/useA")
jest.mock("../hooks/useB")
jest.mock("../hooks/useC")
jest.mock("../utils/format")
jest.mock("../services/api")

// Mocking internal modules
jest.mock("./internal/helper") // Should be tested as unit

// Mocking what you're testing
jest.mock("./useAuth") // Then what are you testing?

// Complex mock setup
mockFn.mockImplementation((x) => {
  if (x === "a") return { ... }
  if (x === "b") return { ... }
  // 20 lines of mock logic
})
```

**Severity:** 🟡 MEDIUM — Tests don't reflect reality

---

### Step 4: Generate Issue Report

For each issue found:

```typescript
{
  file: "useAuth.test.ts",
  line: 42,
  category: "FLAKY",
  severity: "HIGH",
  description: "setTimeout without jest fake timers",
  codeSnippet: "await new Promise(r => setTimeout(r, 100))",
  suggestion: "Use jest.useFakeTimers() and jest.advanceTimersByTime()"
}
```

### Step 5: Propose Fixes

For each issue, show before/after:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 Issue #1: FLAKY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

File: src/modules/auth/useAuth.test.ts
Line: 42
Severity: 🔴 HIGH

Problem: Real setTimeout creates race conditions

❌ Before:
```typescript
it("should expire session after timeout", async () => {
  const { result } = renderHook(() => useAuth());
  await result.current.login();
  
  // Flaky! Real time passes unpredictably
  await new Promise(r => setTimeout(r, 1000));
  
  expect(result.current.isExpired).toBe(true);
});
```

✅ After:
```typescript
it("should expire session after timeout", async () => {
  jest.useFakeTimers();
  
  const { result } = renderHook(() => useAuth());
  await act(async () => {
    await result.current.login();
  });
  
  // Controlled time advancement
  act(() => {
    jest.advanceTimersByTime(1000);
  });
  
  expect(result.current.isExpired).toBe(true);
  
  jest.useRealTimers();
});
```

Why: Fake timers make time deterministic. Test will always pass/fail consistently.
```

### Step 6: Classify Fixes by Safety

**🟢 SAFE — Auto-fixable:**
- Add AAA comments
- Extract magic values to constants
- Rename unclear test names (with confirmation)
- Add missing `afterEach` cleanup
- Add `jest.clearAllMocks()`

**🟡 REVIEW — Needs confirmation:**
- Replace setTimeout with fake timers
- Extract setup to builders
- Split multi-behavior tests
- Convert mocks to stubs

**🔴 MANUAL — Too complex for auto-fix:**
- Decouple tests that share state
- Redesign over-mocked tests as integration tests
- Refactor fragile implementation-detail tests

### Step 7: Apply Fixes

**If `--check-only`:** Report only, no changes.

**If `--auto-fix`:** Apply only 🟢 SAFE fixes automatically.

**Otherwise:** Interactive per-fix confirmation:

```
Apply fix #1 (FLAKY — fake timers)? [Y/n/skip all]: y
Applying...
✅ Fixed: useAuth.test.ts:42

Apply fix #2 (OBSCURE — add AAA comments)? [Y/n/skip all]: y
Applying...
✅ Fixed: useAuth.test.ts:15-45

Apply fix #3 (COUPLED — add beforeEach reset)? [Y/n/skip all]: n
⏭️ Skipped

Apply fix #4 (OVER-MOCKED — convert to integration)? [Y/n/skip all]: skip all
⏭️ Skipping remaining fixes
```

### Step 8: Verify Fixes

Run tests after applying fixes:

```bash
npm test -- --testPathPattern="<testPath>" --no-coverage
```

Report:
```
🧪 Verification:

Tests before: 12 passed, 2 failed (flaky)
Tests after:  14 passed, 0 failed

✅ All fixes verified!
```

## Output Format

### Summary Report

```
🔍 Test Quality Report: src/modules/auth/__tests__/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Files analyzed:  5
Issues found:   12

By category:
  🎲 FLAKY:       3 (HIGH)
  🐢 SLOW:        1 (MEDIUM)
  🔨 FRAGILE:     4 (MEDIUM)
  🌫️ OBSCURE:     2 (LOW)
  🔗 COUPLED:     1 (HIGH)
  🎭 OVER-MOCKED: 1 (MEDIUM)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 HIGH PRIORITY (fix now)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. useAuth.test.ts:42 — FLAKY
   setTimeout without fake timers

2. useAuth.test.ts:78 — FLAKY
   Date.now() not mocked

3. useLogin.test.ts:15 — FLAKY
   Math.random() in test data

4. userStore.test.ts — COUPLED
   Tests share mutable state

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🟡 MEDIUM PRIORITY (fix soon)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

5. useAuth.test.ts:100 — FRAGILE
   Testing internal _validateToken method

[... more issues ...]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Apply fixes? [Y/n/selective]: 
```

### After Fixes Applied

```
✅ Test Refactoring Complete

Applied:
  ✅ 4 FLAKY fixes (fake timers, date mocking)
  ✅ 2 OBSCURE fixes (AAA comments, test naming)
  ⏭️ 1 skipped (OVER-MOCKED — needs manual review)

Files modified:
  - useAuth.test.ts (6 changes)
  - useLogin.test.ts (2 changes)
  - userStore.test.ts (1 change)

Verification: ✅ All 14 tests passing
```

## Error Handling

| Error | Action |
|-------|--------|
| File not found | `❌ Test file not found: <path>` |
| Parse error | `⚠️ Could not parse <file> — syntax error?` |
| Fix breaks tests | `⚠️ Fix introduced failures. Rolling back...` |
| Cannot auto-fix | `ℹ️ This issue requires manual refactoring. See suggestion.` |

## Common Refactoring Patterns

### Pattern 1: Flaky → Fake Timers

```typescript
// ❌ Before
await new Promise(r => setTimeout(r, 1000));
expect(result).toBe(expected);

// ✅ After
jest.useFakeTimers();
act(() => jest.advanceTimersByTime(1000));
expect(result).toBe(expected);
jest.useRealTimers();
```

### Pattern 2: Fragile → Test Behavior

```typescript
// ❌ Before — tests implementation
expect(hook.internalState.count).toBe(3);

// ✅ After — tests behavior
expect(result.current.displayCount).toBe("3 items");
```

### Pattern 3: Obscure → Clear Naming + AAA

```typescript
// ❌ Before
it("works", () => {
  const x = setup();
  x.do();
  expect(x.y).toBe(1);
});

// ✅ After
it("should increment counter when increment is called", () => {
  // Arrange
  const counter = createCounter();
  
  // Act
  counter.increment();
  
  // Assert
  expect(counter.value).toBe(1);
});
```

### Pattern 4: Coupled → Isolated Setup

```typescript
// ❌ Before — shared state
let store: Store;

it("test 1", () => { store.add(item); });
it("test 2", () => { expect(store.items).toHaveLength(1); }); // Depends on test 1!

// ✅ After — fresh per test
let store: Store;

beforeEach(() => {
  store = createStore(); // Fresh each time
});

afterEach(() => {
  store.reset();
});

it("test 1", () => { 
  store.add(item); 
  expect(store.items).toHaveLength(1);
});

it("test 2", () => { 
  expect(store.items).toHaveLength(0); // Starts empty
});
```

### Pattern 5: Over-mocked → Integration Test

```typescript
// ❌ Before — everything mocked
jest.mock("../api");
jest.mock("../storage");
jest.mock("../logger");
jest.mock("../analytics");

it("should sync data", () => {
  // Testing nothing real
});

// ✅ After — integration with real dependencies
it("should sync data", async () => {
  // Arrange — only mock external boundary
  const apiStub = new ApiStub().withSyncSuccess();
  const storage = new InMemoryStorage();
  const syncService = new SyncService(apiStub, storage);
  
  // Act
  await syncService.sync();
  
  // Assert — real behavior
  expect(storage.get("lastSync")).toBeDefined();
});
```

### Pattern 6: Slow → Targeted Setup

```typescript
// ❌ Before — heavy setup for every test
beforeEach(async () => {
  await seedDatabase();
  await startServer();
  await createTestUser();
});

// ✅ After — minimal setup per test
const userBuilder = () => ({ id: "1", name: "Test" });

beforeEach(() => {
  jest.clearAllMocks();
});

it("should display user name", () => {
  const user = userBuilder();
  render(<UserCard user={user} />);
  expect(screen.getByText("Test")).toBeTruthy();
});
```

## Examples

### Example 1: Check only

```
> test-refactor src/modules/auth --check-only

🔍 Test Quality Report: src/modules/auth

Issues found: 8

  🔴 HIGH:   2 (FLAKY, COUPLED)
  🟡 MEDIUM: 4 (FRAGILE, OVER-MOCKED)
  🟢 LOW:    2 (OBSCURE)

Run without --check-only to fix.
```

### Example 2: Auto-fix safe issues

```
> test-refactor src/modules/auth --auto-fix

🔍 Analyzing tests...

Found 8 issues.
Auto-fixing 3 safe issues...

✅ Added AAA comments to useAuth.test.ts
✅ Added afterEach cleanup to useLogin.test.ts  
✅ Added jest.clearAllMocks() to userStore.test.ts

5 issues require manual review:
  - 2 FLAKY (run interactively to fix)
  - 2 FRAGILE (needs design decision)
  - 1 OVER-MOCKED (consider integration test)
```

### Example 3: Focus on category

```
> test-refactor src/modules/auth --category flaky

🔍 Scanning for FLAKY issues only...

Found 3 FLAKY issues:

1. useAuth.test.ts:42 — setTimeout without fake timers
2. useAuth.test.ts:78 — Date.now() not mocked
3. useLogin.test.ts:15 — Random value in assertion

Apply fixes? [Y/n]: y

✅ All 3 flaky issues fixed!
🧪 Verified: Tests pass 5/5 times
```

## Notes

- Always load conventions skill first
- HIGH priority issues should block PRs
- `--auto-fix` is conservative (only safe fixes)
- Some issues need design decisions → flag for discussion
- Run tests after each fix to catch regressions
- Consider test speed as part of quality
