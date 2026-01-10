# Characterization Tests

Generate characterization tests to capture existing behavior before refactoring legacy code.

## What is a Characterization Test?

A test that **documents current behavior**, not expected behavior. It creates a safety net for refactoring by capturing what the code actually does — even if that behavior is buggy.

## When to Use

- Before refactoring legacy code
- When you don't know what the code does
- When there are no existing tests
- To create a safety net before making changes

## Process

1. **Write a test that executes the code** with real-ish inputs
2. **Add a failing assertion** (assert something obviously wrong)
3. **Run only this test** with `--testNamePattern` or file path to stay token efficient
4. **Parse the failure output** to extract actual values
5. **Update assertions** with real values
6. **Run again** to confirm green

## Example Flow

### Example A: Testing a Use Case (Core Layer)

```typescript
// Step 1-2: Write test with wrong assertion
describe("PublishArticleUseCase", () => {
  it("should characterize publish article behavior", async () => {
    // Arrange
    const articleRepository = new ArticleRepositoryStub().withArticle(
      articleBuilder().id("123").status("draft").build()
    );
    const notificationService = new NotificationServiceSpy();
    const useCase = new PublishArticleUseCase(
      articleRepository,
      notificationService
    );

    // Act
    const result = await useCase.execute({ articleId: "123" });

    // Assert - Wrong on purpose
    expect(result.success).toBe(false);
  });
});
```

```bash
# Step 3: Run only this test
jest --testNamePattern="should characterize publish article behavior"

# Output: Expected: false, Received: true
```

```typescript
// Step 4-5: Parse output, update with actual value
expect(result.success).toBe(true);
if (result.success) {
  expect(result.data.status).toBe("published");
}
```

```bash
# Step 6: Run again — should be green
jest --testNamePattern="should characterize publish article behavior"
```

### 2. **Side effects via spies** — stable

Capture service calls and interactions:

```typescript
// Characterize notification calls
const notificationService = new NotificationServiceSpy();
const useCase = new PublishArticleUseCase(
  articleRepository,
  notificationService
);

await useCase.execute({ articleId: "123" });

expect(notificationService.sendEmailCallCount()).toBe(999); // Wrong

// After failure: Expected: 999, Received: 1
expect(notificationService.sendEmailCallCount()).toBe(1);
expect(notificationService.lastEmailRecipient()).toBe("author@example.com");
```

### 3. **API responses (Infrastructure)** — stable

Characterize fetch calls and HTTP interactions:

```typescript
// Characterize fetch calls
const fetchSpy = jest.spyOn(global, "fetch");
const adapter = new ArticlesApiAdapter();
await adapter.publish("123");

expect(fetchSpy).toHaveBeenCalledWith(
  "https://api.example.com/articles/999/publish", // Wrong endpoint
  expect.any(Object)
);

// After failure: Expected: "/articles/999/publish", Received: "/articles/123/publish"
expect(fetchSpy).toHaveBeenCalledWith(
  "https://api.example.com/articles/123/publish",
  expect.objectContaining({
    method: "POST",
    headers: { "Content-Type": "application/json" },
  })
);
```

### 4. **Component rendering** — least stable, use sparingly

Only capture UI state if absolutely necessary:

```typescript
// Characterize UI state (only if needed)
import { render, screen } from "@testing-library/react-native";

render(<ArticleCard article={article} />);

expect(screen.getByText("Wrong Text")).toBeTruthy(); // Wrong

// After failure: TestingLibraryElementError: Unable to find element with text: Wrong Text
// Found: "Published on Jan 15, 2024"
expect(screen.getByText("Published on Jan 15, 2024")).toBeTruthy();
```

## Test Location

### Colocated with Source Files

Following the colocation pattern, place characterization tests next to the files they test:

```
modules/articles/core/usecases/
├── PublishArticle.usecase.ts
└── PublishArticle.usecase.characterization.test.ts

modules/articles/infrastructure/adapters/
├── ArticlesApi.adapter.ts
└── ArticlesApi.adapter.characterization.test.ts
```

### Naming Convention

Use the `.characterization.test.ts` suffix to distinguish from regular `.test.ts` files:

| Type                  | Pattern                                  | Example                                           |
| --------------------- | ---------------------------------------- | ------------------------------------------------- |
| Regular unit test     | `[Name].[type].test.ts`                  | `PublishArticle.usecase.test.ts`                  |
| Characterization test | `[Name].[type].characterization.test.ts` | `PublishArticle.usecase.characterization.test.ts` |

This makes it easy to:

- Identify temporary characterization tests
- Exclude them from coverage reports if needed
- Delete them after refactoring is complete

## Key Rules

- **Always use filtering** — run only the specific test, never the full suite

  ```bash
  # By test name pattern
  jest --testNamePattern="should characterize publish article behavior"

  # By file path (faster for characterization tests)
  jest src/modules/articles/core/usecases/PublishArticle.usecase.characterization.test.ts
  ```

- **Don't fix bugs** — capture current behavior, even if wrong
- **These tests are temporary** — replace with proper unit tests after refactoring
- **Capture one behavior per test** — easier to understand failures
- **Iterate until green** — keep adjusting assertions until test passes
- **Use explicit type guards** — When using Result pattern, check `result.success` before accessing `result.data`

## Jest Configuration for Characterization Tests

### Exclude from Coverage

Add to `jest.config.js`:

```javascript
module.exports = {
  collectCoverageFrom: [
    "src/**/*.{ts,tsx}",
    "!src/**/*.characterization.test.ts", // Exclude characterization tests
  ],
};
```

### Create a Script for Running Characterization Tests Only

Add to `package.json`:

```json
{
  "scripts": {
    "test:characterize": "jest --testMatch='**/*.characterization.test.ts'"
  }
}
```

## TypeScript-Specific Patterns

### Capturing Type Information

Characterization tests can document inferred types:

```typescript
it("should characterize return type structure", async () => {
  const result = await legacyFunction();

  // This will fail and show you the actual structure
  expect(result).toEqual({
    wrongField: "wrong",
  });

  // After seeing failure, update to actual structure
  expect(result).toEqual({
    id: expect.any(String),
    status: "published",
    metadata: expect.objectContaining({
      publishedAt: expect.any(String),
    }),
  });
});
```

### Using Type Assertions Safely

```typescript
// Capture actual types returned
const result = await adapter.getArticle("123");

// Wrong on purpose to see what type is actually returned
const wrongType: { id: number } = result; // Will show type error in IDE

// Fix based on error message
const correctType: { id: string; title: string; status: string } = result;
```

## After Refactoring

1. Extract business logic to Core (entities, use cases)
2. Write proper unit tests for Core layer with stubs
3. Keep only smoke tests for Infrastructure adapters
4. Delete verbose characterization tests
5. Verify coverage with `jest --coverage`

### Migration Path Example

```typescript
// Before: Characterization test
it("should characterize publish article behavior", async () => {
  const result = await useCase.execute({ articleId: "123" });
  expect(result.success).toBe(true);
  expect(result.data.status).toBe("published");
  expect(notificationService.sendEmailCallCount()).toBe(1);
});

// After refactoring → Multiple focused unit tests
describe("PublishArticleUseCase", () => {
  it("should return success when article is successfully published", async () => {
    const repository = new ArticleRepositoryStub().withPublishSuccess();
    const useCase = new PublishArticleUseCase(repository, notificationService);

    const result = await useCase.execute({ articleId: "123" });

    expect(result.success).toBe(true);
  });

  it("should send notification when article is published", async () => {
    const spy = new NotificationServiceSpy();
    const useCase = new PublishArticleUseCase(repository, spy);

    await useCase.execute({ articleId: "123" });

    expect(spy.sendEmailCallCount()).toBe(1);
  });
});
```
