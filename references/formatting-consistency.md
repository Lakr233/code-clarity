# Formatting Consistency: The Prettier Pattern

Mechanical style — indentation, quotes, semicolons, trailing commas, line width, import wrapping — is a clarity concern, but it is the one concern that should be decided **once, by a repository-owned tool and reviewed configuration**, and then rarely revisited. This reference is about adopting an opinionated formatter (Prettier for the JS/TS world; `gofmt`, `swift-format`, `clang-format`, `black` for others) and letting it own mechanical style so human attention goes to names and structure.

---

## The core idea

Inconsistent mechanical style is noise. The eye trips on a stray single-quote-where-everything-else-is-double exactly the way it trips on a misleading name. Worse, hand-formatting pollutes diffs: a one-line logic change arrives wrapped in forty lines of reflow, and the reviewer can no longer see what actually changed.

A formatter removes the entire category. Every file looks the same, so a diff shows only genuine changes, and code review never spends a sentence on whitespace.

**The Prettier philosophy, stated plainly:** stop debating style, encode it in a config, run it on save and in CI. Style stops being a per-file decision and becomes invisible infrastructure.

---

## What the formatter owns vs. what the linter owns

These are different jobs. Do not blur them.

| Tool | Owns | Examples |
|------|------|----------|
| **Formatter** (Prettier) | Mechanical style — anything with no semantic meaning | Quotes, semicolons, indent width, trailing commas, line width, argument wrapping, JSX layout |
| **Linter** (ESLint) | Correctness and intent — things a human could get *wrong* | Unused variables, `no-floating-promises`, exhaustive `switch`, banned APIs, hook rules |

Run Prettier for style and ESLint for bugs. Disable ESLint's stylistic rules so the two don't fight. Once the repository has validated the formatter configuration for its language, the formatter owns those mechanical decisions.

---

## Prettier's defaults are a fine baseline

Prettier is opinionated on purpose; its defaults are a reasonable house style for most TypeScript projects:

| Option | Default | Effect |
|--------|---------|--------|
| `printWidth` | `80` | Wrap target; not a hard limit |
| `tabWidth` | `2` | 2-space indent |
| `semi` | `true` | Statement-terminating semicolons |
| `singleQuote` | `false` | Double quotes |
| `trailingComma` | `'all'` (current) | Trailing commas everywhere legal — cleaner diffs |
| `arrowParens` | `'always'` | `(x) => x`, not `x => x` |
| `bracketSpacing` | `true` | `{ foo }` not `{foo}` |

You may override any of these — but only in a **committed `.prettierrc`**, applied by the tool, never by hand. The exact values matter far less than that there is exactly one set of them.

```json
// .prettierrc — example override set, then never formatted by hand again
{
  "printWidth": 100,
  "singleQuote": true,
  "semi": false,
  "trailingComma": "all"
}
```

---

## Make it non-negotiable: save + CI

Consistency only holds if it is enforced in two places:

1. **Editor: format on save.** Each contributor's editor runs the project's Prettier config on every save (e.g. VS Code `editor.formatOnSave` + the Prettier extension picking up `.prettierrc`). This keeps the working tree clean continuously.
2. **CI: a check gate.** A pipeline step runs `prettier --check .` (and `eslint .`) and fails the build on any unformatted file, so nothing unformatted can merge. A common pattern is a `build`/`verify` script whose first step is `lint`:

```jsonc
// package.json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint src/",
    "build": "bun run lint && bun run format:check && bun run build:app"
  }
}
```

Format-on-save keeps it pleasant; the CI check keeps it true.

---

## Match the config, not your preference

The single most common formatting clarity failure in review is hand-introducing the "default" style into a repo whose committed config says otherwise.

- If the repo's `.prettierrc` says `semi: false, singleQuote: true`, then semicolon-less single-quoted code **is** the correct style there. Do not "fix" it toward Prettier's stock defaults.
- The config file is the source of truth for the dialects it has been validated against. Accept isolated formatter output; when the same output repeatedly harms readability or macros, fix the configuration instead of hand-correcting each file.
- Never reformat unrelated lines in a logic PR. If a file needs reformatting, do it in a separate, formatting-only commit so logic diffs stay legible.

## Validate the configuration for the language

A formatter is authoritative only after its configuration has been shown to produce maintainable code for the repository's language and dialect. Parser-backed does not mean universally readable or semantically risk-free.

Objective-C is especially sensitive because long selectors, nested message sends, blocks, adjacent string literals, C declarations, and macros interact with generic C/C++ wrapping rules. Reject a configuration that routinely:

- leaves `=`, `return`, a receiver such as `[NSError`, or a selector colon stranded on its own line;
- turns each selector piece into a separate line when the complete statement is still readable;
- rewrites or pads macro bodies and escaped newlines;
- creates whole-file churn to satisfy a line-count or nominal width target.

Fix the Objective-C-specific formatter configuration before changing source. Preserve intentionally vertical invariant lists, mapping tables, command templates, and low-level calls whose argument positions need auditing. For detailed Objective-C guidance, read [objective-c.md](objective-c.md).

---

## When to override the formatter (rarely)

Reach for `// prettier-ignore` only when alignment carries genuine meaning the formatter would destroy:

```ts
// prettier-ignore
const KERNEL = [
  0, 1, 0,
  1, 4, 1,
  0, 1, 0,
]
```

Tabular data, ASCII diagrams, and matrices qualify. "I like my arguments aligned this way" does not — hand alignment breaks on the next edit and re-introduces the churn you adopted a formatter to avoid.

---

## Other languages, same pattern

The principle is not Prettier-specific; it is *defer mechanical style to a canonical tool*:

| Language | Tool | Note |
|----------|------|------|
| TypeScript / JS / JSON / CSS / Markdown | **Prettier** | The reference case here |
| Go | **gofmt** / **goimports** | Non-negotiable by community norm — there is no style debate in Go |
| Swift | **swift-format** / SwiftFormat | Adopt one, commit its config, run in CI |
| Objective-C / Objective-C++ | **clang-format** | Validate an Objective-C-specific style on messages, blocks, literals, and macros before repository-wide use |
| Python | **Black** (+ isort/ruff) | "Any color you like, as long as it's black" — the same opinionated stance |
| Rust | **rustfmt** | Same |

---

## Diagnostic

| Symptom | Fix |
|---------|-----|
| Files disagree on quotes/semicolons/indent | Add a committed formatter config; run it across the repo once |
| Logic PRs carry large reformatting diffs | Format on save; never reformat by hand; separate formatting commits |
| Reviewers comment on whitespace/quotes | Move the decision into the config; it is not a review topic |
| Contributors' editors format differently | Commit the config; enable format-on-save; add a `--check` CI gate |
| Someone "fixed" a file toward stock defaults | Revert to the repo's `.prettierrc` — the config is the source of truth |
| Hand-aligned code keeps breaking on edits | Let the formatter wrap; reserve `// prettier-ignore` for true tabular data |
