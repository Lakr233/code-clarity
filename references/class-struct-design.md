# Class, Record, and Struct Design

Types are the primary units of ownership. A well-designed type has a coherent responsibility, a name that reflects what it owns, invariants it protects, and an interface smaller than the implementation it hides. A poorly designed type either accumulates unrelated responsibilities or is split into shallow wrappers that expose rather than contain complexity.

---

## Coherent Ownership

**A type should own one coherent body of knowledge or lifecycle — and the invariants that come with it.** It can contain several steps when keeping them together hides decisions from callers. Split when parts have independent owners, invariants, consumers, or reasons to change — not merely because a description contains "and".

- ✅ `TokenRefresher` — refreshes expired authentication tokens
- ✅ `InvoiceNumberGenerator` — issues gap-free invoice numbers per year
- ❌ `UserManager` — authentication, profile updates, session state, and avatar uploads

A coordinator that owns one workflow may legitimately call several collaborators; do not replace it with a chain of one-method forwarding classes.

### The type guards its invariants

If a rule must always hold for an object ("total equals sum of lines", "end ≥ start"), the type enforces it — through its constructor, private setters, and methods — so no caller can break it. Public setters on domain state move the invariant to every caller.

```csharp
// Avoid — every caller must remember the rule
public class DateRange { public DateOnly Start { get; set; } public DateOnly End { get; set; } }

// Prefer — the type cannot exist in an invalid form
public readonly record struct DateRange
{
    public DateOnly Start { get; }
    public DateOnly End { get; }

    public DateRange(DateOnly start, DateOnly end)
    {
        if (end < start) throw new ArgumentException($"End {end} is before start {start}.");
        (Start, End) = (start, end);
    }

    public int Days => End.DayNumber - Start.DayNumber + 1;
}
```

---

## Choosing the kind of type

### `record` / `readonly record struct` — values

- Defined by contents, not identity; two equal instances are interchangeable.
- Immutable; "modification" via `with` creates a new value.
- `readonly record struct` for small values (IDs, money, coordinates, ranges); `record` (class) for larger values or when `default` would be a dangerous valid-looking instance.

```csharp
public readonly record struct Money(decimal Amount, Currency Currency);
public sealed record Address(string Street, string HouseNumber, string PostalCode, string City);
```

### `sealed class` — identities and owners

- Identity matters (an `Order` with id 42 is the same order even after changes).
- Owns a lifecycle or resource (`IDisposable`, connections, subscriptions).
- Mutable state shared by reference (a scoped unit of work, a cache).

Seal by default. Inheritance is a public contract you must design for; unsealed classes invite it accidentally.

### EF Core entities

Entities are identity types. Keep invariants in methods (`order.AddLine(...)`, `order.MarkPaid(...)`), use private setters, and a private parameterless constructor for EF. Do not expose `List<T>` navigation properties publicly; expose `IReadOnlyCollection<T>` backed by a private field.

### `struct` (non-record)

Only for measured performance work (`Span`-heavy parsers, hot-path math). Mutable structs are a common source of bugs; make them `readonly`.

---

## Coupling and Cohesion

### Low coupling

```csharp
// High coupling — the constructor lists every subsystem
public sealed class CheckoutController(
    UserService users, PaymentService payments, InventoryService inventory,
    AnalyticsService analytics, NotificationService notifications, EmailService email) { ... }
```

Six dependencies in a controller usually means the use case is being orchestrated in the wrong layer. Move the workflow to one handler/service that owns it; the controller depends on that one thing.

Reduce coupling by:

- passing only the values or capabilities the type actually needs;
- keeping a concrete dependency when behavior does not vary;
- introducing a small consumer-owned interface when a real substitute exists;
- grouping coordination only when it hides a stable workflow from callers.

### High cohesion

Everything inside a type relates to its responsibility. Methods that don't use instance state, fields used by only one method, and presentation logic on domain types are cohesion failures.

```csharp
// Low cohesion — presentation on a domain entity
public sealed class Invoice
{
    public string ToReceiptHtml() { ... }
}

// High cohesion — formatting lives in presentation
public sealed class InvoiceReceiptRenderer
{
    public string RenderHtml(Invoice invoice) { ... }
}
```

---

## Accumulation: The Most Common Design Failure

Types rarely start with too many responsibilities; they accumulate them. `AuthService` gains password reset ("it's auth"), then account deletion ("users do it from settings"), then rate limiting ("it's security").

### Signs of accumulation

- Methods that use disjoint sets of fields
- Many unrelated `using` namespaces
- Tests that require setting up unrelated dependencies (`Mock<T>` for six services to test one method)
- PRs that "should be simple" always touch this class
- A `partial class` used to hide size rather than to separate generated code

### Split by reason to change

```csharp
// Wrong axis — split by kind of code
public sealed class AuthServiceValidation { }
public sealed class AuthServicePersistence { }

// Right axis — split by reason to change
public sealed class LoginService { }
public sealed class TokenStore { }
public sealed class PasswordResetService { }
```

---

## Interfaces

An interface defines a capability contract. Use one when a consumer needs a stable subset of behavior **with real substitutes** (multiple implementations, or a durable in-memory fake). An `IFoo` for every `Foo` is not "SOLID"; every member expands all implementations and fakes.

```csharp
public interface IOrderRepository
{
    Task<Order?> FindAsync(OrderId id, CancellationToken ct);
    Task SaveAsync(Order order, CancellationToken ct);
}

public sealed class EfOrderRepository(AppDbContext db) : IOrderRepository { ... }
public sealed class InMemoryOrderRepository : IOrderRepository { ... }   // real fake for tests
```

For a single varying operation, a `Func<...>` parameter or `TimeProvider`-style abstraction may be smaller. With no variation, inject the concrete class.

Avoid default interface methods for shared logic; they blur where behavior lives.

---

## Design Checklist

| Question | Pass | Fail |
|---|---|---|
| Does the type own one coherent body of knowledge or lifecycle? | Yes | Unrelated invariants and consumers accumulate |
| Does the type enforce its own invariants? | Constructor + methods guard them | Public setters; callers must remember rules |
| Does the name reveal that ownership in repository vocabulary? | Yes | Readers must inspect the implementation |
| `record`/struct for values, `sealed class` for identities? | Yes | Mutable classes as pure data bags |
| Do all methods use the type's state? | Yes | Static-like methods living on an instance |
| Does each dependency represent something the type genuinely needs? | Yes | Dependencies exist only for wiring or mocks |
| Would a split create deeper modules rather than pass-through wrappers? | Yes | The split only adds forwarding classes |
| Is the type easy to test without unrelated setup? | Yes | Many mocks for one behavior |
