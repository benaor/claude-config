# Code Review Checklist

## Layer Compliance

### Domain Layer

- [ ] No SwiftUI/UIKit imports
- [ ] No concrete infrastructure (UserDefaults, URLSession, etc.)
- [ ] Only Foundation and pure Swift
- [ ] Entities are value types (struct) when possible
- [ ] Errors conform to `Error` and `LocalizedError`

### App Layer

- [ ] Views only depend on ViewModels
- [ ] ViewModels use UseCases, not Adapters directly
- [ ] UI-specific adapters are in `{app}/Adapters/`
- [ ] No business logic in Views

## Bounded Context

- [ ] Entity belongs to correct context
- [ ] UseCase is in same context as primary entity
- [ ] Port serves single context (or explicitly shared)
- [ ] No cross-context entity references (use IDs instead)

## Ports & Adapters

- [ ] Protocol defined in `domain/{Context}/Ports/`
- [ ] Protocol name ends with `Protocol`
- [ ] Adapter implements exactly one Port
- [ ] Adapter placement correct (shared vs UI-specific)
- [ ] No business logic in Adapters (only translation/mapping)

## Error Handling

- [ ] Custom error type per bounded context
- [ ] Error conforms to `LocalizedError`
- [ ] `errorDescription` provided for all cases
- [ ] Underlying errors wrapped when relevant
- [ ] `throws` used for simple propagation
- [ ] `Result` used for complex branching

## UseCases

- [ ] Single responsibility (one action)
- [ ] Dependencies injected via init
- [ ] Uses Ports, not concrete Adapters
- [ ] Input/output types clearly defined
- [ ] Async operations use `async/await`

## ViewModels

- [ ] Conforms to `ObservableObject`
- [ ] State properties are `@Published`
- [ ] UI updates on `@MainActor`
- [ ] Error state exposed for UI handling
- [ ] Loading state for async operations
- [ ] No direct infrastructure access
- [ ] Receives **Ports** via init (not UseCases)
- [ ] Creates **UseCases** internally when needed
- [ ] Uses **Port direct** for simple reads (no passe-plat UseCase)
- [ ] **No DIContainer reference**

## Composition Roots

### App (`{App}App.swift`)

- [ ] Configures `DIContainer.shared.configureServices()`
- [ ] Resolves all Ports via `container.resolve()`
- [ ] Creates ViewModels with `_viewModel = StateObject(wrappedValue: ...)`
- [ ] Injects ViewModels via `.environmentObject()`
- [ ] Uses `fatalError` for DI resolution failures

### Extension (`{Extension}.swift`)

- [ ] **No DIContainer** - manual instantiation
- [ ] Uses extension-safe implementations (AppleLog, not Sentry)
- [ ] Creates UseCases directly
- [ ] Calls UseCases from trigger functions

## Dependency Injection

- [ ] All dependencies registered in `ServiceRegistration.swift`
- [ ] Registration order respects dependencies
- [ ] Protocol registered, not concrete type
- [ ] Build-specific implementations use `#if`

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Protocol | `XxxProtocol` | `SessionRepositoryProtocol` |
| Adapter | Descriptive name | `UserDefaultsSessionRepository` |
| UseCase | `XxxUseCase` | `StartSessionUseCase` |
| ViewModel | `XxxViewModel` | `SessionViewModel` |
| Entity | Simple noun | `Session` |
| Error | `XxxError` | `SessionError` |

## File Organization

- [ ] One type per file
- [ ] File name matches type name
- [ ] File in correct directory per layer
- [ ] No circular dependencies between files

## Async/Combine

- [ ] `async/await` for operations
- [ ] Combine only for UI binding (`@Published`)
- [ ] `Task` properly cancelled when needed
- [ ] `@MainActor` for UI updates
- [ ] `cancellables` stored and cleaned up

## Common Issues to Flag

### 🚫 Domain importing SwiftUI

```swift
// BAD - in domain/
import SwiftUI

struct Session {
    var color: Color  // SwiftUI type in domain!
}
```

```swift
// GOOD - in domain/
struct Session {
    var colorHex: String
}

// In app layer, convert to Color
```

### 🚫 ViewModel using Adapter directly

```swift
// BAD
class SessionViewModel {
    private let repository = UserDefaultsSessionRepository()
}
```

```swift
// GOOD - ViewModel receives Port (protocol)
class SessionViewModel {
    private let repository: SessionRepositoryProtocol
    
    init(repository: SessionRepositoryProtocol) {
        self.repository = repository
    }
}
```

### 🚫 DIContainer in ViewModel

```swift
// BAD - ViewModel depends on DI infrastructure
class SessionViewModel: ObservableObject {
    private let repository: SessionRepositoryProtocol
    
    init() {
        self.repository = try! DIContainer.shared.resolve(SessionRepositoryProtocol.self)
    }
}
```

```swift
// GOOD - ViewModel receives Ports, App handles DI
class SessionViewModel: ObservableObject {
    private let repository: SessionRepositoryProtocol
    
    init(repository: SessionRepositoryProtocol) {
        self.repository = repository
    }
}

// App composition root handles DI
@main
struct retimeApp: App {
    @StateObject var sessionViewModel: SessionViewModel
    
    init() {
        let repo = try! DIContainer.shared.resolve(SessionRepositoryProtocol.self)
        _sessionViewModel = StateObject(wrappedValue: SessionViewModel(repository: repo))
    }
}
```

### 🚫 UseCase as passe-plat (pass-through)

```swift
// BAD - UseCase just wraps repository call
class GetAllGroupsUseCase {
    private let repository: GroupRepositoryProtocol
    
    func execute() async throws -> [Group] {
        return try await repository.getAll()  // No logic, just pass-through
    }
}

// In ViewModel
func loadGroups() async {
    groups = try await getAllGroupsUseCase.execute()
}
```

```swift
// GOOD - Call Port directly for simple reads
class GroupViewModel {
    private let repository: GroupRepositoryProtocol
    
    func loadGroups() async {
        groups = (try? await repository.getAll()) ?? []
    }
}
```

### 🚫 ViewModel receiving UseCases instead of Ports

```swift
// BAD - ViewModel receives UseCases
class SessionViewModel {
    init(
        startSessionUseCase: StartSessionUseCase,
        stopSessionUseCase: StopSessionUseCase
    ) { ... }
}

// Caller must create UseCases
let useCase1 = StartSessionUseCase(repository: repo, shieldManager: shield)
let useCase2 = StopSessionUseCase(repository: repo, shieldManager: shield)
let vm = SessionViewModel(startSessionUseCase: useCase1, stopSessionUseCase: useCase2)
```

```swift
// GOOD - ViewModel receives Ports, creates UseCases internally
class SessionViewModel {
    private let startSession: StartSessionUseCase
    private let stopSession: StopSessionUseCase
    
    init(
        sessionRepository: SessionRepositoryProtocol,
        shieldManager: ShieldManagerProtocol
    ) {
        self.startSession = StartSessionUseCase(
            sessionRepository: sessionRepository,
            shieldManager: shieldManager
        )
        self.stopSession = StopSessionUseCase(
            sessionRepository: sessionRepository,
            shieldManager: shieldManager
        )
    }
}
```

### 🚫 Business logic in View

```swift
// BAD
struct SessionView: View {
    var body: some View {
        Button("Start") {
            if session.duration > 0 && !session.apps.isEmpty {
                // Business logic in View!
            }
        }
    }
}
```

```swift
// GOOD
struct SessionView: View {
    @StateObject var viewModel: SessionViewModel
    
    var body: some View {
        Button("Start") {
            viewModel.startSession()
        }
        .disabled(!viewModel.canStart)
    }
}
```

### 🚫 Cross-context entity reference

```swift
// BAD
struct Session {
    var group: Group  // Direct reference to another context
}
```

```swift
// GOOD
struct Session {
    var groupId: String  // Reference by ID
}
```

### 🚫 Adapter with business logic

```swift
// BAD
class UserDefaultsSessionRepository: SessionRepositoryProtocol {
    func save(_ session: Session) async throws {
        // Business rule in adapter!
        guard session.duration <= 3600 else {
            throw SessionError.durationTooLong
        }
        // Save...
    }
}
```

```swift
// GOOD - Validation in UseCase, adapter only persists
class StartSessionUseCase {
    func execute(config: SessionConfig) async throws -> Session {
        guard config.duration <= 3600 else {
            throw SessionError.durationTooLong
        }
        let session = Session(config: config)
        try await repository.save(session)
        return session
    }
}
```
