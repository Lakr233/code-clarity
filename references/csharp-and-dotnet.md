# C# and .NET Idioms for Clarity

Language features that reduce concepts when used deliberately — and the ways they add ceremony when used by default.

---

## Types: pick by semantics

| Need | Use | Notes |
|---|---|---|
| Immutable value, equality by content | `record` / `readonly record struct` | Small (≤ ~16 bytes) and copy-cheap → struct; otherwise class record |
| Identity, lifecycle, mutable owner | `sealed class` | Seal by default (`CA1852`); unseal only for a designed extension point |
| Closed set of payload-free states | `enum` + exhaustive `switch` expression | Add `[Flags]` only for real bit sets |
| Closed set of states with different data | abstract record with private ctor + nested sealed records | See [parse-dont-validate.md](parse-dont-validate.md) |
| Stateless functions on one subject | `static class` | Not a dumping ground: name by subject |
| Capability with real substitutes | `interface` | Defined near the consumer, small |

Avoid `abstract class XxxBase` hierarchies for code reuse; prefer composition.

### Primary constructors

```csharp
public sealed class InvoiceService(AppDbContext db, TimeProvider clock, ILogger<InvoiceService> logger)
{
    public async Task IssueAsync(InvoiceId id, CancellationToken ct) { ... }
}
```

Primary constructor parameters are mutable captures, not `readonly` fields. In types where reassignment would be a bug, assign to `private readonly` fields or enable an analyzer that flags reassignment. Do not mix both styles in one class.

### `required`, `init`, and immutability

Make construction complete (`required`) and state immutable (`init`, `IReadOnlyList<T>`) unless mutation is the type's purpose. Expose `IReadOnlyList<T>` / `IReadOnlyCollection<T>`, keep `List<T>` private.

---

## Pattern matching over ladders

```csharp
// Avoid
if (shipment.Carrier == Carrier.PostNL && shipment.Weight > 30) return 12.50m;
else if (shipment.Carrier == Carrier.PostNL) return 6.95m;
else if (shipment.Carrier == Carrier.DHL) return 7.45m;
else return 0m;

// Prefer — one table, exhaustive, impossible default is loud
decimal ShippingCost(Shipment s) => s switch
{
    { Carrier: Carrier.PostNL, WeightKg: > 30 } => 12.50m,
    { Carrier: Carrier.PostNL } => 6.95m,
    { Carrier: Carrier.Dhl } => 7.45m,
    _ => throw new UnreachableException($"No rate for {s.Carrier}"),
};
```

Use `is not null`, `is { } x`, `is not { Count: > 0 }` for readable negative checks.

---

## LINQ

- Use LINQ for transformations; use `foreach` when the body has side effects or early exits.
- Materialize once (`ToList()`, `ToArray()`) when a sequence is consumed more than once (`CA1851`).
- With EF Core, know which part runs in SQL: everything before `ToListAsync()` must be translatable. Do not call `AsEnumerable()` to "fix" a translation error without checking the row count.
- One operator per line for chains longer than ~3 operators; short chains stay on one line.
- `Any()` over `Count() > 0`; `FirstOrDefault` only when absence is legal — otherwise `First`/`Single` (an assertion that the element exists).

---

## Dependency injection

DI is plumbing, not design. Clarity rules:

- Register **concrete types** when nothing varies: `services.AddScoped<InvoiceService>();`. An `IInvoiceService` with one implementation and no fake is pure ceremony.
- Inject what varies or what crosses an infrastructure boundary (`TimeProvider`, `IHttpClientFactory`, `DbContext`, a repository with an in-memory fake). Build stable internals directly.
- A constructor with 6+ dependencies is a cohesion signal, not a DI problem — split the class by reason to change.
- Use `TimeProvider` instead of `DateTime.UtcNow` when time affects behavior; it is the BCL's clock seam.
- Use `IOptions<T>` with `ValidateOnStart()` for configuration; do not inject `IConfiguration` into business code.
- Keep registration grouped per feature in an extension method (`services.AddInvoicing()`), not one 400-line `Program.cs`.

---

## Constants

```csharp
private const int MaxRetryCount = 3;
private static readonly TimeSpan SettleDelay = TimeSpan.FromMilliseconds(300);
```

- `const` for compile-time primitives/strings; `static readonly` for everything else (`TimeSpan`, arrays, objects).
- Narrowest scope: method-local `const` → `private` → `internal` → `public`. A `public const` is baked into consuming assemblies; changing it requires recompiling them — use `static readonly` for public values that may change.
- Group related constants in a nested `private static class Limits { ... }` only when there are several.

---

## Analyzers as enforcement

Encode clarity rules in the build instead of in review comments:

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <AnalysisLevel>latest-recommended</AnalysisLevel>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
</PropertyGroup>
```

Rules worth enabling as errors for clarity and negative space:

| Rule | Catches |
|---|---|
| `CS8509`, `CS8524` | Non-exhaustive switch expression |
| `CA2016` | `CancellationToken` not forwarded |
| `CA1851` | Multiple enumeration of `IEnumerable` |
| `CA1852` | Type can be sealed |
| `CA2007` | Missing `ConfigureAwait` (libraries only) |
| `CA1031` | Catching general `Exception` |
| `CA2000` | Disposable not disposed |
| `IDE0005` | Unnecessary usings |
| `VSTHRD*` (Microsoft.VisualStudio.Threading.Analyzers) | Sync-over-async, `async void` |

Suppress with a justification (`[SuppressMessage(..., Justification = "...")]`), never blanket in `.editorconfig` without a reason.

---

## Architecture placement in ASP.NET Core

| Layer | Contains | Must not contain |
|---|---|---|
| Endpoint / Controller | HTTP binding, auth attributes, mapping to/from DTOs, status codes | Business rules, EF queries |
| Application / Handler | Use-case orchestration, transactions | HTTP types, SQL, presentation formatting |
| Domain | Entities, value objects, invariants, policies | `DbContext`, `HttpContext`, `ILogger` heavy use |
| Infrastructure | EF Core, HTTP clients, file I/O, message bus | Business decisions |

Do not introduce all four projects for a small service. One project with folders per feature is clearer until the boundaries are actually needed. Do not wrap EF Core in a generic `IRepository<T>` that only forwards `DbSet<T>`; `DbContext` already is a unit of work and repository. Introduce a specific repository when it hides real query policy or enables a meaningful fake.
