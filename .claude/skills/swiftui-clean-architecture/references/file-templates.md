# File Templates

Copy-paste templates for Swift/SwiftUI Clean Architecture.

## Entity

```swift
// domain/{Context}/Entities/{EntityName}.swift

import Foundation

struct {EntityName}: Identifiable, Codable, Equatable {
    let id: String
    // Properties
    
    init(id: String = UUID().uuidString) {
        self.id = id
    }
}
```

### Entity with Status

```swift
// domain/{Context}/Entities/{EntityName}.swift

import Foundation

struct {EntityName}: Identifiable, Codable, Equatable {
    let id: String
    var status: {EntityName}Status
    let createdAt: Date
    var updatedAt: Date
    
    init(
        id: String = UUID().uuidString,
        status: {EntityName}Status = .initial,
        createdAt: Date = Date(),
        updatedAt: Date = Date()
    ) {
        self.id = id
        self.status = status
        self.createdAt = createdAt
        self.updatedAt = updatedAt
    }
    
    mutating func updateStatus(_ newStatus: {EntityName}Status) {
        self.status = newStatus
        self.updatedAt = Date()
    }
}

enum {EntityName}Status: String, Codable, Equatable {
    case initial
    case active
    case completed
    case cancelled
}
```

## Error

```swift
// domain/{Context}/Errors/{Context}Error.swift

import Foundation

enum {Context}Error: Error, LocalizedError {
    case notFound
    case invalidState
    case operationFailed(reason: String)
    case storageFailed(underlying: Error)
    
    var errorDescription: String? {
        switch self {
        case .notFound:
            return "{Context} not found"
        case .invalidState:
            return "Invalid {context} state"
        case .operationFailed(let reason):
            return "Operation failed: \(reason)"
        case .storageFailed(let error):
            return "Storage failed: \(error.localizedDescription)"
        }
    }
}
```

## Port (Protocol)

```swift
// domain/{Context}/Ports/{PortName}Protocol.swift

import Foundation

protocol {PortName}Protocol {
    func get(id: String) async throws -> {EntityName}?
    func getAll() async throws -> [{EntityName}]
    func save(_ entity: {EntityName}) async throws
    func delete(id: String) async throws
}
```

### Port with Combine Publisher

```swift
// domain/{Context}/Ports/{PortName}Protocol.swift

import Foundation
import Combine

protocol {PortName}Protocol {
    var currentPublisher: AnyPublisher<{EntityName}?, Never> { get }
    
    func get(id: String) async throws -> {EntityName}?
    func save(_ entity: {EntityName}) async throws
    func observeCurrent() -> AnyPublisher<{EntityName}?, Never>
}
```

## Adapter

### Repository Adapter

```swift
// domain/Adapters/{AdapterName}.swift

import Foundation

class {AdapterName}: {PortName}Protocol {
    private let monitor: MonitorProtocol
    private let key = "{storageKey}"
    
    init(monitor: MonitorProtocol) {
        self.monitor = monitor
    }
    
    func get(id: String) async throws -> {EntityName}? {
        guard let data = UserDefaults.standard.data(forKey: "\(key)_\(id)") else {
            return nil
        }
        do {
            return try JSONDecoder().decode({EntityName}.self, from: data)
        } catch {
            monitor.log("Failed to decode {EntityName}: \(error)", level: .error)
            throw {Context}Error.storageFailed(underlying: error)
        }
    }
    
    func save(_ entity: {EntityName}) async throws {
        do {
            let data = try JSONEncoder().encode(entity)
            UserDefaults.standard.set(data, forKey: "\(key)_\(entity.id)")
            monitor.log("Saved {EntityName} \(entity.id)", level: .debug)
        } catch {
            monitor.log("Failed to save {EntityName}: \(error)", level: .error)
            throw {Context}Error.storageFailed(underlying: error)
        }
    }
    
    func delete(id: String) async throws {
        UserDefaults.standard.removeObject(forKey: "\(key)_\(id)")
        monitor.log("Deleted {EntityName} \(id)", level: .debug)
    }
}
```

### Service Adapter

```swift
// domain/Adapters/{ServiceName}.swift

import Foundation

class {ServiceName}: {PortName}Protocol {
    private let monitor: MonitorProtocol
    private let tracker: TrackerProtocol
    
    init(monitor: MonitorProtocol, tracker: TrackerProtocol) {
        self.monitor = monitor
        self.tracker = tracker
    }
    
    func execute(input: {InputType}) async throws -> {OutputType} {
        monitor.log("Executing {ServiceName} with \(input)", level: .debug)
        
        // Implementation
        
        tracker.track("{ServiceName}.executed", properties: [:])
        return result
    }
}
```

## UseCase

### Standard UseCase

```swift
// domain/{Context}/UseCases/{ActionName}UseCase.swift

import Foundation

class {ActionName}UseCase {
    private let repository: {RepositoryName}Protocol
    
    init(repository: {RepositoryName}Protocol) {
        self.repository = repository
    }
    
    func execute(input: {InputType}) async throws -> {OutputType} {
        // Validation
        guard input.isValid else {
            throw {Context}Error.invalidState
        }
        
        // Business logic
        let entity = {EntityName}(/* ... */)
        
        // Persistence
        try await repository.save(entity)
        
        return entity
    }
}
```

### UseCase with Multiple Dependencies

```swift
// domain/{Context}/UseCases/{ActionName}UseCase.swift

import Foundation

class {ActionName}UseCase {
    private let repository: {RepositoryName}Protocol
    private let service: {ServiceName}Protocol
    private let monitor: MonitorProtocol
    
    init(
        repository: {RepositoryName}Protocol,
        service: {ServiceName}Protocol,
        monitor: MonitorProtocol
    ) {
        self.repository = repository
        self.service = service
        self.monitor = monitor
    }
    
    func execute(id: String) async throws {
        monitor.log("Executing {ActionName} for \(id)", level: .debug)
        
        guard let entity = try await repository.get(id: id) else {
            throw {Context}Error.notFound
        }
        
        try await service.process(entity)
        
        var updated = entity
        updated.updateStatus(.completed)
        try await repository.save(updated)
    }
}
```

## ViewModel

### Standard ViewModel

ViewModels receive Ports, create UseCases internally when needed.

```swift
// {app}/ViewModels/{Feature}ViewModel.swift

import Foundation
import Combine

@MainActor
class {Feature}ViewModel: ObservableObject {
    // MARK: - Published State
    
    @Published var entity: {EntityName}?
    @Published var isLoading = false
    @Published var error: {Context}Error?
    
    // MARK: - Ports (injected)
    
    private let repository: {RepositoryName}Protocol
    private let monitor: MonitorProtocol
    
    // MARK: - UseCases (created internally)
    
    private let {action}UseCase: {Action}UseCase
    
    // MARK: - Init
    
    init(
        repository: {RepositoryName}Protocol,
        monitor: MonitorProtocol
    ) {
        self.repository = repository
        self.monitor = monitor
        
        // Create UseCases with injected Ports
        self.{action}UseCase = {Action}UseCase(
            repository: repository,
            monitor: monitor
        )
    }
    
    // MARK: - Actions (UseCase for business logic)
    
    func {action}() {
        isLoading = true
        error = nil
        
        Task {
            do {
                entity = try await {action}UseCase.execute()
            } catch let err as {Context}Error {
                error = err
            }
            isLoading = false
        }
    }
    
    // MARK: - Actions (Port direct for simple reads)
    
    func loadAll() async -> [{EntityName}] {
        return (try? await repository.getAll()) ?? []
    }
    
    func reset() {
        entity = nil
        error = nil
    }
}
```

### ViewModel with Form State

```swift
// {app}/ViewModels/{Feature}FormViewModel.swift

import Foundation
import Combine

@MainActor
class {Feature}FormViewModel: ObservableObject {
    // MARK: - Form State
    
    struct FormState {
        var field1: String = ""
        var field2: Int = 0
        var selectedItems: Set<String> = []
        
        var isValid: Bool {
            !field1.isEmpty && field2 > 0
        }
    }
    
    @Published var form = FormState()
    @Published var isSubmitting = false
    @Published var error: {Context}Error?
    @Published var didSubmit = false
    
    // MARK: - Ports (injected)
    
    private let repository: {RepositoryName}Protocol
    
    // MARK: - UseCases (created internally)
    
    private let createUseCase: Create{EntityName}UseCase
    
    // MARK: - Init
    
    init(repository: {RepositoryName}Protocol) {
        self.repository = repository
        self.createUseCase = Create{EntityName}UseCase(repository: repository)
    }
    
    // MARK: - Actions
    
    func submit() {
        guard form.isValid else { return }
        
        isSubmitting = true
        error = nil
        
        Task {
            do {
                _ = try await createUseCase.execute(
                    field1: form.field1,
                    field2: form.field2
                )
                didSubmit = true
            } catch let err as {Context}Error {
                error = err
            }
            isSubmitting = false
        }
    }
    
    func reset() {
        form = FormState()
        error = nil
        didSubmit = false
    }
}
```

## View

Views access ViewModels via `@EnvironmentObject` (injected from App composition root).

### Standard View

```swift
// {app}/Views/{Feature}View.swift

import SwiftUI

struct {Feature}View: View {
    @EnvironmentObject private var viewModel: {Feature}ViewModel
    
    var body: some View {
        Group {
            if viewModel.isLoading {
                ProgressView()
            } else if let error = viewModel.error {
                ErrorView(error: error, retry: viewModel.{action})
            } else if let entity = viewModel.entity {
                ContentView(entity: entity)
            } else {
                EmptyStateView()
            }
        }
        .onAppear {
            viewModel.{action}()
        }
    }
}

// MARK: - Subviews

private extension {Feature}View {
    @ViewBuilder
    func ContentView(entity: {EntityName}) -> some View {
        VStack {
            Text(entity.id)
        }
    }
    
    @ViewBuilder
    func EmptyStateView() -> some View {
        ContentUnavailableView(
            "No data",
            systemImage: "tray",
            description: Text("Nothing to display")
        )
    }
    
    @ViewBuilder
    func ErrorView(error: {Context}Error, retry: @escaping () -> Void) -> some View {
        ContentUnavailableView {
            Label("Error", systemImage: "exclamationmark.triangle")
        } description: {
            Text(error.localizedDescription)
        } actions: {
            Button("Retry", action: retry)
        }
    }
}
```

### Form View

```swift
// {app}/Views/{Feature}FormView.swift

import SwiftUI

struct {Feature}FormView: View {
    @EnvironmentObject private var viewModel: {Feature}FormViewModel
    @Environment(\.dismiss) private var dismiss
    
    var body: some View {
        NavigationStack {
            Form {
                Section("Details") {
                    TextField("Field 1", text: $viewModel.form.field1)
                    Stepper("Field 2: \(viewModel.form.field2)", value: $viewModel.form.field2)
                }
                
                if let error = viewModel.error {
                    Section {
                        Text(error.localizedDescription)
                            .foregroundStyle(.red)
                    }
                }
            }
            .navigationTitle("New {EntityName}")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Save") { viewModel.submit() }
                        .disabled(!viewModel.form.isValid || viewModel.isSubmitting)
                }
            }
            .onChange(of: viewModel.didSubmit) { _, didSubmit in
                if didSubmit { dismiss() }
            }
        }
    }
}
```

## DI Registration

```swift
// domain/DI/ServiceRegistration.swift (add to configureServices)

// Repository
register(
    {RepositoryName}Protocol.self,
    implementation: {AdapterName}(monitor: monitor)
)

// Service with multiple dependencies
let repository = try! resolve({RepositoryName}Protocol.self)
register(
    {ServiceName}Protocol.self,
    implementation: {ServiceName}(
        repository: repository,
        monitor: monitor,
        tracker: tracker
    )
)
```

## App Composition Root

```swift
// {app}/{App}App.swift

import SwiftUI
import domain

@main
struct {App}App: App {
    // ViewModels
    @StateObject var sessionViewModel: SessionViewModel
    @StateObject var groupViewModel: GroupViewModel
    
    init() {
        // Configure DI
        let container = DIContainer.shared
        container.configureServices()
        
        // Resolve Ports
        do {
            let sessionRepository = try container.resolve(SessionRepositoryProtocol.self)
            let groupRepository = try container.resolve(GroupRepositoryProtocol.self)
            let shieldManager = try container.resolve(ShieldManagerProtocol.self)
            let monitor = try container.resolve(MonitorProtocol.self)
            let tracker = try container.resolve(TrackerProtocol.self)
            
            // Create ViewModels with Ports
            _sessionViewModel = StateObject(
                wrappedValue: SessionViewModel(
                    sessionRepository: sessionRepository,
                    shieldManager: shieldManager,
                    monitor: monitor,
                    tracker: tracker
                )
            )
            
            _groupViewModel = StateObject(
                wrappedValue: GroupViewModel(
                    groupRepository: groupRepository,
                    monitor: monitor,
                    tracker: tracker
                )
            )
        } catch {
            fatalError("Failed to configure dependencies: \(error)")
        }
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(sessionViewModel)
                .environmentObject(groupViewModel)
        }
    }
}
```

## Extension Composition Root

Extensions run in separate processes and cannot use `DIContainer`. Instantiate dependencies manually.

```swift
// {extension}/{Extension}.swift

import DeviceActivity
import Foundation
import domain

class {Extension}Extension: DeviceActivityMonitor {
    // Ports (instantiated manually)
    private let sessionRepository: SessionRepositoryProtocol
    private let shieldManager: ShieldManagerProtocol
    private let monitor: MonitorProtocol
    private let tracker: TrackerProtocol
    
    // UseCases
    private let startSession: StartSessionUseCase
    private let stopSession: StopSessionUseCase
    
    override init() {
        // Manual instantiation - no DIContainer in extensions
        // Use extension-safe implementations (AppleLog instead of Sentry, etc.)
        self.monitor = AppleLogMonitor()
        self.tracker = AppleLogTracker()
        self.sessionRepository = UserDefaultsSessionRepository(
            monitor: self.monitor,
            tracker: self.tracker
        )
        self.shieldManager = ShieldManager(
            monitor: self.monitor,
            tracker: self.tracker
        )
        
        // Create UseCases
        self.startSession = StartSessionUseCase(
            sessionRepository: self.sessionRepository,
            shieldManager: self.shieldManager,
            tracker: self.tracker,
            monitor: self.monitor
        )
        self.stopSession = StopSessionUseCase(
            sessionRepository: self.sessionRepository,
            shieldManager: self.shieldManager,
            tracker: self.tracker,
            monitor: self.monitor
        )
        
        super.init()
        
        monitor.addBreadcrumb("{Extension} initialized", category: "extension_lifecycle")
    }
    
    override func intervalDidStart(for activity: DeviceActivityName) {
        super.intervalDidStart(for: activity)
        
        let sessionId = extractSessionId(from: activity)
        
        monitor.addBreadcrumb("Interval started", category: "extension_lifecycle")
        tracker.trackEvent("extension_interval_started", properties: ["sessionId": sessionId])
        
        Task {
            _ = await startSession.execute(sessionId: sessionId)
        }
    }
    
    override func intervalDidEnd(for activity: DeviceActivityName) {
        super.intervalDidEnd(for: activity)
        
        let sessionId = extractSessionId(from: activity)
        
        monitor.addBreadcrumb("Interval ended", category: "extension_lifecycle")
        tracker.trackEvent("extension_interval_ended", properties: ["sessionId": sessionId])
        
        Task {
            _ = await stopSession.execute(sessionId: sessionId)
        }
    }
    
    private func extractSessionId(from activity: DeviceActivityName) -> String {
        // Implementation
        return activity.rawValue
    }
}
```
