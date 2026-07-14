# Naming Conventions

Names are the primary medium through which code communicates. In most codebases, 70–80% of the tokens a reader encounters are custom identifiers. If those identifiers are precise and consistent, the code reads like a description of the domain. If they are vague, the reader reconstructs the author's intent on every line.

## The Fundamental Rule

**A name should answer: what does this thing do, contain, or represent?** If the reader needs to look at the implementation to answer that question, the name has failed.

---

## Functions and Methods

### Action Functions (procedures)

Functions that *do something* should start with a strong verb. The verb should be specific — avoid generic verbs like `handle`, `process`, `manage`, `deal with`.

| Generic (avoid) | Specific (prefer) |
|----------------|------------------|
| `handleUser()` | `deactivateUser()`, `promoteUser()`, `mergeUserAccounts()` |
| `processPayment()` | `chargeCard()`, `refundPayment()`, `authorizePayment()` |
| `manageSession()` | `startSession()`, `invalidateSession()`, `extendSession()` |
| `doRequest()` | `fetchLatestPosts()`, `submitOrderForm()`, `cancelSubscription()` |

**Swift:**
```swift
func deactivateUser(_ user: User) { ... }
func invalidateSession(for userId: UserID) { ... }
func submitOrderForm(_ form: OrderForm) async throws { ... }
```

**Go:**
```go
func deactivateUser(u User) error { ... }
func invalidateSession(userID string) error { ... }
```

### Query Functions (return-value functions)

Functions that *return a value* should describe what they return, not what they do to get it. Avoid the `get` prefix — it implies effort and says nothing about the value.

| Getter style (avoid) | Descriptive (prefer) |
|---------------------|---------------------|
| `getUser()` | `currentUser()`, `authenticatedUser()` |
| `getData()` | `pendingOrders()`, `cachedResponse()` |
| `getCount()` | `unreadMessageCount()`, `activeSessionCount()` |
| `getValue()` | `discountedPrice()`, `normalizedScore()` |

**Exception:** `get` is acceptable in Swift when a property can't be computed (e.g., `UIView.getFrame()` is unusual; the bare `frame` property is right). Use `get` only when the semantics of "retrieval from somewhere" need to be explicit: `fetchFromDatabase()`.

**Swift:**
```swift
var pendingOrders: [Order] { ... }         // computed property
func discountedPrice(for order: Order) -> Decimal { ... }
func cachedAvatar(for userId: UserID) -> UIImage? { ... }
```

### Async Functions

Async functions that fetch from a remote source should make that clear:

```swift
func fetchUser(id: UserID) async throws -> User { ... }
func loadImage(from url: URL) async -> UIImage? { ... }
```

Async functions that perform an operation:

```swift
func submitForm(_ form: ContactForm) async throws { ... }
func synchronizeCalendar() async throws { ... }
```

---

## Boolean Naming

Boolean names should make the true case obvious. `is`/`has`/`can`/`should` are strong defaults in Swift and TypeScript. Go commonly uses concise names such as `ok`, `done`, or `enabled` when scope makes the meaning clear. Follow language and repository idiom rather than forcing one prefix everywhere.

### Prefixes

| Prefix | Use for | Example |
|--------|---------|---------|
| `is` | State or classification | `isVisible`, `isExpired`, `isAuthenticated` |
| `has` | Possession or containment | `hasChildren`, `hasUnreadMessages`, `hasPendingChanges` |
| `can` | Capability or permission | `canSubmit`, `canEdit`, `canRetry` |
| `should` | Policy or recommendation | `shouldRefreshToken`, `shouldShowOnboarding` |

### Avoid double negatives

| Double negative (avoid) | Positive (prefer) |
|------------------------|------------------|
| `!isNotEnabled` | `isEnabled` |
| `!isDisabled` | `isEnabled` |
| `!isInvalid` | `isValid` |

Do not add a project-wide helper merely to remove one ordinary negation. `if !items.isEmpty` is idiomatic and clearer than another extension unless the positive predicate is reused and carries real domain meaning.

### Avoid ambiguous state words

`flag`, `status`, `check`, `valid`, `result` — none of these tell the reader what is being checked.

```swift
// Avoid
var valid = true
var flag = false
var status = true

// Prefer
var isFormValid = true
var hasNetworkConnection = false
var isSessionActive = true
```

---

## Class, Struct, and Enum Naming

### The Name Test

Describe the type's coherent role at its call sites. If the description becomes a list of unrelated policies, resources, or reasons to change, the type may have accumulated responsibilities. The word “and” is a prompt to inspect cohesion, not an automatic reason to split.

- ✅ `TokenRefresher` — refreshes authentication tokens
- ✅ `RequestThrottler` — throttles outgoing requests to respect rate limits
- ❌ `NetworkManager` — manages... what exactly? Authentication? Caching? Retry? Parsing?

### Red flag words

These words are useful when the repository gives them a stable meaning, but otherwise deserve a more specific alternative:

| Red flag | Why | Alternative |
|----------|-----|-------------|
| `Manager` | Can hide what lifecycle or resources it owns | Keep for established coordinators; otherwise name the owned role |
| `Helper` | Can hide the operation or subject | Name the operation or subject when that improves discovery |
| `Util` / `Utils` | Bag of unrelated functions | Group by domain: `DateFormatting`, `StringEncoding` |
| `Handler` | Clear only when the handled event/boundary is obvious | `WebhookProcessor`, `KeyboardEventResponder` |
| `Controller` | Acceptable in MVC/UIKit, but vague elsewhere | `CheckoutCoordinator`, `FormValidator` |
| `Base` | Signals premature inheritance | Prefer protocols and composition |
| `Common` | "Common to what?" | Name the shared concept |

### Enums

Enum cases should not repeat the type name:

```swift
// Avoid
enum Status {
    case statusActive
    case statusInactive
    case statusPending
}

// Prefer
enum Status {
    case active
    case inactive
    case pending
}
```

---

## Parameters

Parameters define the function's interface. They should name the *role* of the value, not its type.

```swift
// Avoid — type names, not roles
func send(_ data: Data, _ url: URL, _ bool: Bool) { }

// Prefer — roles
func send(_ payload: Data, to endpoint: URL, encrypted: Bool) { }
```

### Argument labels in Swift

Swift's external/internal parameter label split is a powerful naming tool:

```swift
// Reads naturally at the call site
func insert(_ element: Element, at index: Int) { }
func move(from source: Int, to destination: Int) { }
func convert(_ temperature: Double, from unit: TemperatureUnit, to target: TemperatureUnit) -> Double { }

// Call sites read like prose:
list.insert(item, at: 0)
tableView.move(from: 3, to: 7)
```

### Related parameters

Group parameters when they form one concept, share validation, or repeatedly travel together. Do not introduce a configuration type solely because a numeric threshold was crossed:

```swift
// Avoid
func createRequest(url: URL, method: HTTPMethod, headers: [String: String], body: Data?, timeout: TimeInterval, retryCount: Int) -> URLRequest

// Prefer
struct RequestConfiguration {
    let url: URL
    let method: HTTPMethod
    var headers: [String: String] = [:]
    var body: Data? = nil
    var timeout: TimeInterval = 30
    var retryCount: Int = 0
}

func createRequest(_ config: RequestConfiguration) -> URLRequest
```

---

## Variables and Constants

### Scope-proportional naming

Names should be more specific as their scope grows:

| Scope | Naming rule | Example |
|-------|-------------|---------|
| Loop variable | Single letter ok for indices | `for i in 0..<count` |
| Closure parameter | Short but typed | `users.filter { $0.isActive }` or `users.filter { user in user.isActive }` |
| Local variable | Descriptive | `let pendingTransactions = ...` |
| Instance property | Clear and specific | `var lastSuccessfulSyncDate: Date?` |
| Global / module-level | Fully qualified | `let defaultRequestTimeout: TimeInterval = 30` |

### Magic numbers

Named constants eliminate confusion and centralize change:

```swift
// Avoid
if retries >= 3 { ... }
DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) { ... }

// Prefer
let maxRetryCount = 3
let animationSettleDuration: TimeInterval = 0.3

if retries >= maxRetryCount { ... }
DispatchQueue.main.asyncAfter(deadline: .now() + animationSettleDuration) { ... }
```

---

## Multi-language Reference

| Concept | Swift | Go | TypeScript |
|---------|-------|-----|-----------|
| Action function | `func submitOrder()` | `func submitOrder()` | `function submitOrder()` |
| Boolean | `isEnabled: Bool` | `enabled bool`, `ok bool` | `isEnabled: boolean` |
| Async fetch | `func fetchUser() async throws` | `func fetchUser() (User, error)` | `async function fetchUser(): Promise<User>` |
| Computed value | `var activeUsers: [User]` (property) | `func activeUsers() []User` | `get activeUsers(): User[]` |
| Config struct | `struct RequestConfig` | `type RequestConfig struct` | `interface RequestConfig` |
