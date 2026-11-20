# File Organization and Named Constants

Two decisions shape readability before a reader looks at a single method: **how code is split into files and folders**, and **how fixed values are named and scoped**.

---

## Part 1 — One Cohesive Subject Per File

### The rule

**A file has one discoverable subject, and its filename names that subject.** In C# the default is one top-level type per file, named after the type (`InvoiceService.cs`). StyleCop SA1402/SA1649 enforce this; follow the repository if it does.

Legitimate exceptions — co-locate when the declarations form one concept and change together:

- a closed record hierarchy (`PaymentState` + its nested sealed records);
- a request record and its validator, if the repository colocates them;
- a small private nested type used only by its owner;
- a generic and non-generic pair (`Result.cs` with `Result` and `Result<T>`), using `Result{T}.cs` naming only if that is the house style.

Split when a declaration has independent consumers or reasons to change. Do not split a type across `partial` files merely to hide its size — that is accumulation in disguise. `partial` is for generated code (source generators, designers, `[GeneratedRegex]`, `[LoggerMessage]`).

### Namespaces mirror folders

Use file-scoped namespaces (`namespace Acme.Invoicing;`) and keep namespace = project root namespace + folder path. A namespace that does not match the folder makes "where is this type?" unanswerable. IDE0130 enforces it.

### Folder organization: by feature, not by kind

```
src/
  Acme.Billing/
    Invoicing/                     ← feature folder
      Invoice.cs
      InvoiceLine.cs
      InvoiceId.cs
      IssueInvoice.cs              ← endpoint/handler for one use case
      InvoiceNumberGenerator.cs
      InvoicingServiceCollectionExtensions.cs
    Payments/
      Payment.cs
      PaymentState.cs
      CapturePayment.cs
    Infrastructure/
      AppDbContext.cs
      Configurations/
        InvoiceConfiguration.cs
  Acme.Billing.Tests/
    Invoicing/
      InvoiceTests.cs
      IssueInvoiceTests.cs
```

Prefer feature folders (vertical slices) over technical folders (`Controllers/`, `Services/`, `Repositories/`, `Models/`, `Interfaces/`). With technical folders, one change touches five folders, and `Services/` becomes a grab-bag. Follow the repository's existing structure if it is consistent; do not reorganize as part of a behavior change.

Avoid `Helpers/`, `Utils/`, `Common/`, `Shared/` folders that collect unrelated code. Name by subject.

### Projects are boundaries, not folders

Create a separate project (`.csproj`) only for a real boundary: a deployable unit, a reusable package, or an enforced dependency rule (domain must not reference infrastructure). Ten projects for a small API add build time and navigation cost without adding clarity. Use `internal` + `InternalsVisibleTo` for tests rather than making everything `public`.

### Diagnostic

| Symptom | Fix |
|---|---|
| File name doesn't match its type | Rename file |
| Namespace doesn't match folder | Align namespace (IDE0130) |
| Several unrelated types in one file | Split per type |
| `partial` used to hide a large class | Split by reason to change |
| `Helpers.cs` / `Utils/` keeps growing | Name files by subject; move members to their owners |
| One feature change touches `Controllers/`, `Services/`, `Repositories/`, `Models/` | Consider feature folders for new code |
| Everything `public` in an application project | Default to `internal`; expose deliberately |

---

## Part 2 — Named Constants

A literal with meaning — a timeout, a retry ceiling, a limit, a route, a header name — is a **named constant at the narrowest scope that serves its readers**, declared at the top of its type.

```csharp
public sealed class ImportJob
{
    private const int MaxBatchSize = 1_000;
    private const int MaxRetryCount = 3;
    private static readonly TimeSpan RetryDelay = TimeSpan.FromSeconds(2);

    // ...
}
```

### Rules

- `const` for primitives and strings known at compile time; `static readonly` for `TimeSpan`, `Uri`, arrays, and objects.
- Scope: method-local `const` → `private` → `internal` → `public`. Don't make a constant `public` only because it's convenient.
- `public const` values are copied into consuming assemblies at compile time. For public values that might change, use `public static readonly`.
- Use digit separators (`1_000_000`) and real units (`TimeSpan.FromMinutes(5)`, not `300000`).
- Name a literal when the name carries domain meaning, centralizes a shared contract, or prevents inconsistent edits. Obvious local values (`0`, `1`, `string.Empty`) stay inline.
- Bounds are negative space: every named `MaxXxx` should be checked somewhere (see [negative-space-programming.md](negative-space-programming.md)).

### Registries for wire strings

Routes, header names, claim types, queue names, and cache keys that appear in several places belong in one static class:

```csharp
internal static class Headers
{
    public const string CorrelationId = "X-Correlation-Id";
    public const string IdempotencyKey = "Idempotency-Key";
}
```

### Configuration vs constant

If the value differs per environment or should be tunable without a deploy, it is configuration (`IOptions<T>` with validation), not a constant. If it is a law of the domain (VAT digits, max invoice lines per spec), it is a constant.

### Diagnostic

| Symptom | Fix |
|---|---|
| Bare number/string with non-obvious meaning | Named constant at top of type |
| Same literal in multiple places | One constant; one source of truth |
| `public const` used by one class | Make it `private` |
| `TimeSpan` built from raw milliseconds | `TimeSpan.FromSeconds(...)` in a `static readonly` |
| Environment-specific value as constant | Move to `IOptions<T>` |
