# Code Clarity (C# / .NET)

A Claude skill for writing and reviewing readable, intention-revealing **C# / .NET** code: minimal derivable state, invalid states made unrepresentable, invariants asserted (negative space programming), failures grouped by recovery behavior, flat control flow, consistent abstraction levels, focused files, formatter-owned style, repository conventions, and testable seams without interface-per-class ceremony.

Forked from a multi-language (Swift / Go / Objective-C / TypeScript) version and rebuilt for C#.

## What This Covers

| Topic | Description |
|-------|-------------|
| **Negative Space Programming** | Asserting what must never be true: preconditions, postconditions, pair assertions, unreachable defaults, bounds, `Debug.Assert` vs always-on invariants (Tiger Style, Design by Contract) |
| **Parse, Don't Validate** | Value objects with smart constructors, strongly-typed IDs, closed record hierarchies, state-specific types — illegal states unrepresentable |
| **Nullability & Exceptions** | Nullable reference types, BCL guard helpers, exceptions vs `TryX` vs `Result`, catching rules, logging once |
| **Async & Resources** | Async all the way, `CancellationToken` forwarding, no `async void`, `IDisposable`/`await using` ownership, DI lifetimes |
| **C# & .NET Idioms** | Records vs sealed classes, pattern matching, LINQ, DI without ceremony, `TimeProvider`, analyzers as enforcement, ASP.NET Core layering |
| **Blazor Components** | Parameter vs state ownership, `EventCallback` up, markup shows / code decides, `RenderFragment` slots over flags, lifecycle, disposal, bUnit |
| **Naming Conventions** | .NET casing, `Get`/`Find`/`TryGet` vocabulary, `Async` suffix, red-flag type names |
| **Early Return** | Guard clauses, pattern-based guards that bind, when not to use them |
| **Method Design** | One contract, one abstraction level, strong-typed parameters, return over mutate |
| **Abstraction Levels** | Endpoint / application / domain / infrastructure separation, step-down rule |
| **Class & Record Design** | Ownership, invariants inside the type, record vs class vs struct, accumulation |
| **File Organization** | One type per file, namespaces = folders, feature folders, named constants |
| **Formatting Consistency** | `.editorconfig` + `dotnet format` / CSharpier, structural indentation, comfortable line width, CI gate |
| **Repository Conventions** | Inferring and preserving the local house style |
| **Dependencies & Test Seams** | One canonical implementation, seams only where behavior varies, fakes over mocks, Testcontainers / `WebApplicationFactory` |

## Install

Place `SKILL.md` and the `references/` folder in a skills directory (e.g. `~/.claude/skills/code-clarity/`) or upload it as a skill in Claude.

## Usage

The skill activates when you ask Claude to:

- Review or simplify a C# change by removing unnecessary state, exceptions, branches, interfaces, or DI registrations
- Add or review invariants, guard clauses, and assertions
- Replace primitive obsession or boolean piles with value objects, records, or a closed state hierarchy
- Clean up nullability (`!` operators, redundant null checks, nullable state machines)
- Review async code for missing `CancellationToken`s, `async void`, sync-over-async, or leaked disposables
- Clean up Blazor components that mutate parameters, sync state by hand, or hide business rules in markup
- Split a grab-bag service or `Helpers` class
- Reduce interface-per-class and mock-heavy tests
- Set up `.editorconfig`, analyzers, and a formatting CI gate

**Example prompts:**

```
Review this handler with code-clarity: what can be deleted, derived, or turned into a type?
```

```
Apply negative space programming to this OrderService — which invariants are missing?
```

```
This service has 7 constructor dependencies and every test uses Moq. Simplify.
```

## Skill Structure

```
code-clarity/
├── SKILL.md                              # Working method, decision gates, review verdicts
└── references/
    ├── negative-space-programming.md     # Assertions, invariants, pair assertions, bounds
    ├── parse-dont-validate.md            # Value objects, typed IDs, closed hierarchies
    ├── nullability-and-exceptions.md     # NRT, guards, exceptions vs Result
    ├── async-and-resources.md            # CancellationToken, IDisposable, DI lifetimes
    ├── csharp-and-dotnet.md              # Language idioms, DI, analyzers, layering
    ├── blazor.md                         # Component state, callbacks, lifecycle, bUnit
    ├── naming-conventions.md
    ├── early-return.md
    ├── function-design.md
    ├── abstraction-levels.md
    ├── class-struct-design.md
    ├── file-organization.md
    ├── formatting-consistency.md
    ├── repository-conventions.md
    └── testing-and-seams.md
```

## Design Philosophy

- **Concept reduction** — every field, enum member, exception type, interface, and branch is a cost that must earn its place
- **Negative space programming** — make what must never happen explicit, through types first and assertions second; fail fast instead of limping on (TigerBeetle's Tiger Style, Meyer's Design by Contract)
- **Parse, don't validate** — check untrusted input once at the boundary and keep the proof in the type (Alexis King; "make illegal states unrepresentable", Yaron Minsky)
- **Deep modules** — small interfaces over substantial behavior; no pass-through services (Ousterhout)
- **Tool-owned style** — `.editorconfig`, `dotnet format`, and analyzers decide mechanical style so reviews focus on design
- **Repository-local clarity** — preserve the codebase's existing vocabulary and structure before introducing new patterns

## License

MIT — see [LICENSE](LICENSE).
