# Code Clarity

A Claude Code skill for writing readable, intention-revealing code with precise names, flat control flow, consistent abstraction levels, focused files, named constants, formatter-aligned style, repository conventions, and testable seams. **Swift, TypeScript, Electron, Go, and Python examples** are included throughout.

## What This Covers

| Topic | Description |
|-------|-------------|
| **Naming Conventions** | Function, boolean, class, parameter, and TS interface/suffix naming with cross-language examples |
| **Early Return** | Guard clause pattern — Swift `guard`, Go-style early exit, TS guards, when to use and when not to |
| **Function Design** | Single responsibility, one abstraction level per function, parameter design |
| **State Modeling** | Replacing boolean piles with Swift enums and TypeScript discriminated unions / schemas |
| **Abstraction Levels** | Keeping orchestration and implementation separate, the step-down rule, layered architecture |
| **Repository Conventions** | How to infer and preserve a codebase's existing naming, layout, and file-organization style |
| **Error Boundaries** | Handling failures at the right level; Swift `do/catch` and TS `unknown`/Result idioms |
| **Class & Struct Design** | Single responsibility, Swift struct vs class decisions, coupling and cohesion |
| **File Organization** | One object per file, file naming, domain folders, explicit barrels |
| **Named Constants** | Hoisting and scoping fixed values — Swift `fileprivate let`, TS module-top `const` and `as const` |
| **Formatting Consistency** | The Prettier pattern — deferring mechanical style to a formatter, config, and CI gate |
| **Electron Boundaries** | Main / preload / renderer separation and one typed, allowlisted IPC bridge |
| **Dependencies & Test Seams** | One canonical implementation, protocol/subclass seams over closures, minimal DI, fakes over mocks |

## Install

Add to your Claude Code project by placing `SKILL.md` and the `references/` folder where Claude Code can find them, or reference them directly in your project's skill configuration.

With [Claude Code](https://claude.ai/code), skills in `~/.claude/skills/` or your project's configured skills directory are loaded automatically.

## Usage

Once installed, the skill activates when you ask Claude to:

- Review or rename functions, variables, or types for clarity
- Refactor nested conditionals into early-return style
- Audit a class for single-responsibility violations
- Check abstraction level consistency in a module
- Split a grab-bag file into one-object-per-file modules
- Hoist magic numbers and strings into named, correctly-scoped constants
- Set up formatter-based mechanical consistency (Prettier + CI gate)
- Review an Electron app's main/preload/renderer boundary and typed IPC surface
- Replace closure-injected dependencies with protocol/subclass seams in Swift
- Reduce over-injection and mock-heavy tests; design real fakes and one canonical implementation
- Review a change for clarity while preserving the repository's existing style
- Design a new type with appropriate struct/class choice, or a TS discriminated union

**Example prompts:**

```
Review this function for naming and abstraction level consistency.
```

```
This code has deeply nested conditionals — refactor using early return.
```

```
This class feels too big. Help me identify where to split it.
```

```
Review this file for clarity, but preserve the repo's existing naming and file-organization style.
```

## Review Workflow

When the skill is working well, it should:

1. Sample representative files from the same module or layer.
2. Infer the repository's local conventions.
3. Score current clarity.
4. Recommend the smallest changes that improve clarity without fighting the house style.
5. Call out explicitly when the local style itself is the source of confusion.

## Skill Structure

```
code-clarity/
├── SKILL.md                        # Main skill — 13 principles, scoring, quick diagnostics
├── agents/
│   └── openai.yaml                 # UI metadata for skill lists and chips
└── references/
    ├── naming-conventions.md       # Full naming rules by category
    ├── early-return.md             # Guard clause pattern with Swift, Go, and TS examples
    ├── function-design.md          # Single responsibility, parameters, side effects
    ├── repository-conventions.md   # How to match and extend an existing house style
    ├── abstraction-levels.md       # Level consistency, step-down rule, layering
    ├── class-struct-design.md      # SRP, struct vs class, coupling and cohesion
    ├── file-organization.md        # One object per file, file naming, barrels, named constants
    ├── formatting-consistency.md   # The Prettier pattern — formatter-owned mechanical style
    ├── typescript-and-electron.md  # TS naming/types/errors + Electron main/preload/renderer + typed IPC
    └── testing-and-seams.md        # One implementation, protocol seams over closures, minimal DI, fakes over mocks
```

## Design Philosophy

This skill is shaped by complementary perspectives:

- **Go's pragmatism** — handle errors early, keep the happy path flat, avoid unnecessary nesting
- **Swift's expressiveness** — `guard let`, protocol-oriented design, value semantics for data
- **TypeScript's structural typing** — discriminated unions over boolean piles, `as const` registries, schemas at the boundary
- **Electron's trust model** — three processes, one typed allowlisted bridge, no Node in the renderer
- **Prettier's stance** — encode mechanical style once, enforce it in CI, stop debating it
- **Test-driven design pressure** — one canonical implementation, protocol/subclass seams over injected closures, minimal DI, and fakes over mocks
- **Repository-local clarity** — clarity is partly local, so preserve the codebase's existing vocabulary and file patterns before introducing a new one

The underlying principle is language-agnostic: **code is written once but read many times**. Every name, file boundary, constant, and structural decision is a communication decision.

## Related Skills

- [`software-design-philosophy`](https://github.com/Lakr233/software-design-philosophy) — module depth, information hiding (Ousterhout)
- [`clean-architecture`](https://github.com/Lakr233/clean-architecture) — dependency rule, layer boundaries
- [`swiftui-expert-skill`](https://github.com/Lakr233/SwiftUI-Agent-Skill) — SwiftUI-specific patterns

## License

MIT — see [LICENSE](LICENSE).
