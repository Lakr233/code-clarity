# Repository Conventions

Code clarity is partly universal and partly local. Universal clarity covers things like intention-revealing names, early returns, and single-responsibility types. Local clarity is what makes a change feel native to a specific repository: the file split strategy, naming suffixes, comment tone, layout style, and the small habits that experienced contributors recognize immediately.

Ignoring local conventions is one of the easiest ways to make code technically cleaner but practically harder to maintain.

---

## The Local Clarity Principle

**A change is clear when it reads like the surrounding good code.**

Before suggesting refactors, sample a few representative files and answer:

- How are large types split across files?
- What suffixes are used repeatedly in filenames and type names?
- Are booleans consistently named with `is/has/can/should`?
- Does the repository prefer `guard` / early-return or deeply structured branching?
- Are comments sparse and operational, or explanatory and tutorial-like?
- Is UI built with one dominant layout system, or with a stable hybrid pattern?

If the repository already has a consistent answer, preserve it unless it is creating real bugs or serious maintenance cost.

---

## What To Infer First

### File organization patterns

Examples of repository-local patterns:

- feature folders (`Invoicing/IssueInvoice.cs`) vs technical folders (`Controllers/`, `Services/`)
- one type per file, or request + validator + handler colocated per use case
- `XxxServiceCollectionExtensions` for DI registration per feature
- EF Core `IEntityTypeConfiguration<T>` classes vs attributes on entities
- test project layout mirroring the production folders

If a codebase consistently uses vertical slices with one file per use case, adding a new feature as a method on a shared `XxxService` reduces clarity even if the logic itself is clean.

### Naming vocabulary

Look for repeated suffixes and role words:

- `Manager`
- `Service`
- `Handler`
- `Store`
- `Provider`
- `Command` / `Query`

Generic advice like “never use `Manager`” is weaker than repository-aware advice like “this repository uses `Manager` consistently for stateful coordination types, so keep the term unless the responsibility is actually unclear.”

### Control-flow style

Some codebases read best when written as a flat sequence of guard clauses. Others use symmetrical `if/else` branches intentionally. Match the dominant local style where it is already working well.

### Error and validation style

Look for the established answers before adding your own:

- exceptions + ProblemDetails middleware, or a `Result` type, or both (and where each is used);
- FluentValidation, DataAnnotations, or parsing in the handler;
- `Debug.Assert`, a custom `Invariant`/`Guard` helper, or BCL throw helpers;
- Moq vs NSubstitute vs hand-written fakes; xUnit vs NUnit vs MSTest.

Introducing a second error model or validation library into a repository that already has one adds a concept every maintainer must learn.

### Comment and log tone

Comments can be:

- terse rationale only
- dense explanatory guidance
- architectural notes
- operational warnings

Logs can be:

- short and lowercase
- structured and machine-friendly
- verbose and tutorial-like

In .NET, also check whether logging uses `[LoggerMessage]` source generators or plain `ILogger` extension calls, and which structured property names are standard.

New code should sound like it belongs.

---

## Review Heuristic

When reviewing clarity, use this order:

1. Fix names that are objectively misleading.
2. Fix control flow that hides the happy path.
3. Fix abstraction-level mixing.
4. Fix responsibility leaks in functions and types.
5. Then align the result with repository-local conventions.

This ordering matters. Local consistency should not preserve genuinely confusing code, but once a change is clear in principle, it should be made locally fluent.

---

## Examples

| Situation | Repository-blind suggestion | Repository-aware suggestion |
|----------|-----------------------------|-----------------------------|
| Large controller | “Introduce MediatR and CQRS” | If the repository uses plain application services, extract a focused service per use case and preserve the local architecture |
| Stateful coordinator type | “Rename every `Manager` to `Service`” | Keep `Manager` if the repository uses it for mutable orchestration and the name is already well-understood |
| Interface per class | “Delete every single-implementation interface” | If the repository mandates `IFoo` for DI/mocking everywhere, flag the cost but don't remove them piecemeal in a feature PR |
| Data access | “Wrap EF Core in a generic repository” | If the repository uses `DbContext` directly in handlers, keep doing so; add a specific repository only for real query policy |
| Comments | “Add comments to every block” | Match the repository's comment density and keep comments focused on rationale |
| Error handling | “Never throw, always return `Result`” | Respect the repository's split between exceptions and results; respect intentional fail-fast invariants (`UnreachableException`, `Debug.Assert`) protecting impossible states |

---

## Practical Rule

Do not ask only, “Is this code clear in isolation?”

Also ask, “Will a maintainer who knows this repository read this change as obviously belonging here?”

The highest clarity is achieved when both answers are yes.
