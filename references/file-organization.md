# File Organization and Named Constants

Two decisions shape a codebase's readability before a reader looks at a single function: **how code is split into files**, and **how fixed values are named and scoped**. Both are about making the file tree and the top of each file act as documentation. This reference covers cohesive modules and the module-level constant pattern.

The examples below use a neutral, illustrative desktop-app domain (documents, uploads, settings); apply the same shapes to your own domain.

---

## Part 1 — One Cohesive Subject Per File

### The rule

**A file should have one discoverable subject, and its filename should name that subject.** A file may contain several exports when they form one API, change together, and are normally understood together. The file tree should remain a table of contents rather than a collection of arbitrary grab-bags.

Common reasons to co-locate declarations:

- a public type and the small values/functions that constitute its API;
- a schema and the type inferred from it;
- a discriminated union and its exhaustive helpers;
- private implementation used only by the subject;
- declarations that always change in the same commits for the same reason.

Split when a declaration has independent consumers, independent reasons to change, or makes the filename misleading. Do not split merely to satisfy one-type-per-file style; excessive splitting makes readers traverse shallow files to understand one concept.

### Why it works

| Benefit | Mechanism |
|---------|-----------|
| The tree is navigable | Filename = symbol name, so search-by-file works |
| Diffs are scoped | A change to one concept touches one file |
| Merge conflicts shrink | Two people editing two classes edit two files |
| Files resist growth | A file named `slug-sanitizer.ts` rejects unrelated additions; `utils.ts` invites them |
| Imports read as a dependency list | `import { WindowManager } from './window-manager'` says where it lives |

The "pull force" of a specific filename is the same effect as a specific type name: `date-formatting.ts` will never feel like the right home for retry logic.

### File naming conventions

Follow the repository's established convention and make new filenames predictable. Two common, internally consistent schemes are:

| File kind | Convention | Examples |
|-----------|-----------|----------|
| Logic / utility / service module | **kebab-case** | `document-store.ts`, `slug-sanitizer.ts`, `upload-queue.ts`, `endpoint-validation.ts` |
| React component file | **PascalCase**, matching the component | `DocumentCard.tsx`, `App.tsx`, `SettingsPanel.tsx` |
| Abstract contract / interface | kebab-case with a clear suffix | `payment-gateway-interface.ts`, `download-manager-interface.ts` |
| Closely-related satellite of a component | component name + dotted concern | `DocumentCard.toolbar.ts` next to `DocumentCard.tsx` |
| Barrel / re-export | `index.ts` | `packages/shared/src/documents/index.ts` |

Swift analog: name the file after the owning type or feature. Keep tightly coupled support types nearby; use `Type+Feature.swift` when an extension is independently navigable and the repository already follows that pattern.

Avoid: camelCase filenames, plural grab-bag names (`managers.ts`, `helpers.ts`, `utils.ts`, `types.ts` as a dumping ground), and files whose name doesn't appear as an export.

### Folder organization: domain folders, flat files

Group by **domain**, then keep files flat inside the folder. Depth comes from domains, not from nesting files five levels deep.

```
src/
  main/
    window-manager.ts
    download-manager.ts
    handlers/                 ← domain folder
      index.ts                ← barrel: registerAllHandlers()
      system.ts
      documents.ts
      service-context.ts      ← the dependency-bag type, one export
  shared/
    documents/
      index.ts                ← barrel
      storage.ts
      upload-queue.ts
      types.ts
  renderer/
    components/
      app-shell/
        DocumentCard.tsx
        DocumentCard.toolbar.ts
```

### Barrels are tables of contents, not code

An `index.ts` barrel should make the public surface easy to inspect. Prefer re-exports only; keep a tiny registration/composition function there only when the repository consistently treats the index as that folder's entry point.

```ts
// Good — explicit, visible surface
export { loadDocument, saveDocument } from './storage'
export { UploadQueue } from './upload-queue'
export type { Document, PersistedField } from './types'

// Avoid — blanket re-export hides what is public and what is internal
export * from './storage'
export * from './upload-queue'
export * from './types'
```

Explicit re-exports make the package's API reviewable in one glance and stop internal helpers from leaking out by accident. A single `export *` at the root of a deliberately-shallow package (e.g. a `core` package) is an acceptable exception when the whole package *is* the public surface.

### When one file legitimately holds several exports

- A **discriminated union** plus its small type-guard helpers (`isLoaded`, `isFailed`) — they are one concept.
- A **zod schema** plus its inferred type (`export const FooSchema` + `export type Foo = z.infer<typeof FooSchema>`).
- A set of **free functions on one subject** (`date-formatting.ts` with `formatRelative`, `formatRange`) — cohesive, not a grab-bag.
- A component plus its **props interface** and private subcomponents used nowhere else.

The test is cohesion: would a reader expect to find all of these by opening this filename? If yes, keep them together. If one is a surprise, split it.

### Diagnostic

| Symptom | Fix |
|---------|-----|
| File needs a banner comment to separate two exports | Split into two files |
| Filename is plural and generic (`managers.ts`) | Rename around the cohesive subject or split unrelated declarations |
| You can't predict the filename from the symbol you want | Rename file to its primary export |
| Barrel uses `export *` and you can't tell what's public | Convert to explicit named re-exports |
| A "helper" file keeps absorbing unrelated functions | Name files by subject so each resists drift |

---

## Part 2 — Named Constants (the `fileprivate let` pattern)

A literal with meaning — a timeout, a retry ceiling, a path, a wire string, a debounce — should be a **named constant, hoisted to the top of its file, at the narrowest scope that serves its readers.** This is the same instinct as Swift's `private`/`fileprivate let`: give the value a name and keep it from leaking.

### Why hoist and name

- `0.3` is a mystery; `animationSettleDuration` is documentation.
- A hoisted constant centralizes the value — a change happens in one place, not at three scattered call sites.
- Scoping it (file-private) keeps it out of the package's public API where it would become an accidental contract.

### TypeScript

Use the repository's constant naming convention. Keep module constants above their readers and unexported unless genuinely shared; many TypeScript repositories use `camelCase`, while others reserve `SCREAMING_SNAKE_CASE` for process-wide or protocol constants.

```ts
// module top — file-private by default (no `export`)
const MAX_LOG_AGE_MS = 24 * 60 * 60 * 1000
const CONFIG_CACHE_TTL_MS = 100
const DEBOUNCE_MS = 100

// Compose from constants so the relationship stays visible
const META_DEBOUNCE_MS = process.platform === 'win32' ? 300 : DEBOUNCE_MS

function writeLog(entry: LogEntry) {
  if (Date.now() - entry.createdAt > MAX_LOG_AGE_MS) return
  // ...
}
```

Rules of thumb:

- **Don't `export` a constant only this file uses.** An un-exported `const` is module-private — that is the `fileprivate` you want. Export only when the value is a shared contract.
- **Name a literal when the name carries domain meaning, centralizes a shared contract, or prevents inconsistent edits.** Repetition alone is not enough; obvious local values can remain inline.
- Keep the constant at the lowest scope covering its readers: function-local if one function uses it, module-private if the file uses it, exported only if shared.

### `as const` for fixed maps and tuples

Lock registries and option sets with `as const` so TypeScript narrows to literal types — typos become compile errors and types can be derived from the data:

```ts
// Wire-protocol registry: one source of truth, literal-typed
export const IPC_CHANNELS = {
  documents: { GET: 'documents:get', CREATE: 'documents:create' },
  window: { CLOSE: 'window:close', FOCUS_STATE: 'window:focusState' },
} as const
// IPC_CHANNELS.documents.GET has type 'documents:get', not string

// Derive a type from the data instead of restating it
const PERSISTED_FIELDS = ['id', 'title', 'updatedAt'] as const
type PersistedField = (typeof PERSISTED_FIELDS)[number]
// 'id' | 'title' | 'updatedAt'
```

This is the TypeScript form of a frozen enum: one declaration drives both the runtime values and the compile-time type, so they cannot drift.

### Swift

```swift
// File-private constants, scoped deliberately
private let maxRetryCount = 3
private let settleDelay: TimeInterval = 0.3

// Group related design constants in a namespacing enum
private enum Layout {
    static let cardSpacing: CGFloat = 12
    static let cornerRadius: CGFloat = 8
}
```

Use `private` / `fileprivate` on purpose. A constant only one type needs should not be `internal`, where it becomes part of the module's surface.

### Diagnostic

| Symptom | Fix |
|---------|-----|
| Bare number/string at a call site with non-obvious meaning | Hoist to a named top-of-file constant |
| Same literal in multiple places | Single named constant; one source of truth |
| `export const` used by only this file | Drop `export` — make it file-private |
| A `string[]` of names also restated as a union type | `const NAMES = [...] as const; type Name = typeof NAMES[number]` |
| Magic wire strings sprinkled through IPC/HTTP code | One `as const` channel/route registry, referenced everywhere |
