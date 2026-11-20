# Nullability, Guard Clauses, and Exceptions

---

## Part 1 — Nullable reference types (NRT)

Enable NRT project-wide and make warnings errors. Without it, every reference is "maybe null" and the negative space is invisible.

```xml
<!-- Directory.Build.props -->
<PropertyGroup>
  <Nullable>enable</Nullable>
  <WarningsAsErrors>nullable</WarningsAsErrors>
</PropertyGroup>
```

### What the annotations mean

| Declaration | Meaning | Consumer obligation |
|---|---|---|
| `string name` | Never null — guaranteed by the producer | None. Do not null-check it. |
| `string? nickname` | Absence is a legal, expected state | Handle the null case |
| `name!` | "I know better than the compiler" — an unchecked assertion | Justify in review; prefer a real proof |

Redundant null checks on non-nullable values are noise and hide which values can actually be absent. Delete them.

### Prove non-null rather than suppress

```csharp
// Avoid — ! hides an assumption
var user = _users.Find(id)!;

// Prefer — make the absence a decision
var user = _users.Find(id) ?? throw new NotFoundException($"User {id}");

// Or — absence is normal, so model it
if (_users.Find(id) is not { } user) return Results.NotFound();
```

Use flow attributes so helpers inform the compiler: `[NotNullWhen(true)]` on `TryX` out parameters, `[MemberNotNull]` on initialization helpers, `[DoesNotReturn]` on throw helpers.

### Nullable is not a state machine

`DateTime? ShippedAt` plus `string? TrackingCode` plus `bool IsShipped` is three ways to say one thing. Pick one authoritative representation (often a state type, see [parse-dont-validate.md](parse-dont-validate.md)) and derive the rest.

### Collections: empty, not null

Return empty collections (`[]`, `Array.Empty<T>()`), never `null`. `IReadOnlyList<T>?` is almost always a design smell.

---

## Part 2 — Guard clauses

Guards protect the **public contract** against caller mistakes. Use the BCL throw helpers; they are concise, consistent, and fill in the parameter name.

```csharp
public Invoice CreateInvoice(Customer customer, IReadOnlyList<InvoiceLine> lines, int paymentTermDays)
{
    ArgumentNullException.ThrowIfNull(customer);
    ArgumentNullException.ThrowIfNull(lines);
    ArgumentOutOfRangeException.ThrowIfZero(lines.Count);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(paymentTermDays);

    // happy path at base indentation
}
```

Available helpers (.NET 8+): `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrEmpty`, `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegative / ThrowIfZero / ThrowIfNegativeOrZero / ThrowIfGreaterThan / ThrowIfLessThan / ThrowIfEqual`, `ObjectDisposedException.ThrowIf`.

Where to guard:

- **Public / protected members of a library or shared module:** guard. Callers are outside your control.
- **`private` / `internal` members:** do not repeat guards the public entry already performed. Use `Debug.Assert` if the invariant is non-obvious.
- **Non-nullable parameters with NRT on:** `ThrowIfNull` is still useful on public APIs consumed by code without NRT, or from reflection/deserialization. Inside one NRT-enabled solution, it is often redundant — follow repository convention.

---

## Part 3 — Exceptions vs. results

Three kinds of failure, three tools:

| Kind | Example | Tool |
|---|---|---|
| **Programmer error** | null argument, invalid state transition, unreachable case | Throw (`ArgumentException`, `InvalidOperationException`, `UnreachableException`). Never catch locally. |
| **Exceptional operational failure** | DB down, network timeout, disk full | Let the exception propagate to a boundary that can retry, translate, or return 5xx |
| **Expected business outcome** | not found, validation failed, insufficient funds, duplicate | `TryX` pattern, `bool`/nullable return, or a `Result` type — when the caller routinely handles it |

### Use exceptions when the caller usually cannot handle the failure here

```csharp
public async Task<Order> GetOrderAsync(OrderId id, CancellationToken ct) =>
    await _db.Orders.FindAsync([id], ct) ?? throw new OrderNotFoundException(id);
```

…and translate centrally (`IExceptionHandler` / ProblemDetails middleware) into HTTP responses. Do not wrap every controller action in try/catch.

### Use `TryX` / nullable when absence is normal and local

```csharp
public bool TryGetCachedPrice(ProductId id, out Money price) { ... }
public Customer? FindByEmail(EmailAddress email) { ... }
```

### Use a `Result` type when there are several expected outcomes the caller branches on

```csharp
public abstract record RegisterResult
{
    private RegisterResult() { }
    public sealed record Success(UserId Id) : RegisterResult;
    public sealed record EmailTaken : RegisterResult;
    public sealed record WeakPassword(IReadOnlyList<string> Reasons) : RegisterResult;
}

return await _registration.RegisterAsync(command, ct) switch
{
    RegisterResult.Success s => Results.Created($"/users/{s.Id}", null),
    RegisterResult.EmailTaken => Results.Conflict(),
    RegisterResult.WeakPassword w => Results.ValidationProblem(...),
    _ => throw new UnreachableException(),
};
```

Do not introduce a generic `Result<T, TError>` library and wrap *every* method in it. That turns infrastructure failures (which nobody handles locally) into ceremony at every call site. Use results for the few operations whose outcomes callers genuinely branch on; let exceptions carry the rest. Follow the repository if it has already committed to one approach.

### Custom exception types

Create one only when a caller catches it specifically (see the decision gate in SKILL.md). Otherwise use the BCL type with a precise message. If you create one:

```csharp
public sealed class OrderNotFoundException(OrderId id)
    : Exception($"Order {id.Value} was not found.")
{
    public OrderId OrderId { get; } = id;
}
```

### Catching rules

- Catch the **narrowest** type at the **boundary that can act**: retry, fall back, translate, or add context.
- When adding context, keep the cause: `throw new ImportException($"Row {row}", ex);`.
- Rethrow with `throw;`, never `throw ex;` (loses the stack trace).
- Use exception filters to avoid catching what you do not handle: `catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)`.
- `catch (OperationCanceledException)` when `ct.IsCancellationRequested` is cancellation, not failure — do not log it as an error.
- Never `catch (Exception) { }`. A top-level handler (middleware, `BackgroundService` wrapper, `AppDomain.UnhandledException`) is the only legitimate catch-all, and it logs and terminates the unit of work.

### Logging is diagnostics, not control flow

Log once, where the exception is handled — not at every layer it passes through. Use structured templates (`_logger.LogWarning(ex, "Import of {FileName} failed at row {Row}", file, row)`), not string interpolation.

---

## Review questions

1. Is NRT enabled and are its warnings errors?
2. Which `!` operators lack a justification?
3. Which null checks guard non-nullable values?
4. Which `catch` blocks neither recover, translate, nor add context?
5. Which custom exception types are never caught specifically?
6. Is a `Result` type used for failures nobody branches on?
