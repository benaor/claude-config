# Characterization Test Generator

Generate a characterization test for the selected code and iterate until green.

## Instructions

1. Read the skill at `/skills/characterization-test/SKILL.md`
2. Analyze the selected code (use case, adapter, or function)
3. Identify the bounded context from the module structure
4. Generate the test file with **intentionally wrong assertions**
5. **Run the test** with `--testNamePattern` or file path (only this test, never full suite)
6. **Parse the failure output** to extract actual values
7. **Update the test** with actual values
8. **Run again** and repeat until green

## Step-by-Step Execution

### Step 1: Generate test with wrong assertions

```typescript
expect(result.success).toBe(false); // Wrong on purpose
expect(notificationService.sendEmailCallCount()).toBe(999); // Wrong on purpose
```

### Step 2: Run only this test

```bash
jest --testNamePattern="should characterize"
# OR by file path (faster)
jest src/modules/articles/core/usecases/PublishArticle.usecase.characterization.test.ts
```

**CRITICAL: Always use `--testNamePattern` or file path to run only the specific test. Never run the full test suite.**

### Step 3: Parse failure output

Look for patterns like:
- `Expected: false, Received: true`
- `Expected: 999, Received: 1`

### Step 4: Update assertions

Replace wrong values with actual values from the output.

### Step 5: Run again

```bash
jest --testNamePattern="should characterize"
```

### Step 6: Repeat until green

If still failing, parse new failures and update. Continue until test passes.

## Test File Templates

### Template A: Use Case (Core Layer)

```typescript
/**
 * CHARACTERIZATION TEST
 *
 * Captures CURRENT behavior for safe refactoring.
 * Delete after refactoring is complete.
 */
describe("{UseCaseName}", () => {
  it("should characterize {action} behavior", async () => {
    // Arrange
    const repository = new {Repository}Stub()
      .with{Entity}({entityBuilder}().id("123").build());
    const service = new {Service}Spy();
    const useCase = new {UseCaseName}(repository, service);

    // Act
    const result = await useCase.execute({ /* params */ });

    // Assert — will be filled with actual values
    expect(result.success).toBe(false); // Wrong on purpose
  });
});
```

### Template B: API Adapter (Infrastructure Layer)

```typescript
/**
 * CHARACTERIZATION TEST
 *
 * Captures CURRENT behavior for safe refactoring.
 * Delete after refactoring is complete.
 */
describe("{AdapterName}", () => {
  it("should characterize {endpoint} response", async () => {
    // Arrange
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      status: 200,
      json: async () => ({ /* mock response */ }),
    } as Response);

    const adapter = new {AdapterName}();

    // Act
    const result = await adapter.{method}(/* params */);

    // Assert — will be filled with actual values
    expect(result.success).toBe(false); // Wrong on purpose
  });
});
```

## Output to User

After test is green, summarize:
- What behavior was captured
- Which assertions document the current behavior
- Reminder that this is a temporary safety net
