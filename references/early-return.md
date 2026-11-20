# Early Return and Guard Clauses

Handle preconditions, edge cases, and error conditions at the top of a method with an immediate `return` or `throw`, so the main logic runs at base indentation. Guards are the runtime front line of [negative space programming](negative-space-programming.md): they state what the method refuses before it states what it does.

## The Core Problem: Arrow Code

```csharp
// Arrow code — avoid
public IResult PlaceOrder(Order? order)
{
    if (order != null)
    {
        if (order.IsValid)
        {
            if (_currentUser.CanPlaceOrders)
            {
                if (_inventory.Has(order.Items))
                {
                    Charge(order);           // main logic buried 4 levels deep
                    return Results.Ok();
                }
                else return Results.Conflict("Out of stock");
            }
            else return Results.Forbid();
        }
        else return Results.BadRequest();
    }
    return Results.BadRequest();
}
```

## The Solution: Exit Early, Stay Flat

```csharp
public IResult PlaceOrder(Order? order)
{
    if (order is null) return Results.BadRequest();
    if (!order.IsValid) return Results.BadRequest();
    if (!_currentUser.CanPlaceOrders) return Results.Forbid();
    if (!_inventory.Has(order.Items)) return Results.Conflict("Out of stock");

    Charge(order);
    return Results.Ok();
}
```

Each guard is a complete thought that can be read and dismissed. After `if (order is null) return ...;` the compiler's flow analysis knows `order` is non-null for the rest of the method.

---

## C# guard tools

### BCL throw helpers for contract violations

```csharp
public void Schedule(Appointment appointment, TimeSpan duration)
{
    ArgumentNullException.ThrowIfNull(appointment);
    ArgumentOutOfRangeException.ThrowIfLessThanOrEqual(duration, TimeSpan.Zero);

    // ...
}
```

### Pattern-based guards that bind

```csharp
if (await _db.Customers.FindAsync([id], ct) is not { } customer)
    return Results.NotFound();

if (customer is not { IsActive: true, Subscription: { } subscription })
    return Results.Problem("Customer has no active subscription");

// customer and subscription are non-null here
```

### `TryX` guards

```csharp
if (!EmailAddress.TryParse(request.Email, out var email))
    return Results.ValidationProblem(...);
```

### `continue` in loops

```csharp
foreach (var line in lines)
{
    if (line.Quantity == 0) continue;
    if (line.IsGift) continue;

    total += line.Price * line.Quantity;
}
```

---

## The `else` After `return` Anti-Pattern

```csharp
// Redundant else — avoid
string Label(Status status)
{
    if (status == Status.Active) { return "Active"; }
    else { return "Inactive"; }
}

// Prefer — or a switch expression when there are more cases
string Label(Status status) => status == Status.Active ? "Active" : "Inactive";
```

---

## When NOT to Use Early Return

### Don't force asymmetry

If two branches are equally valid outcomes, `if/else` (or a ternary / switch expression) communicates parity:

```csharp
var view = user.IsPremium ? RenderPremiumDashboard() : RenderFreeDashboard();
```

### Don't chain tiny guards for trivial logic

```csharp
if (isReady && hasData) DisplayContent();
```

### Resources requiring cleanup

Use `using`/`await using` declarations and `try/finally` so early returns cannot skip cleanup:

```csharp
public async Task<byte[]> ReadSmallFileAsync(string path, CancellationToken ct)
{
    await using var stream = File.OpenRead(path);
    if (stream.Length > MaxFileSize) throw new InvalidDataException("File too large");

    var buffer = new byte[stream.Length];
    await stream.ReadExactlyAsync(buffer, ct);
    return buffer;
}
```

### Guards are not a substitute for types

A method that starts with eight guards on primitives is a sign that a parsed type is missing. Move the guards into a value object's constructor once (see [parse-dont-validate.md](parse-dont-validate.md)).

---

## Decision Guide

| Condition | Pattern |
|---|---|
| Null argument on public API | `ArgumentNullException.ThrowIfNull(x);` |
| Out-of-range argument | `ArgumentOutOfRangeException.ThrowIfXxx(...)` |
| Object in wrong state | `if (...) throw new InvalidOperationException(...)` |
| Lookup may miss, caller handles it | `if (... is not { } x) return NotFound;` |
| Untrusted input | `TryParse` guard → validation response |
| Internal invariant | `Debug.Assert(...)` (not a guard) |
| Two equally valid outcomes | ternary / `switch` expression / `if-else` |
| Cleanup required | `using` declaration + early return |
