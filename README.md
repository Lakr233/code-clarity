# Code Clarity

A Claude Code skill for writing readable, intention-revealing code — precise naming, consistent abstraction levels, repository-aware conventions, and the early-return pattern. Swift-primary with Go, TypeScript, and Python equivalents throughout.

## What This Covers

| Topic | Description |
|-------|-------------|
| **Naming Conventions** | Function, boolean, class, and parameter naming rules with cross-language examples |
| **Early Return** | Guard clause pattern — Swift `guard` and Go-style early exit, when to use and when not to |
| **Function Design** | Single responsibility, one abstraction level per function, parameter design |
| **Abstraction Levels** | Keeping orchestration and implementation separate, the step-down rule, layered architecture |
| **Repository Conventions** | How to infer and preserve a codebase's existing naming, layout, and file-organization style |
| **Class & Struct Design** | Single responsibility, Swift struct vs class decisions, coupling and cohesion |

## Install

Add to your Claude Code project by placing `SKILL.md` and the `references/` folder where Claude Code can find them, or reference them directly in your project's skill configuration.

With [Claude Code](https://claude.ai/code), skills in `~/.claude/skills/` or your project's configured skills directory are loaded automatically.

## Usage

Once installed, the skill activates when you ask Claude to:

- Review or rename functions, variables, or types for clarity
- Refactor nested conditionals into early-return style
- Audit a class for single-responsibility violations
- Check abstraction level consistency in a module
- Review a change for clarity while preserving the repository's existing style
- Design a new type with appropriate struct/class choice

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
├── SKILL.md                        # Main skill — framework, scoring, quick diagnostics
├── agents/
│   └── openai.yaml                 # UI metadata for skill lists and chips
└── references/
    ├── naming-conventions.md       # Full naming rules by category
    ├── early-return.md             # Guard clause pattern with Swift and Go examples
    ├── function-design.md          # Single responsibility, parameters, side effects
    ├── repository-conventions.md   # How to match and extend an existing house style
    ├── abstraction-levels.md       # Level consistency, step-down rule, layering
    └── class-struct-design.md      # SRP, struct vs class, coupling and cohesion
```

## Design Philosophy

This skill is shaped by two complementary perspectives:

- **Go's pragmatism** — handle errors early, keep the happy path flat, avoid unnecessary nesting
- **Swift's expressiveness** — `guard let`, protocol-oriented design, value semantics for data
- **Repository-local clarity** — clarity is partly local, so preserve the codebase's existing vocabulary and file patterns before introducing a new one

The underlying principle is language-agnostic: **code is written once but read many times**. Every name and structural decision is a communication decision.

## Related Skills

- [`software-design-philosophy`](https://github.com/Lakr233/software-design-philosophy) — module depth, information hiding (Ousterhout)
- [`clean-architecture`](https://github.com/Lakr233/clean-architecture) — dependency rule, layer boundaries
- [`swiftui-expert-skill`](https://github.com/Lakr233/SwiftUI-Agent-Skill) — SwiftUI-specific patterns

## License

MIT — see [LICENSE](LICENSE).
