# Class and Struct Design

Classes and structs are primary units of ownership. A well-designed type has a coherent responsibility, a name that reflects what it owns, and an interface smaller than the implementation it hides. A poorly designed type either accumulates unrelated responsibilities or is split into shallow wrappers that expose rather than contain complexity.

---

## Coherent Ownership

**A type should own one coherent body of knowledge or lifecycle.** It can contain several steps and methods when keeping them together hides decisions from callers. Split when parts have independent owners, invariants, consumers, or reasons to change—not merely because a description contains “and.”

The practical test: can callers understand the type through a small, stable interface without knowing its internal sequence?

- ✅ `TokenRefresher` — "Refreshes expired authentication tokens"
- ✅ `RequestThrottler` — "Throttles outgoing requests to stay within API rate limits"
- ❌ `UserManager` — "Manages user authentication, profile updates, session state, and avatar uploads"

The last example lists unrelated policy areas and is a likely split. A coordinator that owns one workflow may legitimately call several collaborators; do not replace it with a chain of one-method forwarding types.

---

## The Name Test

The name is the first contract a type makes with the reader. Names that feel uncomfortable to write are diagnostic signals:

### Red Flag Names

| Name pattern | Signal | Fix |
|-------------|--------|-----|
| `XxxManager` | Ownership may be unclear | Keep for an established lifecycle coordinator; otherwise name the owned role |
| `XxxHelper` | Operation or subject may be unclear | Name the operation or subject when possible |
| `XxxUtil` / `XxxUtils` | Bag of unrelated functions | Group by domain: `StringEncoding`, `ImageProcessing` |
| `XxxService` (too broad) | Acceptable but often too vague | Be specific: `AuthenticationService` vs just `UserService` |
| `XxxController` (outside MVC) | Fine in UIKit MVC, vague elsewhere | `CheckoutCoordinator`, `FormValidator` |
| `XxxBase` | Premature inheritance hierarchy | Prefer protocols and composition |
| `XxxCommon` | "Common to what?" | Name the concept that makes it common |
| `XxxData` / `XxxInfo` | No behavioral identity | What kind of data? `UserProfile`, `SessionCredentials` |

### Specific Names Scale Better

Specific names have a natural "pull force" — they attract only what belongs, and resist additions that don't fit. `TokenRefresher` will never feel like the right place to add profile picture logic. `UserManager` has no such resistance.

---

## Swift: Class vs Struct

This is a design decision, not just a performance consideration.

### Use `struct` when:

- The type represents a **value** — something defined by its contents, not its identity
- Copying the type creates an independent, interchangeable copy
- The type has no lifecycle (no init/deinit side effects that must be balanced)
- The type is inherently equatable by its contents

```swift
struct Coordinate {
    let latitude: Double
    let longitude: Double
}

struct Money {
    let amount: Decimal
    let currency: Currency
}

struct PriceBreakdown {
    let subtotal: Decimal
    let tax: Decimal
    let discount: Decimal
    var total: Decimal { subtotal + tax - discount }
}
```

Copying a `Money(amount: 10, currency: .usd)` and modifying the copy does not affect the original. This is the correct behavior — there is no "shared money object."

### Use `class` when:

- The type represents an **identity** — something where two references might point to the same instance and you need to observe mutations
- The type has a lifecycle that must be managed (connections, resources, subscriptions)
- You need reference sharing (two parts of the system holding the same mutable object)
- The type conforms to a protocol requiring `AnyObject`

```swift
class UserSession {
    var currentUser: User?
    var authToken: String?
    // Multiple parts of the app share one session
}

class DatabaseConnection {
    private let connection: OpaquePointer
    deinit { sqlite3_close(connection) }  // lifecycle matters
}

@Observable
class DocumentListViewModel {
    var documents: [Document] = []
    var isLoading = false
    // SwiftUI observes mutations on this shared reference
}
```

### The `@Observable` consideration

In modern Swift (iOS 17+, Swift 5.9+), `@Observable` requires `class`. This is a valid reason to use class for ViewModels and other state containers in SwiftUI:

```swift
@Observable
class FeedViewModel {
    var documents: [Document] = []
    var isRefreshing = false
    var errorMessage: String?
}
```

---

## Coupling and Cohesion

### Low Coupling

A type should communicate with as few other types as possible and pass minimal information across those connections. Each dependency is a reason a type might need to change for reasons outside its responsibility.

```swift
// High coupling — depends on many concrete types
class CheckoutViewController: UIViewController {
    let userService: UserService
    let paymentService: PaymentService
    let inventoryService: InventoryService
    let analyticsService: AnalyticsService
    let notificationService: NotificationService
    let emailService: EmailService
    // ...6 dependencies
}

// Lower coupling — depends on a single coordinator
class CheckoutViewController: UIViewController {
    let viewModel: CheckoutViewModel  // ViewModel handles all coordination
}
```

Reduce coupling by:
- passing only the values or capabilities the type actually needs;
- keeping a concrete dependency when behavior does not vary;
- introducing a small consumer-owned protocol when a real substitute exists;
- grouping coordination only when it hides a stable workflow from callers.

### High Cohesion

Everything inside a type should be strongly related to its single responsibility. Methods that don't use `self`, properties that are only used by one method, functionality that belongs to a different domain — these are cohesion failures.

```swift
// Low cohesion — string formatting method on a data type
struct Order {
    let id: String
    let items: [OrderItem]
    let total: Decimal

    // This doesn't belong here — it's a presentation concern
    func formattedReceiptHTML() -> String { ... }
}

// High cohesion — each type does its own thing
struct Order {
    let id: String
    let items: [OrderItem]
    let total: Decimal
}

class OrderReceiptFormatter {
    func formatHTML(for order: Order) -> String { ... }
}
```

---

## Accumulation: The Most Common Design Failure

Types rarely start with too many responsibilities. They accumulate them. A class that starts as `AuthService` gains password reset logic ("it's auth-related"), then account deletion ("users request it in auth flows"), then rate limiting ("it's part of security").

Each addition feels locally justified. The result is a class with four responsibilities that none of them fit cleanly.

### Signs of accumulation

- The class has methods that don't use many of the same properties as each other
- The class imports many unrelated frameworks
- The class is hard to test because each test requires setting up unrelated state
- PRs that "should be simple" always touch this class
- New team members find the class confusing despite reading it carefully

### The split decision

When splitting, organize by **reason to change**, not by type of code:

```swift
// Don't split by type of operation
class AuthService_Validation { }    // wrong axis
class AuthService_Persistence { }   // wrong axis

// Split by reason to change
class LoginService { }              // changes when login flow changes
class TokenStore { }                // changes when token storage policy changes
class PasswordResetService { }      // changes when password reset UX changes
```

---

## Protocol-Oriented Design in Swift

Protocols define capability contracts. Use one when a consumer needs a stable subset of behavior with real substitutes. A protocol is not automatically clearer than a concrete type; every requirement expands all conformers and fakes.

```swift
// Protocol defines what this type needs from the outside world
protocol OrderRepository {
    func fetch(id: String) async throws -> Order
    func save(_ order: Order) async throws
}

// Real substitutes justify the capability boundary
class RemoteOrderRepository: OrderRepository { ... }
class InMemoryOrderRepository: OrderRepository { }  // for tests

// The class that uses it depends only on the protocol
class CheckoutViewModel {
    private let repository: any OrderRepository   // not the concrete type

    init(repository: any OrderRepository) {
        self.repository = repository
    }
}
```

This pattern makes testing natural because the alternate implementation has coherent behavior. For one local operation, a closure may be smaller; with no variation, use the concrete type.

---

## Design Checklist

| Question | Pass | Fail |
|----------|------|------|
| Does the type own one coherent body of knowledge or lifecycle? | Yes | Unrelated invariants and consumers accumulate together |
| Does the name reveal that ownership in repository vocabulary? | Yes | The name forces readers to inspect implementation |
| Does the type use `struct` for values and `class` for identities? | Yes | Classes used for pure data containers |
| Do all methods use properties of the type? | Yes | Methods that don't touch `self` (free functions belong elsewhere) |
| Does each dependency represent data or behavior the type genuinely needs? | Yes | Dependencies exist only for wiring or tests |
| Would a split create deeper modules rather than pass-through wrappers? | Yes | The split only adds forwarding files and call hops |
| Is the type easy to test without setting up unrelated state? | Yes | Complex setup required for simple tests |
| Has the type grown by accumulation rather than design? | No | Each new "related feature" landed here |
