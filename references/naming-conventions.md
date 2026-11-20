# Naming Conventions (C# / .NET)

Names are the primary medium through which code communicates. If identifiers are precise and consistent, the code reads like a description of the domain. If they are vague, the reader reconstructs intent on every line.

## The Fundamental Rule

**A name should answer: what does this thing do, contain, or represent?** If the reader needs the implementation to answer that, the name has failed.

---

## .NET casing at a glance

Follow the [.NET Framework Design Guidelines](https://learn.microsoft.com/dotnet/standard/design-guidelines/naming-guidelines) unless the repository's `.editorconfig` says otherwise.

| Element | Casing | Example |
|---|---|---|
| Namespace, type, method, property, event, enum member | PascalCase | `Invoicing`, `InvoiceService`, `IssueAsync`, `DueDate`, `Paid` |
| Interface | `I` + PascalCase | `IPaymentGateway` |
| Type parameter | `T` + PascalCase | `TResult`, `TEntity` |
| Parameter, local | camelCase | `invoiceId`, `dueDate` |
| Private field | `_camelCase` (most common) | `_db`, `_clock` |
| Constant | PascalCase | `MaxRetryCount` |
| Async method | `Async` suffix | `GetOrderAsync` |

Acronyms: two letters stay upper (`IO`, `UI`), longer become Pascal (`Html`, `Json`, `Iban`, `Dhl`).

---

## Methods

### Action methods

Start with a specific verb. Avoid `Handle`, `Process`, `Manage`, `Do` unless the framework dictates it (`IRequestHandler<T>.Handle`).

| Generic (avoid) | Specific (prefer) |
|---|---|
| `HandleUser()` | `DeactivateUser()`, `MergeUserAccounts()` |
| `ProcessPayment()` | `CapturePayment()`, `RefundPayment()` |
| `ManageSession()` | `StartSession()`, `RevokeSession()` |
| `DoRequest()` | `FetchLatestInvoicesAsync()` |

### Query methods

`Get` is the established .NET convention for retrieval (`GetById`, `GetOrderAsync`), so do not fight it in a repository that uses it. Prefer more specific verbs when they add information: `Find` (may return `null`), `Fetch`/`Load` (I/O), `Calculate`/`Compute` (non-trivial computation), `Try` (returns `bool` + `out`).

| Prefix | Implied contract |
|---|---|
| `GetX` | Returns X; throws if missing (or returns a non-null value) |
| `FindX` | Returns `X?`; `null` when absent |
| `TryGetX` | Returns `bool`, X via `[NotNullWhen(true)] out` |
| `CalculateX` | Pure, potentially expensive |
| `EnsureX` | Throws if X is not true (a guard) |

Keep the prefix honest: a `GetX` that sometimes returns `null` breaks the vocabulary.

### Async

Async methods end with `Async`, return `Task`/`Task<T>`/`ValueTask<T>`, and take `CancellationToken ct` last.

---

## Booleans

| Prefix | Use for | Example |
|---|---|---|
| `Is` | State or classification | `IsExpired`, `IsAuthenticated` |
| `Has` | Possession | `HasUnpaidInvoices` |
| `Can` | Capability / permission | `CanRefund` |
| `Should` | Policy | `ShouldRetry` |

Avoid negatives (`IsNotEnabled`, `IsDisabled` paired with `!`). `if (!items.Any())` is fine; do not add an extension just to remove one `!`.

Avoid `flag`, `status`, `check`, `valid`, `result` as boolean names.

---

## Types

### The Name Test

Describe the type's role at its call sites. If the description becomes a list of unrelated policies, the type has accumulated responsibilities.

- ✅ `TokenRefresher`, `InvoiceNumberGenerator`, `RateLimiter`
- ❌ `UserManager` — manages... authentication? profiles? sessions?

### Red flag words

| Red flag | Why | Alternative |
|---|---|---|
| `Manager` | Hides what it owns | Name the owned role, unless the repo uses it consistently |
| `Helper` / `Utils` | Bag of unrelated functions | Name by subject: `IbanFormatting`, `DateRangeExtensions` |
| `Service` (bare) | Says nothing beyond "a class" | `InvoiceIssuer`, `PaymentCapture` — or accept it if the repo uses `XxxService` uniformly per feature |
| `Base` | Premature inheritance | Composition |
| `Common` / `Shared` | "Common to what?" | Name the shared concept |
| `Data` / `Info` / `Dto` everywhere | No behavioral identity | `InvoiceSummary`, `CreateInvoiceRequest` |

### Suffixes with established meaning

`Request`/`Response` (API contracts), `Command`/`Query` (CQRS), `Options` (`IOptions<T>` configuration), `Exception`, `Attribute`, `EventArgs`, `Extensions` (static extension classes), `Handler` (MediatR/message handlers), `Configuration` (EF `IEntityTypeConfiguration<T>`). Use them for those roles only.

### Interfaces

Name the capability, not the implementation: `IPaymentGateway`, not `IStripeService`. If the only implementation would be named `PaymentGateway` and no fake exists, reconsider whether the interface is needed at all (see [testing-and-seams.md](testing-and-seams.md)).

### Enums

Singular for normal enums (`OrderStatus`), plural for `[Flags]` (`FilePermissions`). Members do not repeat the type name:

```csharp
public enum OrderStatus { Draft, Paid, Shipped, Cancelled }   // not OrderStatusDraft
```

---

## Parameters

Name the role, not the type: `Send(byte[] payload, Uri endpoint)`, not `Send(byte[] data, Uri uri)`. Use named arguments at call sites for literals whose meaning is not obvious: `Retry(maxAttempts: 3)`.

---

## Variables and Constants

### Scope-proportional naming

| Scope | Rule | Example |
|---|---|---|
| Loop index | Single letter ok | `for (var i = 0; i < count; i++)` |
| Lambda parameter | Short, but meaningful for non-trivial bodies | `users.Where(u => u.IsActive)` |
| Local | Descriptive | `var unpaidInvoices = ...` |
| Field / property | Clear and specific | `_lastSuccessfulSyncAt` |
| Constant | Fully qualified meaning | `DefaultRequestTimeout` |

### Units in names

When a type does not carry the unit, the name must: `timeoutMs`, `sizeBytes`, `priceCents`. Better: use `TimeSpan`, a `Money` value object, or `DateTimeOffset`, and drop the suffix.

### Magic numbers

```csharp
// Avoid
if (retries >= 3) { ... }
await Task.Delay(300, ct);

// Prefer
private const int MaxRetryCount = 3;
private static readonly TimeSpan SettleDelay = TimeSpan.FromMilliseconds(300);
```
