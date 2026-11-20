# Negative Space Programming

Most code describes the *positive space*: what the program does when everything is as expected. Negative space programming makes the *negative space* explicit too: what must never be true, which states are impossible, which values are out of bounds. The idea is popularized by TigerBeetle's "Tiger Style" and builds on Design by Contract (Meyer) and the fail-fast principle.

The goal is not defensive programming. Defensive code tolerates bad state and limps on ("if null, return empty"). Negative space code **refuses** bad state at the earliest point and makes the bug visible where it was introduced, not three layers later where it corrupts data.

---

## The three layers, in order of preference

1. **Types** — make the invalid state unrepresentable (non-nullable references, value objects, enums, sealed hierarchies, `required` members). Zero runtime cost, checked by the compiler. See [parse-dont-validate.md](parse-dont-validate.md).
2. **Guards on public surfaces** — preconditions callers can violate (`ArgumentNullException.ThrowIfNull`, `ArgumentOutOfRangeException.ThrowIfNegativeOrZero`, `ObjectDisposedException.ThrowIf`). Always on, part of the contract.
3. **Assertions on internal invariants** — conditions only *our own* code can violate. `Debug.Assert` for hot paths and expensive checks; an always-on assert (throwing `UnreachableException` or `InvalidOperationException`) for invariants whose violation would corrupt persistent state.

If a check can be moved one layer up, move it.

---

## Assert the negative space, not just the positive

A function typically checks what it expects. Also check what it does *not* expect.

```csharp
// Positive space only — a negative balance slips through silently
public void Withdraw(Money amount)
{
    _balance -= amount;
}

// Positive and negative space
public void Withdraw(Money amount)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount.Cents);
    if (amount > _balance) throw new InsufficientFundsException(_balance, amount); // expected business failure

    _balance -= amount;

    Debug.Assert(_balance.Cents >= 0, "balance must never go negative");
}
```

The `if` is a real branch: insufficient funds is a valid business outcome a caller handles. The `Debug.Assert` is not a branch: if it fires, the logic above is wrong.

---

## Pair assertions

Whenever two pieces of code must agree, assert the property on *both* sides. A bug then has to slip past two independent checks.

```csharp
public byte[] Encode(Frame frame)
{
    Debug.Assert(frame.Payload.Length <= Frame.MaxPayloadSize);
    var buffer = new byte[Frame.HeaderSize + frame.Payload.Length];
    // ... write header + payload ...
    Debug.Assert(buffer.Length == Frame.HeaderSize + frame.Payload.Length);
    return buffer;
}

public Frame Decode(ReadOnlySpan<byte> buffer)
{
    if (buffer.Length < Frame.HeaderSize) throw new InvalidDataException("truncated frame"); // untrusted input
    var length = BinaryPrimitives.ReadInt32LittleEndian(buffer);
    if (length > Frame.MaxPayloadSize) throw new InvalidDataException("payload too large");
    // ...
}
```

Typical pairs: serialize/deserialize, acquire/release, reserve/commit, write-to-cache/read-from-cache, increment/decrement counters, state transition from/to.

---

## Exhaustiveness: the unreachable default

The most common negative space in C# is the `switch` default. Do not silently return something.

```csharp
// Avoid — a new enum member returns 0 without anyone noticing
decimal TaxRate(Region region) => region switch
{
    Region.Netherlands => 0.21m,
    Region.Belgium => 0.21m,
    _ => 0m,
};

// Prefer — the impossible case is loud
decimal TaxRate(Region region) => region switch
{
    Region.Netherlands => 0.21m,
    Region.Belgium => 0.21m,
    _ => throw new UnreachableException($"Unhandled region {region}"),
};
```

Enable `CS8509`/`CS8524` (non-exhaustive switch) as errors so adding an enum member breaks the build at every consumer.

---

## Assert state transitions

State machines are where negative space bugs live. Assert the allowed *from* states on every transition.

```csharp
public void Ship()
{
    if (Status is not OrderStatus.Paid)
        throw new InvalidOperationException($"Cannot ship order in status {Status}");

    Status = OrderStatus.Shipped;
}
```

Better still, encode the transition in the type so `Ship()` only exists on a `PaidOrder` (see [parse-dont-validate.md](parse-dont-validate.md)).

---

## Bounds on everything

Put a limit on every loop, queue, retry, buffer, and collection that grows from input. An unbounded loop is an unasserted assumption that it terminates.

```csharp
private const int MaxRetryCount = 5;
private const int MaxBatchSize = 1_000;

public async Task ImportAsync(IReadOnlyList<Row> rows, CancellationToken ct)
{
    ArgumentOutOfRangeException.ThrowIfGreaterThan(rows.Count, MaxBatchSize);
    // ...
}
```

Name the limit as a constant so the bound is documentation.

---

## Rules for assertions

- **No side effects.** `Debug.Assert` calls are compiled out of Release builds; `Debug.Assert(queue.TryDequeue(out _))` changes behavior between builds.
- **Never catch an assertion.** No `catch (UnreachableException)`; no blanket `catch (Exception)` that swallows invariant failures. Let them reach the top-level handler and crash/500.
- **Not for untrusted input.** User data, files, network payloads are *expected* to be wrong sometimes. Validate/parse them; do not `Debug.Assert` them.
- **One condition per assertion** so the failure message points at exactly one broken fact. `Debug.Assert(a); Debug.Assert(b);` over `Debug.Assert(a && b)`.
- **Message states the invariant**, not the symptom: `"balance must never go negative"`, not `"error"`.
- **Choose the build mode deliberately.** `Debug.Assert` disappears in Release. For invariants protecting persistent data or money, use an always-on check:

```csharp
internal static class Invariant
{
    public static void Requires([DoesNotReturnIf(false)] bool condition, [CallerArgumentExpression(nameof(condition))] string? expression = null)
    {
        if (!condition) throw new UnreachableException($"Invariant violated: {expression}");
    }
}

Invariant.Requires(ledger.Debits == ledger.Credits);
```

`[CallerArgumentExpression]` puts the failing expression in the message for free; `[DoesNotReturnIf(false)]` informs nullable flow analysis.

- **Assertion density.** A useful heuristic from Tiger Style is roughly two assertions per non-trivial function: one on entry (precondition), one on exit (postcondition) or across a critical step. This is a prompt, not a quota — trivial getters need none, ledger code needs many.

---

## What not to do

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| `if (x == null) return;` deep inside internal code | Hides the bug that produced `null`; caller sees silent no-op | Non-nullable type; or `ArgumentNullException.ThrowIfNull` at the public boundary |
| `catch (Exception) { _logger.LogError(...); }` and continue | Invariant broken, program keeps running on corrupt state | Catch only where you can recover; rethrow otherwise |
| `default: return null;` in a switch | New cases silently fall through | `_ => throw new UnreachableException()` |
| `?? string.Empty` everywhere | Converts "impossible" into "plausible" data | Prove non-null by type, or assert |
| `Debug.Assert(SaveChanges() > 0)` | Side effect disappears in Release | Assign first, assert the variable |
| Assert on HTTP request body | Input is untrusted, not an invariant | Model binding + validation / parsing |

---

## Review questions

1. What must *never* be true after this method runs? Is that stated anywhere?
2. Which `switch`/`if` defaults return a plausible value for an impossible case?
3. Which null checks hide a bug instead of preventing one?
4. Where do two components need to agree, and is the agreement asserted on both sides?
5. Which loops, queues, retries, or batches have no explicit bound?
6. Could this assertion be replaced by a type?
