# Dependencies, Seams, and Testing

Testability is a design property, not a test-time afterthought. Code is easy to test when it has **one canonical implementation of each behavior**, a small number of **deliberately designed seams** where behavior genuinely varies, and tests that protect observable outcomes instead of private choreography. This reference covers three intertwined habits: don't duplicate implementations, don't over-inject, and use the smallest seam that matches the real variation.

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

When logic repeats, first decide whether the call sites implement the same policy and should change together. If so, give that policy one owner. If the resemblance is incidental or the abstraction would need flags for each caller, a small amount of duplication can be clearer than a false shared abstraction.

Signs of scattered duplication:
- A bug fix has to be applied "in all the places that do X."
- Two call sites compute the same thing with subtly different code.
- A reader cannot tell which copy is the real one.

---

## 2. A Seam Earns Its Place Only When Behavior Varies

A *seam* is a point where you can substitute one implementation for another — a Swift `protocol` or overridable subclass, a TypeScript `interface`. Seams are valuable, but each one is also indirection a reader must follow. Add a seam only when behavior **genuinely varies** or the consumer needs a stable capability boundary:

- there is a real second implementation (remote vs local, production vs OSS, platform A vs B), **or**
- a focused substitute is needed for an external or nondeterministic dependency and cannot be supplied more locally.

If a type has one implementation and callers need its full API, **a direct concrete reference is clearer** than a protocol plus injection ceremony. A protocol designed from hypothetical future implementations or from a mock transcript usually adds indirection without information.

```swift
// Over-abstracted — one impl, no fake, pure ceremony
protocol GreetingProvider { func greeting() -> String }
final class DefaultGreetingProvider: GreetingProvider { func greeting() -> String { "Hi" } }

// Clearer — use the value directly in its owner
private let greeting = "Hi"
```

Introduce the protocol when concrete variation reveals the smallest capability consumers actually need. For a single injected operation, prefer a function parameter over a one-method protocol.

---

## 3. Match the Seam to the Variation

Do not turn “prefer protocols” or “prefer closures” into a universal rule. Choose the smallest representation that accurately describes how behavior varies.

- Use a concrete dependency when production has one behavior and tests can exercise it directly.
- Use a protocol/interface when several operations form one reusable capability and there is a real alternate implementation or durable fake.
- Use a closure/function parameter for one local operation such as a clock, runner, callback, comparator, or one-shot strategy.
- Use a subclass seam only when the framework or repository already relies on inheritance.

A bag of stored closures is usually an unnamed protocol. A protocol with one method used at one call site is often a named closure. Both can be overdesigned.

```swift
// Avoid — several operations form a hidden contract
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

// Prefer — name a capability that has reusable implementations
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

The protocol version reads as a capability contract because fetching and saving belong together and the fake has real reusable behavior.

For one operation, a closure is clearer:

```swift
func runRefresh(now: () -> Date = Date.init) -> RefreshDecision { ... }
```

Do not invent `ClockProviding` unless a broader clock capability is actually used. In Go, define interfaces in the consuming package and keep them small. In TypeScript, a structural object or function type is often enough; do not add a class solely to satisfy dependency-injection style.

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

At an external API boundary, exercise the real implementation when practical. A canned stub proves only the behavior of the stub. Reserve fakes for fast tests of code on the consuming side of a genuine seam.

## 6. Test Recovery Classes, Not Every Internal Branch

Tests should correspond to behavior a caller can distinguish. If five internal failures intentionally share one fallback, test the fallback contract and add individual regressions only for observed defects or dangerous arithmetic, indexing, or ownership boundaries.

Good behavior classes are usually:

- success;
- accepted fallback or degradation;
- retryable failure, if retry behavior differs;
- terminal rejection;
- an intentionally supported compatibility boundary;
- arithmetic, indexing, ownership, or resource-release boundaries where a mistake can corrupt a value or leak a resource.

Avoid parameterized matrices whose only purpose is to visit every guard in a private helper. Centralize validation, test representative boundaries, and let type checking plus integration tests cover the rest.

If adding one production method forces many fakes to gain no-op implementations, treat that as design feedback: the interface may be too broad, the behavior may belong to a narrower owner, or the test may be asserting the implementation shape.

---

## Diagnostic

| Question | If No / Yes | Action |
|----------|-------------|--------|
| Is each behavior implemented once and called everywhere? | No — same logic at many call sites | Extract a named function/type; reference it everywhere |
| Does every seam have a real second implementation or a real fake? | No — one impl, no fake | Drop the protocol; use the concrete type directly |
| Does the seam shape match the real variation? | No — one-off protocol or closure bag | Use a concrete type, one closure, or one coherent capability |
| Does the initializer inject only what varies? | No — every collaborator injected | Inject the real seams; build stable internals inline |
| Do tests assert outcomes through the public API? | No — they verify mock call transcripts | Use an in-memory fake; assert behavior, not interactions |
| Does boundary testing exercise the real implementation when practical? | No — a stub replaces the behavior under test | Move the test to the real boundary or narrow the claim |
| Does each test represent distinct behavior or a dangerous boundary? | No — it only visits another private guard | Fold it into the behavior-class test or delete it |
