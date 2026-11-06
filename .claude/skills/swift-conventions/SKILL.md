---
name: swift-conventions
description: "Swift style conventions for the iOS app: naming, concurrency (async/await, @MainActor), and UIKit/SwiftUI usage, with severity tags. Load when reviewing or writing .swift files. Do not load for .kt or other non-Swift files."
---

## Rules

Apply these rules only to lines a PR adds or changes in `.swift` files. Cite the rule name in each
finding, for example `Rule: Swift naming`.

### Swift naming — `[LOW]`

- Types (classes, structs, enums, protocols) use `UpperCamelCase`.
- Functions, properties, parameters, and enum cases use `lowerCamelCase`.
- No `snake_case`, no capitalized members (`Granted`, `RequestPermission`), no all-caps names
  (`DATA`).
- The file name matches the primary type (`NotificationsManager.swift` holds
  `NotificationsManager`).

### Swift concurrency

- `[MEDIUM]` UI state changes off the main actor: `@Published` properties, view state, or UIKit
  calls updated from a background context without `@MainActor` or `MainActor.run`.
- `[LOW]` New code uses completion handlers or `DispatchQueue.main.async` where `async/await` and
  `@MainActor` would do.

### Swift UI layer (UIKit/SwiftUI)

- `[LOW]` Views (`View`, `UIViewController`) contain networking or business logic, such as
  `URLSession` calls or request building. Move it into a view model or service.
- `[LOW]` Untyped models (`[String: Any]`) where a typed `struct` or `Codable` model fits.
- `[LOW]` Force unwraps (`!`) outside `@IBOutlet`s and tests.
- `[LOW]` A class that isn't designed for subclassing is missing `final`.

## Template

Compliant:

```swift
@MainActor
final class NotificationsManager: ObservableObject {
    @Published private(set) var isGranted = false
    private let center: UNUserNotificationCenter

    init(center: UNUserNotificationCenter = .current()) {
        self.center = center
    }

    func requestPermission() async {
        isGranted = (try? await center.requestAuthorization(options: [.alert, .badge])) ?? false
    }
}
```

Non-compliant:

```swift
class notifications_manager {                       // [LOW] Swift naming; [LOW] missing final
    var Granted = false                             // [LOW] Swift naming
    var DATA: [String: Any] = [:]                   // [LOW] Swift naming; [LOW] untyped model

    func RequestPermission() {                      // [LOW] Swift naming
        UNUserNotificationCenter.current()
            .requestAuthorization(options: [.alert]) { granted, _ in
                self.Granted = granted              // [MEDIUM] UI state off the main actor
            }                                       // [LOW] completion handler in new code
        let url = URL(string: "https://api.example.com/me")!   // [LOW] force unwrap
    }
}
```
