# Test Utilities Reference

Templates and utilities for Swift testing.

## Protocol Stub Template

### Generic Stub

```swift
// {Protocol}Stub.swift
import Foundation
@testable import Domain

final class {Protocol}Stub: {Protocol}Protocol {
    
    // MARK: - Configurable Results
    
    var {method}Result: Result<{ReturnType}, {ErrorType}> = .success({defaultValue})
    
    // MARK: - Call Tracking (Spy)
    
    private(set) var {method}Calls: [{ParameterType}] = []
    var {method}CallCount: Int { {method}Calls.count }
    var {method}WasCalled: Bool { !{method}Calls.isEmpty }
    var {method}LastCall: {ParameterType}? { {method}Calls.last }
    
    // MARK: - Protocol Implementation
    
    func {method}(_ param: {ParameterType}) async -> Result<{ReturnType}, {ErrorType}> {
        {method}Calls.append(param)
        return {method}Result
    }
    
    // MARK: - Reset
    
    func reset() {
        {method}Result = .success({defaultValue})
        {method}Calls = []
    }
}
```

### Repository Stub Example

```swift
// SessionRepositoryStub.swift
import Foundation
@testable import Domain

final class SessionRepositoryStub: SessionRepositoryProtocol {
    
    // MARK: - Configurable Results
    
    var getSessionResult: Result<Session?, RepositoryError> = .success(nil)
    var saveSessionResult: Result<Void, RepositoryError> = .success(())
    var deleteSessionResult: Result<Void, RepositoryError> = .success(())
    
    // MARK: - Call Tracking
    
    private(set) var getSessionCallCount = 0
    private(set) var saveSessionCalls: [Session] = []
    private(set) var deleteSessionCalls: [UUID] = []
    
    // MARK: - Protocol Implementation
    
    func getSession() async -> Result<Session?, RepositoryError> {
        getSessionCallCount += 1
        return getSessionResult
    }
    
    func saveSession(_ session: Session) async -> Result<Void, RepositoryError> {
        saveSessionCalls.append(session)
        return saveSessionResult
    }
    
    func deleteSession(id: UUID) async -> Result<Void, RepositoryError> {
        deleteSessionCalls.append(id)
        return deleteSessionResult
    }
    
    // MARK: - Reset
    
    func reset() {
        getSessionResult = .success(nil)
        saveSessionResult = .success(())
        deleteSessionResult = .success(())
        getSessionCallCount = 0
        saveSessionCalls = []
        deleteSessionCalls = []
    }
}
```

### Service Stub Example (with closure injection)

```swift
// NetworkServiceStub.swift
import Foundation
@testable import Domain

final class NetworkServiceStub: NetworkServiceProtocol {
    
    // MARK: - Closure-based Stubbing (flexible)
    
    var fetchHandler: ((URL) async -> Result<Data, NetworkError>)?
    var postHandler: ((URL, Data) async -> Result<Data, NetworkError>)?
    
    // MARK: - Simple Result Stubbing
    
    var fetchResult: Result<Data, NetworkError> = .success(Data())
    var postResult: Result<Data, NetworkError> = .success(Data())
    
    // MARK: - Call Tracking
    
    private(set) var fetchCalls: [URL] = []
    private(set) var postCalls: [(url: URL, body: Data)] = []
    
    // MARK: - Protocol Implementation
    
    func fetch(from url: URL) async -> Result<Data, NetworkError> {
        fetchCalls.append(url)
        if let handler = fetchHandler {
            return await handler(url)
        }
        return fetchResult
    }
    
    func post(to url: URL, body: Data) async -> Result<Data, NetworkError> {
        postCalls.append((url, body))
        if let handler = postHandler {
            return await handler(url, body)
        }
        return postResult
    }
}
```

## Builder Template

### Generic Builder

```swift
// {Entity}Builder.swift
import Foundation
@testable import Domain

final class {Entity}Builder {
    
    // MARK: - Properties with Defaults
    
    private var id: UUID = UUID()
    private var createdAt: Date = Date()
    // Add all entity properties with sensible defaults
    
    // MARK: - Fluent Setters
    
    func withId(_ id: UUID) -> Self {
        self.id = id
        return self
    }
    
    func withCreatedAt(_ date: Date) -> Self {
        self.createdAt = date
        return self
    }
    
    // MARK: - Build
    
    func build() -> {Entity} {
        {Entity}(
            id: id,
            createdAt: createdAt
        )
    }
}

// MARK: - Convenience Factory Methods

extension {Entity}Builder {
    
    static func default() -> {Entity} {
        {Entity}Builder().build()
    }
    
    static func valid() -> {Entity} {
        {Entity}Builder()
            // Set valid state
            .build()
    }
    
    static func invalid() -> {Entity} {
        {Entity}Builder()
            // Set invalid state
            .build()
    }
}
```

### Session Builder Example

```swift
// SessionBuilder.swift
import Foundation
@testable import Domain

final class SessionBuilder {
    
    private var id: UUID = UUID()
    private var status: SessionStatus = .idle
    private var startDate: Date = Date()
    private var endDate: Date? = nil
    private var duration: TimeInterval = 3600
    private var settings: SessionSettings = SessionSettings()
    
    func withId(_ id: UUID) -> Self {
        self.id = id
        return self
    }
    
    func withStatus(_ status: SessionStatus) -> Self {
        self.status = status
        return self
    }
    
    func withStartDate(_ date: Date) -> Self {
        self.startDate = date
        return self
    }
    
    func withEndDate(_ date: Date?) -> Self {
        self.endDate = date
        return self
    }
    
    func withDuration(_ duration: TimeInterval) -> Self {
        self.duration = duration
        return self
    }
    
    func withSettings(_ settings: SessionSettings) -> Self {
        self.settings = settings
        return self
    }
    
    func build() -> Session {
        Session(
            id: id,
            status: status,
            startDate: startDate,
            endDate: endDate,
            duration: duration,
            settings: settings
        )
    }
}

// MARK: - Convenience

extension SessionBuilder {
    
    static func idle() -> Session {
        SessionBuilder().withStatus(.idle).build()
    }
    
    static func active() -> Session {
        SessionBuilder().withStatus(.active).build()
    }
    
    static func completed() -> Session {
        SessionBuilder()
            .withStatus(.completed)
            .withEndDate(Date())
            .build()
    }
    
    static func expired() -> Session {
        SessionBuilder()
            .withStatus(.active)
            .withStartDate(Date.distantPast)
            .withDuration(1)
            .build()
    }
}
```

## Test DIContainer

```swift
// TestDIContainer.swift
import Foundation
@testable import Domain

final class TestDIContainer {
    
    // MARK: - Pre-configured Containers
    
    /// Container with all stubs (unit testing)
    static func withStubs(
        sessionRepository: SessionRepositoryProtocol = SessionRepositoryStub(),
        monitor: MonitorProtocol = MonitorStub(),
        tracker: TrackerProtocol = TrackerStub()
    ) -> DIContainer {
        let container = DIContainer()
        container.register(SessionRepositoryProtocol.self, implementation: sessionRepository)
        container.register(MonitorProtocol.self, implementation: monitor)
        container.register(TrackerProtocol.self, implementation: tracker)
        return container
    }
    
    /// Container with real implementations (integration testing)
    static func withRealAdapters() -> DIContainer {
        let container = DIContainer()
        let monitor = MonitorStub()  // Still stub external services
        
        // Use real adapters for integration tests
        container.register(MonitorProtocol.self, implementation: monitor)
        container.register(
            SessionRepositoryProtocol.self,
            implementation: InMemorySessionRepository(monitor: monitor)
        )
        return container
    }
}

// MARK: - In-Memory Implementations for Integration Tests

final class InMemorySessionRepository: SessionRepositoryProtocol {
    
    private var sessions: [UUID: Session] = [:]
    private let monitor: MonitorProtocol
    
    init(monitor: MonitorProtocol) {
        self.monitor = monitor
    }
    
    func getSession() async -> Result<Session?, RepositoryError> {
        .success(sessions.values.first)
    }
    
    func saveSession(_ session: Session) async -> Result<Void, RepositoryError> {
        sessions[session.id] = session
        return .success(())
    }
    
    func deleteSession(id: UUID) async -> Result<Void, RepositoryError> {
        sessions.removeValue(forKey: id)
        return .success(())
    }
}
```

## XCUITest Helpers

### Base Test Case

```swift
// BaseUITestCase.swift
import XCTest

class BaseUITestCase: XCTestCase {
    
    var app: XCUIApplication!
    
    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        configureApp()
        app.launch()
    }
    
    func configureApp() {
        app.launchArguments = ["--ui-testing"]
    }
    
    override func tearDownWithError() throws {
        app = nil
    }
}

// E2E version with state reset
class BaseE2ETestCase: BaseUITestCase {
    
    override func configureApp() {
        app.launchArguments = ["--e2e-testing", "--reset-state"]
    }
}
```

### XCUIElement Extensions

```swift
// XCUIElement+Helpers.swift
import XCTest

extension XCUIElement {
    
    /// Wait for element to exist with timeout
    @discardableResult
    func waitForExistence(timeout: TimeInterval = 5) -> Bool {
        waitForExistence(timeout: timeout)
    }
    
    /// Tap after ensuring element exists
    func safeTap(timeout: TimeInterval = 5) {
        guard waitForExistence(timeout: timeout) else {
            XCTFail("Element \(self) did not appear within \(timeout)s")
            return
        }
        tap()
    }
    
    /// Type text after clearing existing content
    func clearAndType(_ text: String) {
        guard waitForExistence(timeout: 5) else { return }
        tap()
        
        // Select all and delete
        if let stringValue = value as? String, !stringValue.isEmpty {
            let deleteString = String(repeating: XCUIKeyboardKey.delete.rawValue, count: stringValue.count)
            typeText(deleteString)
        }
        
        typeText(text)
    }
    
    /// Assert element has expected label
    func assertLabel(_ expected: String, file: StaticString = #file, line: UInt = #line) {
        XCTAssertEqual(label, expected, file: file, line: line)
    }
    
    /// Assert element is enabled
    func assertEnabled(file: StaticString = #file, line: UInt = #line) {
        XCTAssertTrue(isEnabled, "Expected element to be enabled", file: file, line: line)
    }
    
    /// Assert element is disabled
    func assertDisabled(file: StaticString = #file, line: UInt = #line) {
        XCTAssertFalse(isEnabled, "Expected element to be disabled", file: file, line: line)
    }
}
```

### Page Object Base

```swift
// BasePage.swift
import XCTest

protocol Page {
    var app: XCUIApplication { get }
    init(app: XCUIApplication)
}

extension Page {
    
    /// Wait for page to be fully loaded
    @discardableResult
    func waitForPage(timeout: TimeInterval = 5) -> Self {
        // Override in subclasses to wait for specific elements
        return self
    }
    
    /// Navigate back
    func goBack() {
        app.navigationBars.buttons.element(boundBy: 0).tap()
    }
    
    /// Dismiss keyboard if visible
    func dismissKeyboard() {
        if app.keyboards.element.exists {
            app.toolbars.buttons["Done"].tap()
        }
    }
}
```

### Example Page Object

```swift
// HomePage.swift
import XCTest

struct HomePage: Page {
    let app: XCUIApplication
    
    // MARK: - Elements
    
    var startButton: XCUIElement {
        app.buttons["start-session-button"]
    }
    
    var settingsButton: XCUIElement {
        app.buttons["settings-button"]
    }
    
    var statusLabel: XCUIElement {
        app.staticTexts["session-status-label"]
    }
    
    var sessionCard: XCUIElement {
        app.otherElements["session-card"]
    }
    
    // MARK: - Actions
    
    @discardableResult
    func tapStart() -> SessionPage {
        startButton.safeTap()
        return SessionPage(app: app)
    }
    
    @discardableResult
    func tapSettings() -> SettingsPage {
        settingsButton.safeTap()
        return SettingsPage(app: app)
    }
    
    // MARK: - Assertions
    
    @discardableResult
    func assertStartButtonEnabled() -> Self {
        startButton.assertEnabled()
        return self
    }
    
    @discardableResult
    func assertStatus(_ expected: String) -> Self {
        statusLabel.assertLabel(expected)
        return self
    }
    
    @discardableResult
    func assertSessionCardVisible() -> Self {
        XCTAssertTrue(sessionCard.waitForExistence(timeout: 2))
        return self
    }
}
```

## Common Assertion Patterns

### Result Assertions

```swift
// ResultAssertions.swift
import Testing

extension Result {
    
    /// Assert result is success
    func assertSuccess(file: StaticString = #filePath, line: UInt = #line) {
        guard case .success = self else {
            Issue.record("Expected success, got \(self)", filePath: file, line: line)
            return
        }
    }
    
    /// Assert result is success with specific value
    func assertSuccess<T: Equatable>(
        _ expected: T,
        file: StaticString = #filePath,
        line: UInt = #line
    ) where Success == T {
        guard case .success(let value) = self else {
            Issue.record("Expected success, got \(self)", filePath: file, line: line)
            return
        }
        #expect(value == expected)
    }
    
    /// Assert result is failure
    func assertFailure(file: StaticString = #filePath, line: UInt = #line) {
        guard case .failure = self else {
            Issue.record("Expected failure, got \(self)", filePath: file, line: line)
            return
        }
    }
    
    /// Assert result is failure with specific error
    func assertFailure<E: Equatable>(
        _ expected: E,
        file: StaticString = #filePath,
        line: UInt = #line
    ) where Failure == E {
        guard case .failure(let error) = self else {
            Issue.record("Expected failure, got \(self)", filePath: file, line: line)
            return
        }
        #expect(error == expected)
    }
}
```

### Collection Assertions

```swift
// CollectionAssertions.swift
import Testing

func assertContains<T: Equatable>(
    _ collection: [T],
    element: T,
    file: StaticString = #filePath,
    line: UInt = #line
) {
    #expect(collection.contains(element), "Expected \(collection) to contain \(element)")
}

func assertCount<T>(
    _ collection: [T],
    expected: Int,
    file: StaticString = #filePath,
    line: UInt = #line
) {
    #expect(collection.count == expected, "Expected count \(expected), got \(collection.count)")
}

func assertEmpty<T>(
    _ collection: [T],
    file: StaticString = #filePath,
    line: UInt = #line
) {
    #expect(collection.isEmpty, "Expected empty collection, got \(collection)")
}
```

### Async Test Utilities

```swift
// AsyncTestUtilities.swift
import Testing

/// Execute async code with timeout
func withTimeout<T>(
    seconds: TimeInterval,
    operation: @escaping () async throws -> T
) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask {
            try await operation()
        }
        
        group.addTask {
            try await Task.sleep(nanoseconds: UInt64(seconds * 1_000_000_000))
            throw TimeoutError()
        }
        
        let result = try await group.next()!
        group.cancelAll()
        return result
    }
}

struct TimeoutError: Error {}

/// Wait for condition to become true
func waitFor(
    timeout: TimeInterval = 2,
    interval: TimeInterval = 0.1,
    condition: () -> Bool
) async -> Bool {
    let deadline = Date().addingTimeInterval(timeout)
    while Date() < deadline {
        if condition() { return true }
        try? await Task.sleep(nanoseconds: UInt64(interval * 1_000_000_000))
    }
    return false
}
```

## Monitor/Tracker Stubs

```swift
// MonitorStub.swift
import Foundation
@testable import Domain

final class MonitorStub: MonitorProtocol {
    
    private(set) var loggedMessages: [(level: LogLevel, message: String)] = []
    private(set) var loggedErrors: [Error] = []
    
    func log(_ level: LogLevel, _ message: String) {
        loggedMessages.append((level, message))
    }
    
    func logError(_ error: Error) {
        loggedErrors.append(error)
    }
    
    func reset() {
        loggedMessages = []
        loggedErrors = []
    }
}

// TrackerStub.swift
final class TrackerStub: TrackerProtocol {
    
    private(set) var trackedEvents: [(name: String, properties: [String: Any])] = []
    
    func track(_ event: String, properties: [String: Any]) {
        trackedEvents.append((event, properties))
    }
    
    func hasTracked(_ eventName: String) -> Bool {
        trackedEvents.contains { $0.name == eventName }
    }
    
    func reset() {
        trackedEvents = []
    }
}
```
