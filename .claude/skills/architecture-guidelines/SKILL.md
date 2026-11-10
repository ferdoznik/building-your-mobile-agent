---
name: architecture-guidelines
description: "The mobile team's architecture rules — MVVM layering, the networking stack, the repository pattern, and dependency injection — summarized from the ADRs in docs/adr/, with guideline names to cite. Load when reviewing architecture in Swift or Kotlin code (views, view models, repositories, API clients, navigation, DI)."
---

## How to use

Apply these guidelines only to lines a PR adds or changes. Cite each finding by the guideline's
exact name, for example `Guideline: Repository access (ADR-003)`. Each guideline comes from an
accepted ADR in `docs/adr/`; read the ADR when you need the full context.

The ADRs are written with iOS names (`ViewController`, `UIKit`, `URLSession`). Apply the same rule
to the Android equivalent: `Activity`/`Fragment`/Composable for the View, `androidx.lifecycle.ViewModel`
for the ViewModel, and the app's `ApiClient`/HTTP client for `APIClient`/`URLSession`.

## Guidelines

### MVVM layering (ADR-001)

- New screens use MVVM: a View plus a ViewModel.
- `[HIGH]` The View (`ViewController`, SwiftUI `View`, `Activity`, `Fragment`, Composable) contains
  business logic, networking, or data access. It owns only layout and user input.
- `[HIGH]` A ViewModel imports UIKit (Swift) or holds Android UI types (`View`, `Activity`,
  `Fragment`). ViewModels expose display-ready state only.
- `[LOW]` A new screen ships without its own ViewModel, with presentation logic left in the View.

### Networking stack (ADR-002)

- `[HIGH]` Any type other than `APIClient` constructs a `URLRequest`, calls `URLSession`, or (on
  Android) builds raw HTTP requests. `APIClient` is the only type that performs requests.
- `[LOW]` New request APIs use completion handlers or Combine instead of `async throws`
  (Kotlin: callbacks instead of `suspend` functions).

### Repository access (ADR-003)

- `[HIGH]` A ViewModel imports or references `APIClient`/`ApiClient`, or calls the network
  directly. ViewModels get data only through a repository.
- `[HIGH]` A type other than a repository depends on `APIClient`.
- `[LOW]` A ViewModel depends on a concrete repository class instead of the domain's repository
  protocol/interface (for example `UserRepository`).
- `[LOW]` An endpoint is decoded into its own model outside the repository that owns it, instead
  of reusing the repository's canonical model.

### Dependency injection (from ADR-003)

- `[HIGH]` A ViewModel creates its own repository or `APIClient` (for example
  `let repo = UserRepositoryImpl()` inside the ViewModel). Dependencies are passed in through the
  initializer/constructor as protocols or interfaces.
- `[LOW]` A dependency is reached through a global singleton (`.shared`, an `object`) where it
  could be injected.

### Navigation

No ADR records a navigation decision yet. Do not report navigation findings; if a change looks
questionable, raise it as a question for the author.

## Template

Compliant (Swift):

```swift
protocol RideRepository {
    func recentRides() async throws -> [Ride]
}

@MainActor
final class RideHistoryViewModel: ObservableObject {
    @Published private(set) var rides: [Ride] = []
    private let repository: RideRepository          // injected protocol

    init(repository: RideRepository) {
        self.repository = repository
    }

    func load() async {
        rides = (try? await repository.recentRides()) ?? []
    }
}
```

Non-compliant (Kotlin):

```kotlin
class TripViewModel(
    private val apiClient: ApiClient,                 // [HIGH] Repository access (ADR-003)
) : ViewModel() {
    fun loadTrips(userId: String) {
        viewModelScope.launch {
            _trips.value = apiClient.get("/users/$userId/trips")   // [HIGH] Repository access (ADR-003)
        }
    }
}
```
