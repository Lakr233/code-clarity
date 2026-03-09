# Abstraction Levels

Abstraction level consistency is one of the most important — and most commonly violated — principles of readable code. A function, class, or module should operate at a single, coherent level of abstraction. Mixing levels forces the reader to context-switch between strategic intent and implementation mechanics, increasing cognitive load on every read.

## What Is an Abstraction Level?

An abstraction level describes how close to the machine — or how close to the domain — a piece of code operates.

```
Domain / Business Level      "Complete the checkout process"
       ↓
Application Level            "Submit the order, then update the cart state"
       ↓
Infrastructure Level         "POST to /orders with a JSON body"
       ↓
Mechanism Level              "Encode dictionary to UTF-8 JSON bytes"
```

Each level is a valid place to write code. The mistake is writing code from two different levels in the same function.

---

## The Mixing Violation

```swift
// Mixed levels in one function — hard to read
func submitOrder(_ order: Order) async throws {
    // Application level: decide what to do
    let payload = buildPayload(from: order)

    // Mechanism level: low-level URL construction mixed in
    var components = URLComponents()
    components.scheme = "https"
    components.host = "api.example.com"
    components.path = "/v2/orders"
    components.queryItems = [URLQueryItem(name: "source", value: "ios-app")]
    guard let url = components.url else { throw NetworkError.invalidURL }

    // Back to application level
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.httpBody = try JSONEncoder().encode(payload)

    // Infrastructure level: make the call
    let (data, response) = try await URLSession.shared.data(for: request)
    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 201 else {
        throw NetworkError.unexpectedStatus
    }
    let result = try JSONDecoder().decode(OrderConfirmation.self, from: data)

    // Application level again
    handleConfirmation(result)
}
```

The reader must constantly shift their mental frame: "Am I reading business logic? Network logic? URL encoding?"

---

## The Consistent Level Approach

```swift
// Application level only — consistent
func submitOrder(_ order: Order) async throws {
    let payload = buildPayload(from: order)
    let confirmation = try await network.post(payload, to: .createOrder)
    handleConfirmation(confirmation)
}

// Infrastructure level only — consistent
func post<T: Encodable, R: Decodable>(_ body: T, to endpoint: Endpoint) async throws -> R {
    let request = try buildRequest(for: endpoint, body: body)
    let data = try await execute(request)
    return try decode(data)
}

// Mechanism level only — consistent
private func buildRequest<T: Encodable>(for endpoint: Endpoint, body: T) throws -> URLRequest {
    var components = URLComponents()
    components.scheme = endpoint.scheme
    components.host = endpoint.host
    components.path = endpoint.path
    guard let url = components.url else { throw NetworkError.invalidURL }
    var request = URLRequest(url: url)
    request.httpMethod = endpoint.method.rawValue
    request.httpBody = try JSONEncoder().encode(body)
    return request
}
```

Each function can be read and understood independently. The reader of `submitOrder` never needs to know how HTTP works.

---

## The Step-Down Rule

When reading a file top-to-bottom, functions should be organized so that each function is followed by the functions it calls at the next level down. The file tells a story from high-level to low-level.

```swift
// Top of file — highest level
func processCheckout(_ cart: Cart) async throws {
    try validateCart(cart)
    let order = try await createOrder(from: cart)
    try await confirmPayment(for: order)
    notifyUser(of: order)
}

// One level down
private func validateCart(_ cart: Cart) throws { ... }
private func createOrder(from cart: Cart) async throws -> Order { ... }
private func confirmPayment(for order: Order) async throws { ... }
private func notifyUser(of order: Order) { ... }

// Two levels down
private func validateItems(_ items: [CartItem]) throws { ... }
private func buildOrderPayload(from cart: Cart) -> OrderPayload { ... }
// ...
```

A reader who wants high-level understanding reads the top. A reader debugging a specific step scrolls to find it. The structure itself communicates the hierarchy.

---

## Identifying Level Violations

### Red flag: A function calls other named functions AND does raw computation

```swift
// Violation — named function calls mixed with inline computation
func applyDiscount(to order: Order) -> Decimal {
    let eligibleItems = filterEligibleItems(order.items)  // named call

    // inline computation mixed in — wrong level
    let subtotal = eligibleItems.reduce(Decimal.zero) { $0 + $1.price * Decimal($1.quantity) }
    let discountRate = Decimal(0.15)
    return subtotal * (1 - discountRate)
}

// Fix — extract the computation
func applyDiscount(to order: Order) -> Decimal {
    let eligibleItems = filterEligibleItems(order.items)
    return computeDiscountedTotal(for: eligibleItems)
}

private func computeDiscountedTotal(for items: [OrderItem]) -> Decimal { ... }
```

### Red flag: A view contains networking or business logic

In SwiftUI or UIKit, views are at the presentation level. Anything beyond layout, animation, and user input handling belongs elsewhere:

```swift
// Violation — view contains business logic and networking
struct ProfileView: View {
    var body: some View {
        Button("Save") {
            // Business rule: validate
            guard user.email.contains("@") else { return }

            // Infrastructure: make network call directly in view
            Task {
                let data = try? await URLSession.shared.data(from: saveURL)
                // ...
            }
        }
    }
}

// Prefer — view delegates to view model
struct ProfileView: View {
    @State private var viewModel: ProfileViewModel

    var body: some View {
        Button("Save") {
            Task { await viewModel.saveProfile() }
        }
    }
}
```

### Red flag: A repository contains presentation logic

```swift
// Violation — data layer formats for display
class UserRepository {
    func fetchDisplayName(for id: String) async -> String {
        let user = try? await api.fetchUser(id: id)
        return "\(user?.firstName ?? "") \(user?.lastName ?? "")".trimmingCharacters(in: .whitespaces)
    }
}

// Prefer — repository returns model, presentation logic lives in presentation layer
class UserRepository {
    func fetchUser(id: String) async throws -> User { ... }
}

// Display formatting in the ViewModel or a formatter type
extension User {
    var displayName: String { "\(firstName) \(lastName)".trimmingCharacters(in: .whitespaces) }
}
```

---

## Abstraction Levels in Swift Architecture

| Layer | Abstraction Level | Allowed to contain |
|-------|------------------|--------------------|
| SwiftUI View / UIViewController | Presentation | Layout, animation, user input routing |
| ViewModel / Presenter | Application | State transformation, validation rules, use case coordination |
| Repository / Service | Domain | Domain operations, business rules |
| Network / Persistence | Infrastructure | HTTP, JSON, Core Data, file I/O |
| Utilities / Extensions | Mechanism | Low-level algorithms, type conversions |

When code from one row appears in a type from another row, there is an abstraction level violation. Move the code to its correct layer.

---

## Common Violations and Fixes

| Violation | Location | Fix |
|-----------|----------|-----|
| URL construction in ViewModel | ViewModel | Extract to NetworkLayer or Endpoint enum |
| String formatting in Repository | Repository | Move to ViewModel property or Formatter type |
| Business rule in View | View | Move to ViewModel method |
| JSON parsing in ViewController | ViewController | Move to NetworkLayer or ResponseDecoder |
| Authentication token injection in UI code | Any UI layer | Move to NetworkLayer or RequestInterceptor |
| Retry logic in a ViewModel | ViewModel | Move to NetworkLayer with a RetryPolicy |

---

## Summary

| Question | Good sign | Bad sign |
|----------|-----------|---------|
| Can you describe the function's level in one word (orchestration / transformation / mechanism)? | Yes | "It does a bit of both" |
| Do all calls in the function feel like they belong at the same height? | Yes | Some are high-level, some are raw operations |
| Does the file read like a top-down hierarchy? | Yes | Functions appear in unrelated order |
| Does the view contain only presentation code? | Yes | It has service calls or validation logic |
| Does the repository contain only domain operations? | Yes | It formats strings for display |
