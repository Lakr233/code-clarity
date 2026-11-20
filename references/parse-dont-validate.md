# Parse, Don't Validate

*Validation* checks data and throws the knowledge away: after `IsValidEmail(s)` returns `true`, `s` is still just a `string`, and every downstream consumer must either trust or re-check it. *Parsing* checks data and **keeps the knowledge in the type**: `EmailAddress.Parse(s)` returns an `EmailAddress`, which can only exist in a valid form. The term comes from Alexis King's essay of the same name; the C# equivalent of "make illegal states unrepresentable" (Yaron Minsky).

This is the type-level half of [negative space programming](negative-space-programming.md): the strongest assertion is one the compiler checks.

---

## The shape: parse once at the boundary, trust inside

```
HTTP / JSON / config / DB / UI   →   parse   →   domain types   →   core logic
     (string, int, DTO)              (once)     (EmailAddress,      (no re-checking)
                                                 Quantity, ...)
```

Untrusted input enters as primitives or DTOs. One layer converts it into domain types and reports failure. Everything inward takes the domain types and never re-validates.

```csharp
// Validate — knowledge is lost, re-checked everywhere
public void Register(string email, int age)
{
    if (!EmailValidator.IsValid(email)) throw new ArgumentException(...);
    if (age < 18) throw new ArgumentException(...);
    _mailer.SendWelcome(email);         // mailer checks email again "to be safe"
}

// Parse — the signature carries the proof
public void Register(EmailAddress email, AdultAge age)
{
    _mailer.SendWelcome(email);         // cannot receive an invalid email
}
```

---

## Value objects with smart constructors

```csharp
public readonly record struct EmailAddress
{
    public string Value { get; }

    private EmailAddress(string value) => Value = value;

    public static bool TryParse(string? input, out EmailAddress email)
    {
        email = default;
        if (string.IsNullOrWhiteSpace(input)) return false;
        var trimmed = input.Trim();
        if (!MailAddress.TryCreate(trimmed, out _)) return false;
        email = new EmailAddress(trimmed.ToLowerInvariant());
        return true;
    }

    public static EmailAddress Parse(string input) =>
        TryParse(input, out var email) ? email : throw new FormatException($"'{input}' is not a valid email address.");

    public override string ToString() => Value;
}
```

Rules:

- **Private constructor**; the only way in is `Parse`/`TryParse`/`Create`. Follow the BCL pattern (`int.Parse`/`int.TryParse`) so the API is familiar. Implement `IParsable<T>` when the type is used generically or in minimal-API binding.
- **Normalize inside the parser** (trim, case, unit) so equality is meaningful.
- **Beware `default`** on structs: `default(EmailAddress)` bypasses the constructor. Use a `sealed record` (class) when `default` would be a dangerous valid-looking value, or assert `Value is not null` in members.
- **Do not create a value object for every primitive.** Create one when the value has rules (format, range, unit), is passed through several layers, or has been confused with a sibling primitive (`CustomerId` vs `OrderId` both `Guid`). A local `int count` needs no wrapper.

### Strongly-typed IDs

```csharp
public readonly record struct OrderId(Guid Value);
public readonly record struct CustomerId(Guid Value);

void Ship(OrderId order, CustomerId customer);   // arguments can no longer be swapped
```

---

## Make illegal states unrepresentable

### Replace flag combinations with a closed hierarchy

```csharp
// Avoid — 2^3 combinations, most invalid
public class Payment
{
    public bool IsPaid { get; set; }
    public bool IsRefunded { get; set; }
    public DateTime? PaidAt { get; set; }
    public string? RefundReason { get; set; }
}

// Prefer — each state carries exactly its data
public abstract record PaymentState
{
    private PaymentState() { }   // closed: only nested records can derive

    public sealed record Pending : PaymentState;
    public sealed record Paid(DateTimeOffset PaidAt) : PaymentState;
    public sealed record Refunded(DateTimeOffset PaidAt, string Reason) : PaymentState;
}

string Describe(PaymentState state) => state switch
{
    PaymentState.Pending => "Awaiting payment",
    PaymentState.Paid p => $"Paid at {p.PaidAt:d}",
    PaymentState.Refunded r => $"Refunded: {r.Reason}",
    _ => throw new UnreachableException(),
};
```

The private constructor closes the hierarchy; the `_ => throw` covers the compiler's inability to prove exhaustiveness over class hierarchies.

### Encode transitions in types

When operations are only valid in one state, make them exist only on that state:

```csharp
public sealed record DraftInvoice(InvoiceId Id, IReadOnlyList<InvoiceLine> Lines)
{
    public IssuedInvoice Issue(DateOnly issueDate)
    {
        if (Lines.Count == 0) throw new InvalidOperationException("Cannot issue an empty invoice.");
        return new IssuedInvoice(Id, Lines, issueDate);
    }
}

public sealed record IssuedInvoice(InvoiceId Id, IReadOnlyList<InvoiceLine> Lines, DateOnly IssueDate);
// No AddLine() here — an issued invoice cannot be edited, and the compiler enforces it.
```

Use this when the state graph is small and stable. For an entity persisted with EF Core where state is a column, a status enum plus asserted transitions (see [negative-space-programming.md](negative-space-programming.md)) is often the pragmatic choice.

### Non-empty and bounded collections

If "at least one" is a rule, say so in the type or constructor rather than checking `Any()` at every consumer.

```csharp
public sealed record OrderLines
{
    public IReadOnlyList<OrderLine> Items { get; }
    public OrderLines(IReadOnlyList<OrderLine> items)
    {
        ArgumentOutOfRangeException.ThrowIfZero(items.Count);
        Items = items;
    }
}
```

### `required` and `init`

```csharp
public sealed class SmtpOptions
{
    public required string Host { get; init; }
    public required int Port { get; init; }
}
```

Construction without `Host` no longer compiles. For `IOptions<T>`, combine with `ValidateDataAnnotations()`/`ValidateOnStart()` so configuration fails at startup, not on the first email.

---

## Where parsing lives in an ASP.NET Core app

| Boundary | Parse with |
|---|---|
| HTTP request | Request DTO → mapping/handler converts to domain types; return `ValidationProblem` on failure |
| Configuration | `IOptions<T>` + `ValidateOnStart()` |
| Database | EF Core value converters for value objects / strongly-typed IDs |
| External API response | Deserialize to DTO, then parse; never let the external DTO leak inward |
| Message bus | Same as HTTP: DTO → domain at the consumer edge |

Choose one validation style per boundary (FluentValidation, DataAnnotations, or parsing in the handler) and use it consistently; do not layer all three.

---

## Review questions

1. Is the same primitive validated in more than one place? → Introduce a parsed type.
2. Do two `bool`/nullable properties have combinations that must never occur? → Closed hierarchy or enum.
3. Can two parameters of the same primitive type be swapped by accident? → Strongly-typed IDs.
4. Does a method exist on a type in states where calling it is illegal? → Split the state type, or assert the transition.
5. Does an external DTO travel past the boundary into domain code? → Map it at the edge.
