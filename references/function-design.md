# Function and Method Design

A function is the smallest unit of reusable thought in code. Its design determines how easily a reader can understand what the code does, how safely it can be modified, and how effectively it can be tested. The principles here apply regardless of language.

## One Coherent Contract

**A function should expose one coherent operation at its call site.** It may coordinate several internal steps when those steps always belong to the same operation and share one error/ownership boundary.

- ✅ `validateEmailFormat(_ email: String) -> Bool` — validates an email format
- ✅ `persistUser(_ user: User) throws` — persists a user to storage
- ⚠️ `validateAndSaveUser(_ user: User) throws` — unclear unless validation is an inseparable precondition of this save operation.

Words such as `and`, `then`, or `also` are diagnostic signals, not proof that a split is needed. Split when callers need the steps independently, the steps change for different reasons, or the function mixes unrelated ownership. Keep orchestration together when extraction would create pass-through functions and force every caller to reconstruct the same sequence.

---

## The One Level of Abstraction Rule

Every function should operate at a single, consistent level of abstraction. Mixing strategic orchestration with low-level implementation detail in the same function is one of the most common readability failures.

### Identifying abstraction levels

| Level | Characteristic | Example |
|-------|---------------|---------|
| High | Orchestrates other named functions, reads like a policy | `func checkout(cart: Cart) async throws` |
| Mid | Transforms or coordinates between subsystems | `func buildOrderPayload(from cart: Cart) -> OrderPayload` |
| Low | Direct manipulation, computation, string/data work | `func appendQueryItem(_ item: URLQueryItem, to components: inout URLComponents)` |

### The violation

```swift
// Mixed levels — avoid
func checkout(cart: Cart) async throws {
    // High level: orchestrating steps
    let payload = buildOrderPayload(from: cart)

    // Low level: manually building a URL here — wrong level
    var components = URLComponents(string: baseURL)!
    components.queryItems = [URLQueryItem(name: "source", value: "ios")]
    let url = components.url!

    // Back to high level
    let result = try await network.post(payload, to: url)
    handleCheckoutResult(result)
}
```

```swift
// Consistent levels — prefer
func checkout(cart: Cart) async throws {
    let payload = buildOrderPayload(from: cart)
    let endpoint = checkoutEndpoint()                   // low-level detail hidden
    let result = try await network.post(payload, to: endpoint)
    handleCheckoutResult(result)
}

private func checkoutEndpoint() -> URL { ... }         // level-appropriate
```

The reader of `checkout` should never need to think about URL construction. That knowledge belongs one level down.

---

## Function Length

Function length is a symptom, not a rule. A longer function with one coherent contract and consistent abstraction can be clearer than several tiny functions that only forward arguments.

Length can reveal problems, but no numeric threshold is meaningful across languages and domains. Re-examine a function when it mixes abstraction levels, repeats cleanup or error handling, contains branches with independent purposes, or prevents a reader from seeing the contract. Extract only when the new name hides a coherent detail or creates a reusable operation.

### The comment-as-function signal

When a comment precedes a block of code to explain what it does, that block is probably a function:

```swift
// Avoid
func processCheckout() async throws {
    // Validate cart
    guard !cart.items.isEmpty else { throw CartError.empty }
    guard cart.total > 0 else { throw CartError.invalidTotal }

    // Build payload
    let payload = CheckoutPayload(
        items: cart.items.map { PayloadItem($0) },
        total: cart.total,
        currency: cart.currency
    )

    // Submit
    try await api.submitOrder(payload)
}

// Prefer — comments become function names
func processCheckout() async throws {
    try validateCart()
    let payload = buildCheckoutPayload()
    try await api.submitOrder(payload)
}

private func validateCart() throws { ... }
private func buildCheckoutPayload() -> CheckoutPayload { ... }
```

---

## Parameters

### Group by meaning, not count

Several independent scalar parameters can be clearer than a generic configuration bag. Introduce a parameter type when the values form one domain concept, must be validated together, or repeatedly travel together. Do not create a struct solely to reduce the visible parameter count; that only moves complexity into another declaration.

### Avoid boolean flag parameters

Boolean parameters deserve scrutiny when they select substantially different algorithms or are unclear at the call site:

```swift
// Avoid — caller must know what `true` means
func render(view: UIView, animated: Bool)
render(view: headerView, animated: true)
render(view: headerView, animated: false)

// Prefer — two distinct operations
func render(view: UIView)
func renderAnimated(view: UIView)
```

A well-labeled boolean is appropriate for one small orthogonal variation, such as Swift's `animated:` parameters. Use separate functions or an enum when the alternatives have different preconditions, side effects, or payloads.

### Avoid output parameters

Output parameters (inout / passing a container to be filled) make data flow hard to trace:

```swift
// Avoid — where does result come from?
var result: [User] = []
populateUsers(&result, from: database)

// Prefer — data flows from right to left clearly
let result = fetchUsers(from: database)
```

Swift's `inout` is appropriate for performance-sensitive mutations of value types (e.g., mutating a large array in place) but not for "return multiple values" — use tuples or structs for that.

### Return values over side effects

If a function computes something, return it. Don't mutate external state unless mutation is the explicit purpose of the function:

```swift
// Side-effect-in-disguise — avoid when computation is the goal
func computeTotal() {
    self.total = items.reduce(0) { $0 + $1.price }
}

// Return the value — prefer
func computeTotal() -> Decimal {
    return items.reduce(0) { $0 + $1.price }
}

// Mutation is the explicit purpose — this is fine
func appendItem(_ item: Item) {
    items.append(item)
    invalidateTotal()
}
```

---

## Side Effects and Naming

A function that has side effects beyond what its name implies is a source of bugs. **The name must reveal all significant side effects.**

```swift
// Dangerous — name implies read, but has write side effect
func currentUser() -> User {
    let user = fetchFromAPI()   // network call!
    cache.store(user)           // mutation!
    return user
}

// Better — name reveals the full operation
func fetchAndCacheCurrentUser() async throws -> User { ... }

// Or: separate the concerns
func currentUser() -> User? { cache.user }        // pure read
func refreshCurrentUser() async throws { ... }    // explicit update
```

---

## Swift-Specific Patterns

### Computed properties vs functions

In Swift, zero-argument functions that return a derived value are often better expressed as computed properties when:
- They involve no side effects
- They are fast (no I/O, no heavy computation)
- They represent a property of the type's identity

```swift
// Functions — prefer when: async, throwing, or meaningfully expensive
func fetchProfileImage() async throws -> UIImage { ... }

// Properties — prefer when: derived, synchronous, side-effect-free
var formattedName: String { "\(firstName) \(lastName)" }
var isEmpty: Bool { items.count == 0 }
var isExpired: Bool { expiryDate < Date.now }
```

### `throws` vs optional return

Use `throws` when the failure has a meaningful reason the caller might act on. Use `Optional` when absence is the normal, expected case:

```swift
// Optional: absence is normal
var selectedItem: Item? { ... }
func item(at index: Int) -> Item? { ... }

// throws: failure has a reason worth communicating
func fetchUser(id: String) async throws -> User { ... }
func decode<T: Decodable>(_ data: Data) throws -> T { ... }
```

### `@discardableResult`

Use sparingly. If a return value is commonly ignored, ask why — usually it signals the function should be split or the API redesigned. Mark with `@discardableResult` only when ignoring the result is genuinely the common case.

---

## Quick Reference

| Principle | Check |
|-----------|-------|
| Coherent contract | Callers see one operation and one ownership boundary |
| One abstraction level | All calls in the function should be at the same "height" |
| Parameters | Group values only when they form one concept |
| Boolean flags | Keep small labeled variations; split distinct operations |
| No output parameters | Return a value instead |
| Side effects in name | If it writes/sends/logs, the name should say so |
| Comment-as-function | If you wrote a comment, write a function |
