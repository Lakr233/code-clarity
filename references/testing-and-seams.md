# Dependencies, Seams, and Testing

Testability is a design property, not a test-time afterthought. Code is easy to test when it has **one canonical implementation of each behavior**, a small number of **deliberately-designed seams** where behavior genuinely varies, and tests that run against **real implementations or shared in-memory fakes** rather than mock frameworks. This reference covers three intertwined habits: don't duplicate implementations, don't over-inject, and prefer protocol/subclass seams over closures and mocks.

The examples use a neutral, illustrative domain (documents, items, stores).

---

## 1. One Canonical Implementation — Don't Reimplement at Each Call Site

The most common duplication failure is not copy-pasted files; it is the *same behavior re-written inline wherever it is needed*. The third time you write the same date formatting, slug sanitizing, retry rule, or price calculation at a call site, you have created a maintenance trap: a fix will land in some copies and miss others, and no copy is authoritative.

```swift
// Avoid — the same formatting re-implemented at every call site
let a = "\(user.firstName) \(user.lastName)".trimmingCharacters(in: .whitespaces)
// ...elsewhere...
let b = "\(other.firstName) \(other.lastName)".trimmingCharacters(in: .whitespaces)
// ...ten more places, each slightly different over time...

// Prefer — one implementation, named, reused
extension Person {
    var displayName: String {
        "\(firstName) \(lastName)".trimmingCharacters(in: .whitespaces)
    }
}
// every call site: person.displayName
```

The test: **if you are writing logic you have written before, stop and extract it.** A behavior lives in exactly one place; call sites reference it. This is the logic-level twin of the named-constant rule in [file-organization.md](file-organization.md) — one source of truth, whether the thing being centralized is a value or a behavior.

Signs of scattered duplication:
- A bug fix has to be applied "in all the places that do X."
- Two call sites compute the same thing with subtly different code.
- A reader cannot tell which copy is the real one.

---

## 2. A Seam Earns Its Place Only When Behavior Varies

A *seam* is a point where you can substitute one implementation for another — a Swift `protocol` or overridable subclass, a TypeScript `interface`. Seams are valuable, but each one is also indirection a reader must follow. Add a seam only when behavior **genuinely varies**:

- there is a real second implementation (remote vs local, production vs OSS, platform A vs B), **or**
- a test needs a real substitute for something slow, networked, or non-deterministic.

If a type has exactly one implementation forever and no test substitute, **a direct concrete reference is clearer** than a protocol plus injection ceremony. A speculative `protocol` with a single conformer is indirection with no payoff.

```swift
// Over-abstracted — one impl, no fake, pure ceremony
protocol GreetingProvider { func greeting() -> String }
final class DefaultGreetingProvider: GreetingProvider { func greeting() -> String { "Hi" } }

// Clearer — it never varies, so don't add a seam
enum Greeting { static let text = "Hi" }
```

Introduce the protocol the moment a second real implementation or a genuine test fake appears — not before.

---

## 3. Swift: Protocol / Subclass Seams, Not Closure Properties

When you do need a seam in Swift, design it as a **protocol** (or, where inheritance fits, an overridable subclass) — not a bag of stored closures passed through the initializer. A protocol names the contract, surfaces in the type system, documents every capability in one place, and is reusable across many tests. Closure-injected dependencies hide the contract, resist reuse, and scatter behavior across call sites.

```swift
// Avoid — dependencies as anonymous closures threaded through init
final class ClosureDocumentLoader {
    private let onFetch: () async throws -> [Document]
    private let onCache: ([Document]) -> Void
    init(
        onFetch: @escaping () async throws -> [Document],
        onCache: @escaping ([Document]) -> Void
    ) {
        self.onFetch = onFetch
        self.onCache = onCache
    }
}
// Every construction site re-specifies the closures; no shared contract;
// tests pass ad-hoc stubs that drift apart.

// Prefer — a named protocol seam with real and fake implementations
protocol DocumentStore {
    func fetch() async throws -> [Document]
    func save(_ documents: [Document]) async throws
}

final class InMemoryDocumentStore: DocumentStore {       // the test fake — real code
    private(set) var saved: [Document] = []
    func fetch() async throws -> [Document] { saved }
    func save(_ documents: [Document]) async throws { saved = documents }
}

// RemoteDocumentStore: DocumentStore would implement fetch()/save() over the network.

final class DocumentLoader {
    private let store: any DocumentStore
    init(store: any DocumentStore) { self.store = store }   // one named seam

    func documents() async throws -> [Document] { try await store.fetch() }
    func save(_ documents: [Document]) async throws { try await store.save(documents) }
}
```

The protocol version reads as a capability contract, the `InMemoryDocumentStore` fake is reusable across every test that needs it, and the production call site is `DocumentLoader(store: RemoteDocumentStore())` — no closure soup.

**Closures still have their place:** one-off callbacks (`onTap`, completion handlers), and small strategy parameters (`sorted(by:)`). The rule is narrower than "never use closures" — it is *don't use anonymous closures as a substitute for a named dependency contract that several call sites and tests share.*

The same instinct in TypeScript: an `interface DocumentStore` with a `RemoteDocumentStore` and an `InMemoryDocumentStore`, injected as one parameter — not a constructor taking `{ fetch: () => Promise<Document[]> }` of loose function props. (See the protocol-oriented section of [class-struct-design.md](class-struct-design.md).)

---

## 4. Inject Only What Varies

Dependency injection is a tool for substituting the seams that genuinely have substitutes — not a mandate to thread every collaborator through every initializer. Over-injection ("pass everything in so it's testable") makes real call sites verbose, buries which dependencies actually matter, and spreads wiring noise through the codebase.

```swift
// Over-injected — six dependencies, most with exactly one implementation
init(
    store: any DocumentStore,
    formatter: DateFormatter,          // stable, never substituted
    logger: Logger,                    // stable
    decoder: JSONDecoder,              // stable
    calendar: Calendar,                // stable
    notifier: any Notifier
) { ... }

// Focused — inject the seams that vary; construct stable internals directly
init(store: any DocumentStore, notifier: any Notifier) {
    self.store = store
    self.notifier = notifier
    self.formatter = DateFormatter()   // stable internal, built here
    self.decoder = JSONDecoder()
}
```

Inject the seam when it has a real substitute. Build stable internals directly. The initializer then reads as a list of *what genuinely varies*, which is information.

---

## 5. Mock Less — Fakes and Real Implementations Over Mock Frameworks

Prefer **real implementations and in-memory fakes** over mock frameworks that record and assert on interactions.

- A **fake** is real code with real behavior, written once and reused — `InMemoryDocumentStore` actually stores and returns documents. Tests assert on *outcomes*: "after saving, fetch returns these."
- A **mock** asserts on *interactions* — "`fetch` was called exactly once with these arguments." This couples the test to the implementation's internal call pattern. Refactor the implementation without changing behavior and the mock test breaks; introduce a behavioral bug that happens to preserve the call pattern and the mock test still passes.

Mock-style asserts on calls and couples the test to the implementation — e.g. a mock
framework's `verify(mockStore.fetch).wasCalled(times: 1)` is brittle because it tests the
"how". Fake-style asserts on behavior through the public API and tests the "what":

```swift
// Fake-style — asserts on behavior through the public API
let store = InMemoryDocumentStore()
let loader = DocumentLoader(store: store)
try await loader.save([doc])
let result = try await loader.documents()
#expect(result == [doc])                       // tests the "what"
```

At the system boundary, the same principle scales up: **real fixtures against staging beat a stubbed mock server.** A mock HTTP server that returns canned responses is not an acceptance substitute — it tests your stub, not the integration. Use real services with real test fixtures for end-to-end acceptance, and reserve fakes for fast unit-level isolation of genuinely external seams.

---

## Diagnostic

| Question | If No / Yes | Action |
|----------|-------------|--------|
| Is each behavior implemented once and called everywhere? | No — same logic at many call sites | Extract a named function/type; reference it everywhere |
| Does every seam have a real second implementation or a real fake? | No — one impl, no fake | Drop the protocol; use the concrete type directly |
| Are Swift seams protocols/subclasses rather than injected closures? | No — closure properties as dependencies | Replace with a named `protocol` + real and in-memory implementations |
| Does the initializer inject only what varies? | No — every collaborator injected | Inject the real seams; build stable internals inline |
| Do tests assert outcomes through the public API? | No — they verify mock call transcripts | Use an in-memory fake; assert behavior, not interactions |
| Is end-to-end acceptance run against real fixtures/staging? | No — a mock server stands in | Use real services with real fixtures at the boundary |
