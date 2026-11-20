# Dependencies, Seams, and Testing

Testability is a design property, not a test-time afterthought. Code is easy to test when it has **one canonical implementation of each behavior**, a small number of **deliberately designed seams** where behavior genuinely varies, and tests that protect observable outcomes instead of private choreography.

---

## 1. One Canonical Implementation — Don't Reimplement at Each Call Site

```csharp
// Avoid — the same formatting re-implemented everywhere
var a = $"{customer.FirstName} {customer.LastName}".Trim();
// ...elsewhere...
var b = $"{other.FirstName} {other.LastName}".Trim();

// Prefer — one owner
public sealed class Customer
{
    public string DisplayName => $"{FirstName} {LastName}".Trim();
}
```

When logic repeats, first decide whether the call sites implement the same policy and should change together. If so, give that policy one owner. If the resemblance is incidental, or a shared abstraction would need flags for each caller, a small amount of duplication can be clearer.

Signs of scattered duplication:

- A bug fix has to be applied "in all the places that do X."
- Two call sites compute the same thing with subtly different code.
- A reader cannot tell which copy is authoritative.

---

## 2. A Seam Earns Its Place Only When Behavior Varies

A seam is a point where one implementation can be substituted for another — in C#, an interface, a delegate, an abstract/virtual member, or `TimeProvider`. Add one only when:

- there is a real second implementation (EF vs in-memory, Stripe vs Mollie, production vs sandbox), **or**
- a focused substitute is needed for an external or nondeterministic dependency (clock, random, HTTP API, message bus).

The `IFoo`/`Foo` pair for every service — "interface per class" — is the most common C# over-abstraction. It doubles the files, adds a DI registration, and every "Go to definition" lands on the interface instead of the code.

```csharp
// Over-abstracted — one implementation, only mocked to verify calls
public interface IVatCalculator { decimal Calculate(decimal net, VatRate rate); }
public sealed class VatCalculator : IVatCalculator { ... }

// Clearer — pure logic needs no seam; test it directly
public static class Vat
{
    public static decimal Calculate(decimal net, VatRate rate) => Math.Round(net * rate.Fraction, 2, MidpointRounding.AwayFromZero);
}
```

---

## 3. Match the Seam to the Variation

- **Concrete dependency** when production has one behavior and tests can exercise it directly (pure logic, `DbContext` with a real test database).
- **Interface** when several operations form one capability with a real alternate implementation or durable fake.
- **Delegate / `Func<>`** for one local operation (a comparer, a strategy used in one place).
- **BCL abstraction** where one exists: `TimeProvider` (time), `IHttpClientFactory` + `HttpMessageHandler` (HTTP), `IFileProvider`/`System.IO.Abstractions` (file system, only if the repo uses it).
- **Virtual member / abstract class** only when the framework or repository already relies on inheritance.

```csharp
// Avoid — a bespoke clock interface
public interface IDateTimeProvider { DateTime UtcNow { get; } }

// Prefer — the BCL seam (.NET 8+), FakeTimeProvider in tests
public sealed class SubscriptionRenewer(TimeProvider clock, AppDbContext db) { ... }
```

---

## 4. Inject Only What Varies

```csharp
// Over-injected — most dependencies have exactly one implementation
public InvoiceExporter(
    IInvoiceRepository repository,
    ICsvFormatter formatter,        // stable, never substituted
    IFileNameBuilder fileNames,     // stable
    IJsonSerializerWrapper json,    // wraps JsonSerializer for no reason
    ILogger<InvoiceExporter> logger,
    TimeProvider clock) { ... }

// Focused — inject the seams; construct stable internals directly
public InvoiceExporter(IInvoiceRepository repository, TimeProvider clock, ILogger<InvoiceExporter> logger) { ... }
```

`ILogger<T>` is fine to inject; it is infrastructure, and `NullLogger<T>.Instance` covers tests.

---

## 5. Mock Less — Fakes and Real Implementations Over Mock Frameworks

- A **fake** is real code with real behavior, written once and reused — `InMemoryOrderRepository` actually stores and returns orders. Tests assert on outcomes.
- A **mock** (Moq, NSubstitute) asserts on interactions — `mock.Verify(r => r.SaveAsync(It.IsAny<Order>(), default), Times.Once)`. This couples the test to the implementation's call pattern: refactor without changing behavior and it breaks; introduce a bug that preserves the call pattern and it passes.

```csharp
// Mock-style — tests the "how"
var repo = new Mock<IOrderRepository>();
await new OrderService(repo.Object).PlaceAsync(order, default);
repo.Verify(r => r.SaveAsync(order, It.IsAny<CancellationToken>()), Times.Once);

// Fake-style — tests the "what"
var repo = new InMemoryOrderRepository();
await new OrderService(repo).PlaceAsync(order, default);
Assert.Equal(OrderStatus.Placed, (await repo.FindAsync(order.Id, default))!.Status);
```

Mocks remain useful for verifying a genuine outbound side effect with no observable state (an email was sent, an event was published) — and even then, a recording fake (`FakeEmailSender.Sent`) is usually clearer.

### Test at the real boundary when practical

- **EF Core:** test against the real provider (SQL Server/PostgreSQL via Testcontainers). The EF in-memory provider does not enforce constraints, transactions, or SQL translation and gives false confidence. SQLite in-memory is a compromise.
- **ASP.NET Core:** `WebApplicationFactory<TProgram>` exercises routing, binding, validation, filters, and serialization end to end; replace only external seams via `ConfigureTestServices`.
- **HTTP clients:** a stub `HttpMessageHandler` or WireMock.Net tests the real typed client.

---

## 6. Test Recovery Classes, Not Every Internal Branch

Tests should correspond to behavior a caller can distinguish:

- success;
- accepted fallback or degradation;
- retryable failure, if retry behavior differs;
- terminal rejection (validation, not found, conflict);
- an intentionally supported compatibility boundary;
- arithmetic, rounding, indexing, or resource-release boundaries.

Use `[Theory]` + `[InlineData]` for genuine equivalence classes of input (boundary values for a parser), not to visit every guard in a private helper.

### Tests and assertions together

Production assertions ([negative-space-programming.md](negative-space-programming.md)) turn every test into an invariant check: tests run in Debug, so `Debug.Assert` fires during test execution. A test suite with no assertion failures across many paths is stronger evidence than return-value checks alone. Do not write tests that *expect* `Debug.Assert` to fire; test the public guard (`Assert.Throws<ArgumentOutOfRangeException>`) instead.

### Design feedback

If adding one interface member forces many fakes or mock setups to change, the interface is probably too broad or the behavior belongs to a narrower owner.

---

## Diagnostic

| Question | If No / Yes | Action |
|---|---|---|
| Is each behavior implemented once? | No — same logic at many call sites | Extract a named member/type; reference it everywhere |
| Does every interface have a real second implementation or a real fake? | No — one impl, only mocked | Drop the interface; inject the concrete class |
| Does the seam shape match the variation? | No — bespoke clock, wrapper around `JsonSerializer` | Use the BCL abstraction or no seam |
| Does the constructor inject only what varies? | No — every collaborator injected | Inject the seams; build stable internals inline |
| Do tests assert outcomes? | No — `Verify(...)` transcripts | Use fakes; assert state/results |
| Does boundary testing exercise the real implementation? | No — EF in-memory provider, stubbed everything | Testcontainers / `WebApplicationFactory` |
| Does each test represent distinct behavior or a dangerous boundary? | No — it only visits another private guard | Fold into a behavior-class test or delete |
