---
name: swiftui-ios-architecture
description: Use when designing or implementing SwiftUI iOS screens, features, or app structure — new apps, new screens in legacy apps, or refactoring existing screens to follow MVVM + Clean Architecture with UseCases and Repositories.
---

# SwiftUI iOS Architecture

## Overview

MVVM + Clean Architecture with three strict layers: **UI** (View + ViewModel), **Domain** (UseCase), **Data** (Repository). Data flows unidirectionally. Navigation lives outside ViewModels in a NavigationGraph.

## Layer Structure

```
View  →  ViewModel  →  UseCase  →  Repository (protocol)
  ↑____________observation/AsyncStream______________|
```

---

## UI Layer

### View

- Sends named events to ViewModel: `viewModel.onLoginTapped()`
- Observes ViewModel state via `@Observable`
- Owns local UI-only state (`@State`) — focus, animation, sheet presentation
- Receives **closures** for all navigation events as constructor parameters
- No business logic, no direct data layer access

```swift
struct LoginView: View {
    @State private var viewModel: LoginViewModel
    let onLoginSuccess: () -> Void  // ✅ navigation via closure param

    init(viewModel: LoginViewModel, onLoginSuccess: @escaping () -> Void) {
        _viewModel = State(initialValue: viewModel)
        self.onLoginSuccess = onLoginSuccess
    }

    var body: some View {
        Button("Login") { viewModel.onLoginTapped() }  // ✅ named event
        // observe viewModel.state, viewModel.emailError, etc.
    }
}
```

**DON'T:**
```swift
// ❌ Business logic in View
Button("Login") {
    if email.contains("@") { ... }
}

// ❌ Navigation inside ViewModel
viewModel.navigate(to: .profile)

// ❌ VM-to-VM dependency
LoginViewModel(appViewModel: appViewModel)
```

### ViewModel

- Receives events from View, spawns `Task` for async work
- Exposes observable state (never expose mutable state directly)
- **Never** handles navigation
- Delegates ALL domain operations to a UseCase
- Use **enum** for multi-state UI (not multiple nullable fields)
- One-time events (alerts, toasts) use `AsyncStream` — not resettable properties

```swift
@Observable
@MainActor
final class LoginViewModel {
    // ✅ Enum for explicit state (not: var isLoading + var errorMessage + var isSuccess)
    enum State { case idle, loading, failed(LoginError) }
    private(set) var state: State = .idle

    // ✅ One-time events via AsyncStream (alerts, toasts)
    private let _alerts = AsyncStream<AlertItem>.makeStream()
    var alerts: AsyncStream<AlertItem> { _alerts.stream }

    private let loginUseCase: LoginUserUseCase  // ✅ UseCase, not Repository

    init(loginUseCase: LoginUserUseCase) {
        self.loginUseCase = loginUseCase
    }

    func onLoginTapped() {
        Task {
            state = .loading
            do {
                try await loginUseCase.execute(email: email, password: password)
                state = .idle
            } catch let error as LoginError {
                state = .failed(error)
            }
        }
    }
}
```

**DON'T:**
```swift
// ❌ Multiple nullable fields instead of enum
var isLoading = false
var errorMessage: String? = nil
var isSuccess = false

// ❌ Resettable one-time event property
var showAlert = false  // View must reset this — breaks unidirectional flow

// ❌ Direct repository call (skip UseCase layer)
let user = try await userRepository.login(email:password:)

// ❌ AppViewModel or other VM as dependency
init(appViewModel: AppViewModel)
```

---

## Domain Layer

### UseCase

- Wraps **one** domain action: `LoginUser`, `LogoutUser`, `FetchUserProfile`
- Either **performs** an action OR **returns** data — not both
- Returns `AsyncStream` for reactive/live data
- Depends on Repository **protocol**
- No protocol needed on UseCase itself

```swift
// ✅ Action UseCase
final class LoginUserUseCase {
    private let authRepository: AuthRepositoryProtocol

    init(authRepository: AuthRepositoryProtocol) {
        self.authRepository = authRepository
    }

    func execute(email: String, password: String) async throws {
        try await authRepository.login(email: email, password: password)
    }
}

// ✅ Data UseCase — returns reactive stream
final class ObserveCurrentUserUseCase {
    private let userRepository: UserRepositoryProtocol

    init(userRepository: UserRepositoryProtocol) {
        self.userRepository = userRepository
    }

    func execute() -> AsyncStream<User?> {
        userRepository.currentUser
    }
}
```

**DON'T:**
```swift
// ❌ UseCase that both acts AND returns data
func execute() async throws -> User {
    let user = try await repository.login(...)
    return user  // Login is an action; observe user separately
}

// ❌ Multiple concerns in one UseCase
class AuthUseCase {
    func login() { ... }
    func logout() { ... }
    func refreshToken() { ... }
}
```

---

## Data Layer

### DTOs and Entities

- **DTO** (Data Transfer Object): models the network response shape — used only inside the Repository implementation
- **Entity**: models the database/persistence shape — used only inside the Repository implementation
- Both are mapped to **domain models** before leaving the Repository — data layer models never cross the layer boundary
- Use a dedicated **mapper** function or type to convert DTO/Entity → domain model

```swift
// ✅ DTO for network response
struct UserDTO: Decodable {
    let id: String
    let display_name: String
    let avatar_url: String?
}

// ✅ Domain model (lives in Domain layer)
struct User {
    let id: String
    let displayName: String
    let avatarURL: URL?
}

// ✅ Mapper (lives in Data layer, alongside DTO)
extension UserDTO {
    func toDomain() -> User {
        User(
            id: id,
            displayName: display_name,
            avatarURL: avatar_url.flatMap(URL.init)
        )
    }
}

// ✅ Repository maps before returning
final class UserRepository: UserRepositoryProtocol {
    func fetchProfile(id: String) async throws -> User {
        let dto: UserDTO = try await apiClient.get("/users/\(id)")
        return dto.toDomain()  // DTO never leaves this layer
    }
}
```

**DON'T:**
```swift
// ❌ DTO returned from Repository (leaks into Domain/UI)
func fetchProfile(id: String) async throws -> UserDTO { ... }

// ❌ ViewModel maps raw data
let dto = try await userRepository.fetchProfile(id: id)
let user = User(id: dto.id, displayName: dto.display_name, ...)

// ❌ Domain model conforms to Decodable (couples domain to transport format)
struct User: Decodable { ... }
```

### Repository

- Single source of truth for its domain entity
- Define as **protocol** (for mocking in tests)
- Exposes reactive data via `AsyncStream`
- Concrete implementation holds actual data sources (network, DB, cache)

```swift
// ✅ Protocol for mockability
protocol AuthRepositoryProtocol: Sendable {
    func login(email: String, password: String) async throws
    func logout() async throws
}

protocol UserRepositoryProtocol: Sendable {
    var currentUser: AsyncStream<User?> { get }
}

// ✅ Concrete implementation
final class AuthRepository: AuthRepositoryProtocol {
    private let apiClient: APIClient

    init(apiClient: APIClient) {
        self.apiClient = apiClient
    }

    func login(email: String, password: String) async throws {
        try await apiClient.post("/auth/login", body: ...)
    }
}
```

**DON'T:**
```swift
// ❌ No protocol — can't mock
final class UserRepository { ... }  // ViewModel depends directly on concrete type

// ❌ Multiple sources of truth
class ProfileViewModel {
    var user: User  // local copy — not the repository's stream
}
```

---

## Navigation

Use a `NavigationGraph` view that:
1. Owns a `NavigationPath` or destination enum
2. Constructs ViewModels (injected with dependencies)
3. Passes navigation closures into screens
4. Maps destination enum/struct to concrete Views

```swift
// ✅ Destination type with required args
enum AppDestination: Hashable {
    case profile(userId: String)
    case settings
}

// ✅ NavigationGraph owns routing
struct AppNavigationGraph: View {
    @State private var path = NavigationPath()

    // Dependencies (from DI container / service locator)
    let loginUseCase: LoginUserUseCase
    let profileUseCase: ObserveUserProfileUseCase

    var body: some View {
        NavigationStack(path: $path) {
            LoginView(
                viewModel: LoginViewModel(loginUseCase: loginUseCase),
                onLoginSuccess: { path.append(AppDestination.profile(userId: $0)) }
            )
            .navigationDestination(for: AppDestination.self) { destination in
                switch destination {
                case .profile(let userId):
                    ProfileView(
                        viewModel: ProfileViewModel(
                            userId: userId,
                            observeProfile: profileUseCase
                        ),
                        onLogout: { path.removeLast() }
                    )
                case .settings:
                    SettingsView(...)
                }
            }
        }
    }
}
```

**DON'T:**
```swift
// ❌ ViewModel triggers navigation
func onLoginSuccess() {
    appViewModel.currentScreen = .profile  // VM owns nav state
}

// ❌ Screen constructs its own ViewModel
struct ProfileView: View {
    @State private var viewModel = ProfileViewModel()  // dependencies not injected
}

// ❌ Ad-hoc navigation without central graph
struct LoginView: View {
    @State private var showProfile = false
    // ...sheet/fullScreenCover in-view
}
```

---

## Dependency Injection

- All dependencies via **constructor injection** — no property injection, no singletons accessed inline
- Use a DI framework (e.g., Factory) or a manual service locator
- NavigationGraph is the composition root for screen-level VMs
- Avoid constructing heavy objects (network clients, DB connections) more than once

```swift
// ✅ Service locator (manual DI, simple approach)
final class AppContainer {
    static let shared = AppContainer()
    lazy var apiClient = APIClient()
    lazy var authRepository: AuthRepositoryProtocol = AuthRepository(apiClient: apiClient)
    lazy var userRepository: UserRepositoryProtocol = UserRepository(apiClient: apiClient)
    lazy var loginUseCase = LoginUserUseCase(authRepository: authRepository)
}

// ✅ NavigationGraph uses container
struct AppNavigationGraph: View {
    private let container = AppContainer.shared
    // ...pass container.loginUseCase into LoginViewModel
}
```

---

## File Organization

- Each type (class, struct, enum, protocol) lives in **its own file**, named after the type
- Exception: small private helper types used only within that file may coexist in the same file
- Never put multiple public/internal types in one file

```
LoginView.swift          // LoginView only
LoginViewModel.swift     // LoginViewModel only
LoginUserUseCase.swift   // LoginUserUseCase only
AuthRepository.swift     // AuthRepository + AuthRepositoryProtocol
UserDTO.swift            // UserDTO + UserDTO.toDomain() (private to Data layer)
```

**DON'T:**
```swift
// ❌ Multiple unrelated types in one file
// AuthModels.swift
struct LoginRequest { ... }
struct LoginResponse { ... }
class AuthRepository { ... }
protocol AuthRepositoryProtocol { ... }
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| VM calls Repository directly | Add UseCase between VM and Repository |
| Navigation inside ViewModel | Move to NavigationGraph; pass closure to View |
| `LoginViewModel(appViewModel:)` | Pass only the specific UseCase(s) needed |
| `var errorMessage: String?` for multi-state | Use `enum State { case idle, loading, failed(Error) }` |
| `var showAlert = false` (view resets it) | Emit via `AsyncStream` — consumed once, no reset needed |
| Screen constructs its own VM | NavigationGraph constructs and injects VM |
| Concrete Repository dependency | Add `RepositoryProtocol`, inject protocol type |
| One UseCase for multiple domain actions | One UseCase per action (`LoginUser`, `LogoutUser` separately) |
| `@StateObject` on new screens | Use `@Observable` + `@State private var vm = VM(...)` |
| DTO/Entity returned from Repository | Map to domain model inside Repository; data layer types stay in data layer |
| Multiple public types in one file | One type per file; only small private helpers may share a file |
