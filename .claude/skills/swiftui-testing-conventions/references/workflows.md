# Testing Workflows

Step-by-step workflows for writing and refactoring tests.

## Write Tests Workflow

Use this workflow when adding tests for new or existing code.

### Step 1: Identify Test Type

Determine what to test and select the appropriate type:

| Testing... | Test Type | Framework | Target |
|------------|-----------|-----------|--------|
| Entity logic, pure functions | Unit | Swift Testing | `domainTests` |
| UseCase with stubbed ports | Unit | Swift Testing | `domainTests` |
| ViewModel with stubbed dependencies | Unit | Swift Testing | `{app}Tests` |
| UseCase with real adapters | Integration | Swift Testing | `domainTests` |
| ViewModel + real repository | Integration | Swift Testing | `{app}Tests` |
| Single screen/component interaction | UI | XCUITest | `{app}UITests` |
| Multi-screen user flow | E2E | XCUITest | `{app}E2ETests` |

### Step 2: Follow Type-Specific Steps

#### Unit Test (Swift Testing)

1. **Create test file** in mirror structure:
   ```
   domain/Session/UseCases/GetSessionUseCase.swift
   → domainTests/Session/UseCases/GetSessionUseCaseTests.swift
   ```

2. **Set up test suite**:
   ```swift
   import Testing
   @testable import Domain

   @Suite("GetSessionUseCase")
   struct GetSessionUseCaseTests {
       var repository: SessionRepositoryStub
       var sut: GetSessionUseCase
       
       init() {
           repository = SessionRepositoryStub()
           sut = GetSessionUseCase(repository: repository)
       }
   }
   ```

3. **Identify behaviors to test** — list expected behaviors:
   - Happy path: returns session when exists
   - Edge case: returns nil when empty
   - Error case: propagates repository error

4. **Write first test** (start with happy path):
   ```swift
   @Test("should return session when repository has data")
   func shouldReturnSession_whenRepositoryHasData() async {
       // Given
       let expected = SessionBuilder().build()
       repository.getSessionResult = .success(expected)
       
       // When
       let result = await sut.execute()
       
       // Then
       #expect(try result.get() == expected)
   }
   ```

5. **Add remaining tests** — one test per behavior, follow naming convention `shouldX_whenY()`

6. **Run tests** — verify all pass: `Cmd+U` or `swift test`

#### Integration Test (Swift Testing)

1. **Create test file** with `Integration` suffix:
   ```
   domainTests/Session/UseCases/GetSessionUseCaseIntegrationTests.swift
   ```

2. **Set up with real adapters** (stub only external boundaries):
   ```swift
   @Suite("GetSessionUseCase Integration")
   struct GetSessionUseCaseIntegrationTests {
       var repository: InMemorySessionRepository
       var monitor: MonitorStub  // External service still stubbed
       var sut: GetSessionUseCase
       
       init() {
           monitor = MonitorStub()
           repository = InMemorySessionRepository(monitor: monitor)
           sut = GetSessionUseCase(repository: repository)
       }
   }
   ```

3. **Test real interactions**:
   ```swift
   @Test("should persist and retrieve session")
   func shouldPersistAndRetrieveSession() async throws {
       // Given
       let session = SessionBuilder().build()
       _ = await repository.saveSession(session)
       
       // When
       let result = await sut.execute()
       
       // Then
       #expect(try result.get() == session)
   }
   ```

4. **Run tests** — verify integration points work correctly

#### UI Test (XCUITest)

1. **Create test file** in `{app}UITests/Components/`:
   ```
   {app}UITests/Components/StartButtonTests.swift
   ```

2. **Create or reuse Page Object**:
   ```swift
   // Pages/HomePage.swift
   struct HomePage: Page {
       let app: XCUIApplication
       
       var startButton: XCUIElement {
           app.buttons["start-session-button"]
       }
       
       @discardableResult
       func tapStart() -> SessionPage {
           startButton.safeTap()
           return SessionPage(app: app)
       }
   }
   ```

3. **Add accessibility identifiers** in production code:
   ```swift
   Button("Start") { ... }
       .accessibilityIdentifier("start-session-button")
   ```

4. **Write UI test**:
   ```swift
   final class StartButtonTests: XCTestCase {
       var app: XCUIApplication!
       var homePage: HomePage!
       
       override func setUpWithError() throws {
           continueAfterFailure = false
           app = XCUIApplication()
           app.launchArguments = ["--ui-testing"]
           app.launch()
           homePage = HomePage(app: app)
       }
       
       func test_startButton_shouldNavigateToSession_whenTapped() {
           let sessionPage = homePage.tapStart()
           XCTAssertTrue(sessionPage.timerLabel.waitForExistence(timeout: 2))
       }
   }
   ```

5. **Run UI tests** — `Cmd+U` on UI test target

#### E2E Test (XCUITest)

1. **Create test file** in `{app}E2ETests/Flows/`:
   ```
   {app}E2ETests/Flows/SessionFlowTests.swift
   ```

2. **Define the user flow** to test:
   ```
   Start session → Wait for timer → Pause → Stop → View summary
   ```

3. **Reuse or create Page Objects** for each screen

4. **Write E2E test** with chained page actions:
   ```swift
   final class SessionFlowTests: XCTestCase {
       var app: XCUIApplication!
       
       override func setUpWithError() throws {
           continueAfterFailure = false
           app = XCUIApplication()
           app.launchArguments = ["--e2e-testing", "--reset-state"]
           app.launch()
       }
       
       func test_completeSessionFlow_shouldShowSummary() {
           HomePage(app: app)
               .tapStart()
               .waitForTimerToStart()
               .tapPause()
               .tapStop()
               .assertSummaryVisible()
       }
   }
   ```

5. **Run E2E tests** — these are slower, run separately from unit tests

### Step 3: Verify Test Quality

Before committing, check:

- [ ] Test name follows `shouldX_whenY()` convention
- [ ] Single behavior per test
- [ ] Given/When/Then clearly separated
- [ ] No test interdependence
- [ ] No hardcoded magic values
- [ ] Stubs reset properly (fresh `init()`)

---

## Refactor Test Workflow

Use this workflow to fix a specific test that has issues.

### Step 1: Identify the Problem

Common test smells and their symptoms:

| Smell | Symptom |
|-------|---------|
| **Flaky** | Passes sometimes, fails randomly |
| **Slow** | Takes > 500ms for unit test |
| **Fragile** | Breaks when implementation changes |
| **Obscure** | Hard to understand what it tests |
| **Coupled** | Fails when other tests run first |
| **Over-mocked** | Too many mocks, hard to maintain |

### Step 2: Diagnose Root Cause

#### Flaky Test

Check for:
```swift
// ❌ Race condition
sut.startAsync()
#expect(sut.value == expected)  // May not be ready

// ❌ Time-dependent
#expect(session.startDate == Date())  // Microseconds off

// ❌ Order-dependent
static var shared = 0  // Mutated across tests
```

#### Slow Test

Check for:
```swift
// ❌ Real network/disk
let repo = UserDefaultsRepository()  // Hits disk

// ❌ Unnecessary delays
try await Task.sleep(for: .seconds(2))

// ❌ Too much setup
// 50 lines of Given...
```

#### Fragile Test

Check for:
```swift
// ❌ Testing implementation details
#expect(sut.internalCache.count == 3)

// ❌ Exact string matching
#expect(error.localizedDescription == "The operation couldn't be completed.")

// ❌ Specific call order
#expect(spy.calls == ["load", "parse", "save"])  // Order matters?
```

#### Obscure Test

Check for:
```swift
// ❌ No clear structure
func testIt() async {
    let x = Thing()
    x.a = 1
    x.b = 2
    let y = await sut.do(x)
    #expect(y.c == 3)
}

// ❌ Meaningless name
func testCase1() async { ... }
```

#### Coupled Test

Check for:
```swift
// ❌ Shared mutable state
class Tests: XCTestCase {
    static var session: Session?  // Shared across tests
}

// ❌ Database/file not cleaned
// Previous test left data behind
```

#### Over-mocked Test

Check for:
```swift
// ❌ Mock explosion
let mockA = MockA()
let mockB = MockB()
let mockC = MockC()
let mockD = MockD()
let mockE = MockE()
// Every dependency mocked
```

### Step 3: Apply Fix

#### Fix Flaky Test

```swift
// ✅ Await async operations
await sut.startAsync()
#expect(sut.value == expected)

// ✅ Use fixed dates
let fixedDate = Date(timeIntervalSince1970: 1000)
let session = SessionBuilder().withStartDate(fixedDate).build()

// ✅ Fresh state per test (use struct + init)
@Suite struct Tests {
    var state: Int
    init() { state = 0 }  // Fresh each test
}
```

#### Fix Slow Test

```swift
// ✅ Use in-memory implementations
let repo = InMemorySessionRepository()

// ✅ Remove artificial delays
// Just await the actual operation

// ✅ Use builders for setup
let session = SessionBuilder().active().build()  // One line
```

#### Fix Fragile Test

```swift
// ✅ Test observable behavior
#expect(sut.items.count == 3)  // Public API

// ✅ Check error type, not message
#expect(throws: ValidationError.self) { try sut.validate() }

// ✅ Check calls happened, not order
#expect(spy.loadWasCalled)
#expect(spy.saveWasCalled)
```

#### Fix Obscure Test

```swift
// ✅ Clear Given/When/Then
@Test("should calculate total when items have prices")
func shouldCalculateTotal_whenItemsHavePrices() async {
    // Given
    let items = [
        ItemBuilder().withPrice(10).build(),
        ItemBuilder().withPrice(20).build()
    ]
    repository.itemsResult = .success(items)
    
    // When
    let total = await sut.calculateTotal()
    
    // Then
    #expect(total == 30)
}
```

#### Fix Coupled Test

```swift
// ✅ Use Swift Testing structs (automatic isolation)
@Suite struct SessionTests {
    var repository: SessionRepositoryStub
    var sut: GetSessionUseCase
    
    init() {
        // Fresh instances for EVERY test
        repository = SessionRepositoryStub()
        sut = GetSessionUseCase(repository: repository)
    }
}
```

#### Fix Over-mocked Test

```swift
// ✅ Only mock boundaries, use real objects internally
@Suite struct GetSessionUseCaseTests {
    var repository: SessionRepositoryStub  // Boundary = stubbed
    var sut: GetSessionUseCase             // Real object
    
    // Internal collaborators of UseCase = real
    // Only external dependencies = stubbed
}

// ✅ Consider integration test if too many mocks needed
```

### Step 4: Verify Fix

After refactoring:

1. **Run the test 10 times** — ensure no flakiness:
   ```bash
   for i in {1..10}; do swift test --filter TestName; done
   ```

2. **Run full test suite** — ensure no regressions

3. **Check test still fails when it should** — break the production code temporarily and verify test catches it

4. **Review test readability** — can someone else understand it in 30 seconds?
