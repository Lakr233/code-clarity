---
name: code-clarity
description: Write, review, and simplify C#/.NET code so behavior has one owner, state is minimal and derivable, invalid states are unrepresentable, invariants are asserted (negative space programming), failures are grouped by real recovery behavior, APIs expose only necessary concepts, control flow stays flat, and tests protect observable outcomes without forcing production abstractions. Use when C# code grows through incremental feedback, a diff adds many states/exceptions/branches/types/interfaces, nullability or validation is scattered, async or IDisposable ownership is unclear, naming or flow is hard to follow, DI registrations and mocks multiply, Blazor components mutate parameters or mix business logic into markup, or a refactor should reduce concepts while preserving behavior and repository conventions.
---

# Code Clarity (C# / .NET)

Optimize for the amount of code and state a maintainer must understand, not for the apparent completeness of each local branch.

Two complementary disciplines run through this skill:

- **Positive space** — the code that does the work: the happy path, the owner, the one canonical implementation. Keep it small and flat.
- **Negative space** — what must *never* be true: invariants, preconditions, postconditions, impossible states. Make it explicit through types first and assertions second, so violations fail fast and loudly instead of propagating silently. See [references/negative-space-programming.md](references/negative-space-programming.md).

## Working method

### 1. Establish the real contract

Read the repository instructions (`.editorconfig`, `Directory.Build.props`, analyzers, `README`, existing tests) and representative surrounding code before editing. State:

- the invariant that must remain true (and what must never happen);
- the normal successful path;
- the failures that require different recovery behavior;
- the facts that are already authoritative;
- the requested scope and explicit non-goals.

Do not begin from the line mentioned in feedback. Begin from the abstraction that owns the behavior. A local complaint often reveals that the owner or invariant is wrong.

### 2. Inventory every proposed concept

Count concepts, not only lines. Treat each of these as a cost that needs a reason:

- field, property, serialized member, cache entry, or duplicated value;
- enum member, status, phase, `bool` flag, or nullable used as state;
- custom exception type, error code, `Result` variant, or sentinel value;
- public method, interface member, event, delegate, or wrapper;
- branch, retry path, rollback path, compatibility path, or special case;
- class, record, struct, service, manager, helper, extension class, or DTO;
- DI registration, test seam, fake, mock setup, or duplicated test case.

For each concept ask:

1. Can it be derived from existing authoritative facts?
2. Does a caller behave differently because it exists?
3. Can it be `private`/`internal` to the owner instead of crossing a boundary?
4. Can it be folded into the normal path or an existing type?
5. Would deleting it make a real supported scenario ambiguous or unsafe?

If the last answer is no, delete or avoid the concept.

### 3. Make illegal states unrepresentable, then assert the rest

Push invariants into types before writing checks. **Parse, don't validate**: convert untrusted input (HTTP, JSON, config, database, user input) into a type that can only hold valid values *once*, at the boundary, then pass that type inward. Internal code then never re-checks what the type already guarantees. See [references/parse-dont-validate.md](references/parse-dont-validate.md).

What the type system cannot express, assert. Every non-trivial function states its preconditions on entry and, where cheap, its postconditions on exit. Assert both the positive space (what we expect) and the negative space (what we do not expect). A violated assertion is a bug in *our* code — crash and fix it, never catch-and-continue.

Distinguish clearly:

| Situation | Response |
|---|---|
| Invalid external input (user, network, file) | Parse at the boundary; return a validation result or throw `ArgumentException`-family / domain exception the caller handles |
| Expected operational failure (timeout, not found, conflict) | Model it for the caller that recovers: exception type the caller catches, or `Result`/`TryX` when failure is a normal outcome |
| Programmer error / broken invariant | Guard (`ArgumentNullException.ThrowIfNull`, `ArgumentOutOfRangeException.ThrowIfNegative`) on public surfaces; `Debug.Assert` or `UnreachableException` internally. Never catch. |

### 4. Collapse failures by recovery behavior

Build failure equivalence classes before inventing exception types. If several failures produce the same return value, fallback, or caller action, handle them through one path. Preserve a distinct exception type or result variant only when a caller must choose different behavior.

Do not turn diagnostic detail into control state. Log the underlying exception (with `ILogger` structured properties) while keeping the control model small. Never `catch (Exception)` to swallow; catch at the boundary that can actually recover or translate. See [references/nullability-and-exceptions.md](references/nullability-and-exceptions.md).

### 5. Choose the smallest honest representation

- Derive values (computed properties, expression-bodied members) instead of storing duplicates.
- Use a `bool` for one independent yes/no fact.
- Use an `enum` for a closed set of payload-free states consumed by an exhaustive `switch` expression; use a sealed record hierarchy when cases carry different data.
- Use `record` / `readonly record struct` for values; use `class` only when identity, lifecycle, or resource ownership matters. Seal classes by default.
- Keep intermediate values and lambdas local; do not promote implementation steps into fields or public members.
- Store only facts needed after the current computation that cannot be derived safely.
- Add an error code or exception type only when another component branches on it.
- Add an interface only where behavior genuinely varies — not because every class "should" have an `IFoo`.

Closed modeling is not a command to create an enum for every finite set. The type must remove invalid combinations or enable exhaustive behavior; otherwise it is ceremony.

### 6. Implement one path with one owner

Prefer one parameterized or composable implementation over parallel mode-specific implementations when the mechanics are the same. Put decisions in the calling layer and mechanics in the method or type that owns them.

Keep the source of truth with its owner: interpret, validate, and derive a value where its inputs are owned, then expose the smallest result callers need. Do not expose raw fields so each caller can reconstruct the same predicate, classification, or calculation independently.

Keep the happy path flat with guard clauses. Extract a method when it names a coherent operation or hides a lower abstraction level—not merely to reduce line count. Avoid pass-through services whose only effect is to enlarge the DI graph.

Keep a complete statement, call, LINQ chain, or condition on one line when it remains locally readable. Break at semantic groups (one LINQ operator per line for long queries, one clause per line for complex conditions) only when the unwrapped form is hard to scan. Keep indentation honest: depth equals nesting, stay within two or three levels in a method body, and indent continuation lines one level instead of aligning to a column. For indentation, line width and formatter-owned style, read [references/formatting-consistency.md](references/formatting-consistency.md).

Apply the grain of C#:

- Enable `<Nullable>enable</Nullable>` and treat nullable warnings as errors. `string?` is a statement that absence is legal; `!` (null-forgiving) is an unproven assertion — justify or remove it.
- Prefer `switch` expressions with exhaustive patterns over `if`/`else` ladders; let the compiler warn on missing cases, and throw `UnreachableException` for the truly impossible default.
- Use `required` and `init` members, primary constructors, and records to make construction complete and immutable.
- Async all the way: `async Task`, never `async void` (except event handlers); pass `CancellationToken` as the last parameter; `await using` for `IAsyncDisposable`. See [references/async-and-resources.md](references/async-and-resources.md).
- Use `IEnumerable<T>` returns only for genuinely lazy sequences; return `IReadOnlyList<T>` / `IReadOnlyCollection<T>` for materialized data so callers don't enumerate twice.
- Prefer composition and a concrete dependency over inheritance and an interface until a second implementation exists.

For naming, control flow, methods, types, files, and local style, load only the relevant references:

- [references/negative-space-programming.md](references/negative-space-programming.md) — assertions, invariants, pair assertions, fail fast
- [references/parse-dont-validate.md](references/parse-dont-validate.md) — value objects, smart constructors, illegal states unrepresentable
- [references/nullability-and-exceptions.md](references/nullability-and-exceptions.md) — NRT, guard clauses, exceptions vs `Result`
- [references/async-and-resources.md](references/async-and-resources.md) — `CancellationToken`, `IDisposable`, `await using`, async pitfalls
- [references/csharp-and-dotnet.md](references/csharp-and-dotnet.md) — language idioms, records, pattern matching, DI, analyzers
- [references/blazor.md](references/blazor.md) — component state ownership, parameters vs `EventCallback`, markup vs logic, lifecycle, disposal, bUnit
- [references/naming-conventions.md](references/naming-conventions.md)
- [references/early-return.md](references/early-return.md)
- [references/function-design.md](references/function-design.md)
- [references/abstraction-levels.md](references/abstraction-levels.md)
- [references/class-struct-design.md](references/class-struct-design.md)
- [references/file-organization.md](references/file-organization.md)
- [references/formatting-consistency.md](references/formatting-consistency.md)
- [references/repository-conventions.md](references/repository-conventions.md)
- [references/testing-and-seams.md](references/testing-and-seams.md)

### 7. Test behavior classes, not implementation inventory

Protect each externally distinct outcome and each dangerous arithmetic/resource boundary. Do not create one test per private branch when multiple branches intentionally converge on the same fallback.

Prefer:

- a small unit test for pure policy (domain types, value objects, parsers);
- a test at the real abstraction boundary (e.g. `WebApplicationFactory`, a real database via Testcontainers);
- a regression test for an observed failure;
- a compatibility test only when multiple input or API versions are intentionally supported.

Assertions in production code complement tests: tests exercise paths, assertions check that every exercised path keeps the invariants. A test run that never trips an assertion is stronger evidence than one that only checks return values.

If every fake or `Mock<T>` setup gains a no-op member after a production interface change, reconsider the interface before updating the fakes.

### 8. Run a deletion pass after correctness

Reread the complete changed context, not only the edited lines. Then inspect the diff and ask:

- Which field can now be computed?
- Which cases share the same response?
- Which wrapper, service, or interface only forwards a call?
- Which null check guards a value the type already proves non-null?
- Which validation repeats a check a parsed type already guarantees?
- Which branch protects an unsupported or unactionable scenario?
- Which comments narrate code instead of preserving rationale?
- Which tests assert mock choreography instead of behavior?
- Which compatibility path has no active consumer?

Delete first; rename or document what remains.

## Decision gates

### Store state only when all are true

1. The value is needed beyond the current computation.
2. It cannot be derived from one authoritative source.
3. Its writer and transition rules have one clear owner.
4. A reader needs the stored distinction.

### Add an exception type / error code only when all are true

1. At least one caller catches or matches it structurally.
2. That caller takes a distinct action.
3. The distinction is stable enough to become API surface.
4. The message plus an existing exception type is insufficient.

Otherwise throw an existing BCL exception (`InvalidOperationException`, `ArgumentException`, …) or wrap with useful context via `innerException`.

### Add a branch only when all are true

1. The condition can occur for valid program inputs or dependency results.
2. It is detectable with facts available at that point.
3. The response differs from an existing path.
4. The response improves correctness or observable behavior.
5. The branch can be verified proportionally to its risk.

If the condition *cannot* occur for valid inputs, it is not a branch — it is an assertion.

### Add an assertion when any is true

1. A function has a precondition the parameter types do not express.
2. Two pieces of code must agree (write/read, encode/decode, reserve/release) — assert on both sides (pair assertion).
3. An arithmetic, index, or capacity boundary could silently corrupt data.
4. A `switch` default or state transition is believed unreachable.

Assertions must be side-effect free and must not replace input validation of untrusted data.

### Add an abstraction only when it hides more complexity than it exposes

A useful module offers a small interface over substantial policy, state, or mechanism. A shallow wrapper, `IFooService` with one implementation and no fake, a one-method forwarding class, or a DTO that merely renames fields increases cognitive load.

## Review output

Lead with concrete findings. For a large or feedback-driven diff, use this classification:

| Verdict | Meaning |
|---|---|
| Delete | No distinct supported behavior depends on it |
| Derive | Existing authoritative facts already determine it |
| Fold | It shares ownership or recovery behavior with an existing path |
| Type | The check belongs in a type (value object, non-nullable, enum) instead of runtime code |
| Assert | The condition is an invariant; make it an assertion instead of a branch or silent fallback |
| Keep | It protects a real boundary and changes behavior |
| Prove | Plausible, but evidence is required before paying permanent complexity |

For each finding, cite the owning code and explain the behavior change, not just the style preference. Do not use a clarity score as a substitute for evidence.

## Final standard

The result should make these questions easy to answer:

- Where is the one authoritative state?
- What are the few supported lifecycle states, and which states are impossible by construction?
- Which invariants are asserted, and where?
- Which failures actually change recovery?
- Which type owns each transition, `IDisposable`, and `CancellationToken`?
- What can be deleted without changing observable behavior?
- Do the tests protect the contract rather than the implementation shape?

Clarity is achieved when the normal path is obvious, the negative space is explicit, exceptional paths are few and justified, and every remaining concept earns its place.
