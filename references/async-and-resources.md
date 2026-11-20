# Async and Resource Ownership

Async code and disposable resources both answer one question: **who owns this lifetime?** Every `Task`, `CancellationToken`, `IDisposable`, and `IAsyncDisposable` needs exactly one owner and one end.

---

## Part 1 — Async

### Async all the way

```csharp
// Avoid — sync-over-async deadlocks in some contexts and wastes threads everywhere
var user = _client.GetUserAsync(id).Result;
_client.SaveAsync(user).Wait();

// Prefer
var user = await _client.GetUserAsync(id, ct);
await _client.SaveAsync(user, ct);
```

`.Result`, `.Wait()`, `.GetAwaiter().GetResult()` in application code are almost always bugs. The only acceptable places are `Main` before async entry existed, or a framework callback that cannot be async — and those should be commented.

### No `async void`

`async void` exceptions cannot be caught by the caller and crash the process. Use `async Task`. Only event handlers (`button.Click += async (s, e) => ...`) may be `async void`, and they should catch and report their own exceptions.

### Fire-and-forget is a design decision, not a shortcut

```csharp
// Avoid — unobserved exception, no lifetime owner, no cancellation
_ = SendAnalyticsAsync(evt);
```

If work must outlive the request, give it an owner: a `Channel<T>` consumed by a `BackgroundService`, or a queue. That owner handles exceptions, cancellation, and shutdown.

### CancellationToken

- Accept a `CancellationToken` as the **last** parameter of every public async method that does I/O: `Task<Order> GetAsync(OrderId id, CancellationToken ct)`.
- **Pass it through** to every awaited call. A token accepted and not forwarded is a lie in the signature. Enable analyzer `CA2016`.
- Do not default it (`= default`) on internal APIs; it makes forgetting to forward invisible. A default is acceptable at the outermost public library surface.
- Use `ct.ThrowIfCancellationRequested()` in CPU loops over large inputs.
- Linked timeouts: `using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct); cts.CancelAfter(timeout);` — and dispose the source.
- Cancellation is not an error: catch `OperationCanceledException` only to stop cleanly, and do not log it as a failure.

### Concurrency

- Independent awaits: `await Task.WhenAll(a, b)` instead of sequential awaits — but only when they are truly independent and the dependency (e.g. a `DbContext`) is thread-safe. A single `DbContext` is **not**.
- Bound parallelism: `Parallel.ForEachAsync(items, new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct }, ...)`, not an unbounded `Select(async ...)` + `WhenAll` over 10,000 items.
- Shared mutable state across awaits needs one owner: `SemaphoreSlim`, a `Channel<T>`, or confinement to one task. `lock` cannot contain `await`.
- Every `await` is a point where other code may have run. Revalidate assumptions (cache contents, entity state) after the await if they matter.

### ConfigureAwait

In library code that may be called from a UI or legacy ASP.NET synchronization context, use `ConfigureAwait(false)`. In ASP.NET Core application code there is no synchronization context; omit it unless the repository mandates it. Follow the repository convention consistently rather than sprinkling it.

### `ValueTask`

Return `ValueTask<T>` only for hot paths that usually complete synchronously and where allocation has been measured. Never await a `ValueTask` twice. Default to `Task`.

---

## Part 2 — IDisposable and IAsyncDisposable

### One owner, one end

The code that **creates** a disposable owns it and disposes it, unless ownership is explicitly transferred (returned to the caller, or handed to a DI container).

```csharp
await using var connection = new NpgsqlConnection(_connectionString);
await connection.OpenAsync(ct);
await using var transaction = await connection.BeginTransactionAsync(ct);
// ... work ...
await transaction.CommitAsync(ct);
```

- Prefer `using` / `await using` declarations over nested `using` blocks — flat, and disposal order is reverse declaration order.
- Use `await using` for `IAsyncDisposable` (DbContext, streams, connections); synchronous dispose of async resources may block.
- Do not dispose what you did not create. An injected `HttpClient`, `DbContext`, or `ILogger` belongs to the container.

### DI lifetimes are ownership

| Lifetime | Owner | Pitfall |
|---|---|---|
| Transient | Container disposes at scope end | Transient `IDisposable` resolved from root leaks |
| Scoped | Disposed when request/scope ends | Injecting scoped into singleton ("captive dependency") — enable `ValidateScopes` |
| Singleton | App lifetime | Must be thread-safe; must not hold scoped services |

Inside a `BackgroundService` or singleton, create a scope per unit of work: `await using var scope = _scopeFactory.CreateAsyncScope();`.

### HttpClient

Never `new HttpClient()` per call (socket exhaustion) and never one static instance without `PooledConnectionLifetime` (stale DNS). Use `IHttpClientFactory` / typed clients.

### Implementing IDisposable

Implement it only when the type **owns** an unmanaged resource or another disposable. Seal the class and skip the full dispose pattern unless you truly have a finalizer:

```csharp
public sealed class ReportWriter : IAsyncDisposable
{
    private readonly Stream _stream;
    private bool _disposed;

    public ReportWriter(string path) => _stream = File.Create(path);

    public async Task WriteAsync(Report report, CancellationToken ct)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        // ...
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        _disposed = true;
        await _stream.DisposeAsync();
    }
}
```

`ObjectDisposedException.ThrowIf` is the negative-space assertion for "use after dispose".

---

## Review questions

1. Any `.Result`, `.Wait()`, `async void`, or `_ = SomethingAsync()`?
2. Is every accepted `CancellationToken` forwarded?
3. Does every disposable have exactly one owner that disposes it on every path?
4. Are scoped services captured by singletons or background services?
5. Is parallelism bounded?
6. Is shared state mutated across `await` points without an owner?
