# Method Design

A method is the smallest unit of reusable thought in code. Its design determines how easily a reader can understand what the code does, how safely it can be modified, and how effectively it can be tested.

## One Coherent Contract

**A method should expose one coherent operation at its call site.** It may coordinate several internal steps when those steps always belong to the same operation and share one error/ownership boundary.

- ✅ `bool IsValidIban(string input)` — checks one format
- ✅ `Task PersistUserAsync(User user, CancellationToken ct)` — persists a user
- ⚠️ `Task ValidateAndSaveUserAsync(User user, CancellationToken ct)` — unclear unless validation is an inseparable precondition of this save operation

Words such as `And`, `Then`, or `Also` are diagnostic signals, not proof that a split is needed. Split when callers need the steps independently, the steps change for different reasons, or the method mixes unrelated ownership. Keep orchestration together when extraction would create pass-through methods and force every caller to reconstruct the same sequence.

### Contract = preconditions + postconditions

A method's contract has a negative side: what it refuses (preconditions) and what it guarantees never happens afterwards (postconditions). State preconditions with guards or parameter types; state non-obvious postconditions with an assertion before returning. See [negative-space-programming.md](negative-space-programming.md).

---

## The One Level of Abstraction Rule

| Level | Characteristic | Example |
|---|---|---|
| High | Orchestrates other named methods, reads like a policy | `Task CheckoutAsync(Cart cart, CancellationToken ct)` |
| Mid | Transforms or coordinates between subsystems | `OrderPayload BuildOrderPayload(Cart cart)` |
| Low | Direct manipulation, computation, string/byte work | `static string AppendQuery(string url, string key, string value)` |

```csharp
// Mixed levels — avoid
public async Task CheckoutAsync(Cart cart, CancellationToken ct)
{
    var payload = BuildOrderPayload(cart);

    var builder = new UriBuilder(_options.BaseUrl) { Path = "/v2/orders" };   // low level, wrong place
    var query = HttpUtility.ParseQueryString(string.Empty);
    query["source"] = "web";
    builder.Query = query.ToString();

    var result = await _http.PostAsJsonAsync(builder.Uri, payload, ct);
    await HandleCheckoutResultAsync(result, ct);
}

// Consistent levels — prefer
public async Task CheckoutAsync(Cart cart, CancellationToken ct)
{
    var payload = BuildOrderPayload(cart);
    var result = await _ordersClient.CreateAsync(payload, ct);
    await HandleCheckoutResultAsync(result, ct);
}
```

---

## Method Length

Length is a symptom, not a rule. A longer method with one coherent contract and consistent abstraction can be clearer than several tiny methods that only forward arguments. Re-examine a method when it mixes abstraction levels, repeats cleanup or error handling, contains branches with independent purposes, or hides its contract. Extract only when the new name hides a coherent detail.

### The comment-as-method signal

```csharp
// Avoid
public async Task ProcessCheckoutAsync(CancellationToken ct)
{
    // Validate cart
    if (_cart.Items.Count == 0) throw new InvalidOperationException("Cart is empty");
    if (_cart.Total <= 0) throw new InvalidOperationException("Invalid total");

    // Build payload
    var payload = new CheckoutPayload(_cart.Items.Select(i => new PayloadItem(i)).ToList(), _cart.Total, _cart.Currency);

    // Submit
    await _api.SubmitOrderAsync(payload, ct);
}

// Prefer — comments become method names
public async Task ProcessCheckoutAsync(CancellationToken ct)
{
    EnsureCartIsCheckoutable();
    var payload = BuildCheckoutPayload();
    await _api.SubmitOrderAsync(payload, ct);
}
```

Better still: if "cart is checkoutable" is a state, make it a type (`CheckoutableCart`) produced by a parser.

---

## Parameters

### Group by meaning, not count

Introduce a parameter type (often a `record`) when the values form one domain concept, are validated together, or repeatedly travel together. Do not create a `XxxOptions`/`XxxRequest` record solely to reduce the visible parameter count.

### Strong types over primitives

Two `Guid` or two `string` parameters next to each other invite swapped arguments. Use strongly-typed IDs or value objects (see [parse-dont-validate.md](parse-dont-validate.md)), or named arguments at the call site.

### Avoid boolean flag parameters

```csharp
// Avoid — caller must know what `true` means
Export(report, true);

// Prefer — named argument for a small orthogonal variation
Export(report, includeDrafts: true);

// Prefer — separate methods or an enum when behaviors diverge
ExportPdf(report);
ExportCsv(report);
```

### Avoid `out`/`ref` for multiple results

`out` is idiomatic for the `TryX` pattern. For returning several values, return a tuple (`(int Count, decimal Total)`) or a record. Use `ref`/`in`/`Span<T>` only for measured performance work.

### `CancellationToken` last

Every async method doing I/O takes `CancellationToken ct` as its last parameter and forwards it.

---

## Return values

- Return what you compute; don't mutate fields as a hidden output.
- Return `IReadOnlyList<T>` for materialized collections, never `null` — return empty.
- Return `T?` when absence is normal; throw when absence means the caller's assumption was wrong; return a result type when the caller branches on several expected outcomes. See [nullability-and-exceptions.md](nullability-and-exceptions.md).
- Do not return `Task<bool>` "success" flags that callers ignore; throw or return a meaningful result.

---

## Side Effects and Naming

**The name must reveal all significant side effects.**

```csharp
// Dangerous — reads like a getter, performs I/O and mutation
public User CurrentUser()
{
    var user = _api.FetchUser().GetAwaiter().GetResult();
    _cache.Set("user", user);
    return user;
}

// Better
public User? CachedCurrentUser => _cache.Get<User>("user");
public async Task<User> RefreshCurrentUserAsync(CancellationToken ct) { ... }
```

---

## C#-Specific Patterns

### Properties vs methods

Use a property when the value is cheap, side-effect free, and does not throw in normal use (`FullName`, `IsExpired`, `Total`). Use a method when it does I/O, is expensive, can throw, or returns a new collection each call. Async is always a method.

### Expression-bodied members

Use them for single-expression members (`public bool IsEmpty => Items.Count == 0;`). Do not cram multi-step logic into one expression to save braces.

### Local functions

Use a local function for a helper used only by one method, especially for iterator/async validation splits. Make it `static` when it captures nothing.

### Extension methods

Use for a cohesive capability on a type you don't own, or for fluent DI registration. Do not use extension methods to bolt domain behavior onto your own types — put it on the type.

---

## Quick Reference

| Principle | Check |
|---|---|
| Coherent contract | Callers see one operation and one ownership boundary |
| Explicit preconditions | Guards or parameter types state what is refused |
| One abstraction level | All calls in the method are at the same "height" |
| Parameters | Group only when they form one concept; strong types over primitives |
| Boolean flags | Named arguments for small variations; split distinct operations |
| Return over mutate | Return the value instead of setting a field |
| Side effects in name | If it writes/sends/logs/queries, the name says so |
| Comment-as-method | If you wrote a comment, consider a method or a type |
