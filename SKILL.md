---
name: code-clarity
description: Write, review, and simplify Go, Swift, and TypeScript code so behavior has one owner, state is minimal and derivable, failures are grouped by real recovery behavior, APIs expose only necessary concepts, control flow stays flat, and tests protect observable outcomes without forcing production abstractions. Use when code grows through incremental feedback, a diff adds many states/errors/branches/types, naming or flow is hard to follow, seams and mocks multiply, or a refactor should reduce concepts while preserving behavior and repository conventions.
---

# Code Clarity

Optimize for the amount of code and state a maintainer must understand, not for the apparent completeness of each local branch.

## Working method

### 1. Establish the real contract

Read the repository instructions and representative surrounding code before editing. State:

- the invariant that must remain true;
- the normal successful path;
- the failures that require different recovery behavior;
- the facts that are already authoritative;
- the requested scope and explicit non-goals.

Do not begin from the line mentioned in feedback. Begin from the abstraction that owns the behavior. A local complaint often reveals that the owner or invariant is wrong.

### 2. Inventory every proposed concept

Count concepts, not only lines. Treat each of these as a cost that needs a reason:

- stored property, serialized field, cache entry, or duplicated value;
- enum/union case, status, phase, flag, or optional used as state;
- sentinel, typed error, error code, or result variant;
- public method, protocol requirement, callback, interface, or wrapper;
- branch, retry path, rollback path, compatibility path, or special case;
- type, class, actor, manager, helper, or result object;
- test seam, fake method, fixture variant, or duplicated test matrix row.

For each concept ask:

1. Can it be derived from existing authoritative facts?
2. Does a caller behave differently because it exists?
3. Can it be private to the owner instead of crossing a boundary?
4. Can it be folded into the normal path or an existing type?
5. Would deleting it make a real supported scenario ambiguous or unsafe?

If the last answer is no, delete or avoid the concept.

### 3. Collapse failures by recovery behavior

Build failure equivalence classes before inventing error types. If several failures produce the same return value, fallback, or caller action, handle them through one path. Preserve a distinct error only when a caller must choose different behavior.

Do not turn diagnostic detail into control state. Log the underlying error while keeping the control model small.

### 4. Choose the smallest honest representation

- Derive values instead of storing duplicates.
- Use a boolean for one independent yes/no fact.
- Use an enum or discriminated union for mutually exclusive states with exhaustive consumers.
- Use a value type for data and a reference/lifecycle type only when identity or ownership matters.
- Keep intermediate values and callbacks local; do not promote implementation steps into public state.
- Store only facts needed after the current computation that cannot be derived safely.
- Add a machine-readable error code only when another component branches on it.
- Add a seam only where behavior genuinely varies.

Closed modeling is not a command to create an enum for every finite set. The type must remove invalid combinations or enable exhaustive behavior; otherwise it is ceremony.

### 5. Implement one path with one owner

Prefer one parameterized or composable implementation over parallel mode-specific implementations when the mechanics are the same. Put decisions in the calling layer and mechanics in the function or type that owns them.

Keep the source of truth with its owner: interpret, validate, and derive a value where its inputs are owned, then expose the smallest result callers need. Do not export raw fields so each caller can reconstruct the same predicate, classification, or calculation independently. “Produce and consume together” means one ownership boundary and one canonical derivation, not necessarily one physical file; callers may pass along or display the result, but must not redefine it.

Keep the happy path flat. Extract a function when it names a coherent operation or hides a lower abstraction level—not merely to reduce line count. Avoid pass-through methods whose only effect is to enlarge an interface.

Apply the language's own grain:

- **Go:** Return concrete types and define small interfaces at the consumer. Create a sentinel or typed error only when callers use `errors.Is`/`errors.As` to choose different behavior; wrapping an error makes it API. Keep `context.Context` explicit and first. Handle errors before the normal path and avoid result structs that only rename a short, unambiguous return tuple.
- **Swift:** Use structs for values and classes/actors for identity, resource ownership, or isolated mutation. Use an enum when cases are mutually exclusive and switched exhaustively; keep an independent binary fact as a boolean. Treat every `await` inside an actor as a point where assumptions may need revalidation. Use a protocol for a shared capability with real substitutes, a closure for one local operation that genuinely varies, and a concrete dependency when nothing varies.
- **TypeScript:** Use a discriminated union only for variants consumers handle differently, and use `never` for exhaustive switches. Derive secondary values from one typed source instead of mirroring flags. Narrow or validate `unknown` input once at the boundary; do not repeatedly validate internal typed values. Use `satisfies` for complete mappings without widening useful inference.

For naming, control flow, functions, files, and local style, load only the relevant existing references:

- [references/naming-conventions.md](references/naming-conventions.md)
- [references/early-return.md](references/early-return.md)
- [references/function-design.md](references/function-design.md)
- [references/abstraction-levels.md](references/abstraction-levels.md)
- [references/class-struct-design.md](references/class-struct-design.md)
- [references/file-organization.md](references/file-organization.md)
- [references/formatting-consistency.md](references/formatting-consistency.md)
- [references/repository-conventions.md](references/repository-conventions.md)
- [references/testing-and-seams.md](references/testing-and-seams.md)

### 6. Test behavior classes, not implementation inventory

Protect each externally distinct outcome and each dangerous arithmetic/resource boundary. Do not create one test per private branch when multiple branches intentionally converge on the same fallback.

Prefer:

- a small unit test for pure policy;
- a test at the real abstraction boundary;
- a regression test for an observed failure;
- a compatibility test only when multiple input or API versions are intentionally supported.

If every fake gains a no-op method after a production interface change, reconsider the interface before updating the fakes.

### 7. Run a deletion pass after correctness

Reread the complete changed context, not only the edited lines. Then inspect the diff and ask:

- Which field can now be computed?
- Which cases share the same response?
- Which wrapper only forwards a call?
- Which branch protects an unsupported or unactionable scenario?
- Which comments narrate code instead of preserving rationale?
- Which tests assert internal choreography instead of behavior?
- Which compatibility path has no active consumer?

Delete first; rename or document what remains.

## Decision gates

### Store state only when all are true

1. The value is needed beyond the current computation.
2. It cannot be derived from one authoritative source.
3. Its writer and transition rules have one clear owner.
4. A reader needs the stored distinction.

### Add an error type/code only when all are true

1. At least one caller identifies it structurally.
2. That caller takes a distinct action.
3. The distinction is stable enough to become API surface.
4. Diagnostic text alone is insufficient.

Otherwise return or wrap an ordinary error with useful context.

### Add a branch only when all are true

1. The condition can occur for valid program inputs or dependency results.
2. It is detectable with facts available at that point.
3. The response differs from an existing path.
4. The response improves correctness or observable behavior.
5. The branch can be verified proportionally to its risk.

### Add an abstraction only when it hides more complexity than it exposes

A useful module offers a small interface over substantial policy, state, or mechanism. A shallow wrapper, one-method forwarding type, speculative protocol, or result object that merely renames fields increases cognitive load.

## Review output

Lead with concrete findings. For a large or feedback-driven diff, use this classification:

| Verdict | Meaning |
|---|---|
| Delete | No distinct supported behavior depends on it |
| Derive | Existing authoritative facts already determine it |
| Fold | It shares ownership or recovery behavior with an existing path |
| Keep | It protects a real boundary and changes behavior |
| Prove | Plausible, but evidence is required before paying permanent complexity |

For each finding, cite the owning code and explain the behavior change, not just the style preference. Do not use a clarity score as a substitute for evidence.

## Final standard

The result should make these questions easy to answer:

- Where is the one authoritative state?
- What are the few supported lifecycle states?
- Which failures actually change recovery?
- Which module owns each transition and resource?
- What can be deleted without changing observable behavior?
- Do the tests protect the contract rather than the implementation shape?

Clarity is achieved when the normal path is obvious, exceptional paths are few and justified, and every remaining concept earns its place.
