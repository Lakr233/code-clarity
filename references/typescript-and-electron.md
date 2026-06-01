# TypeScript and Electron Clarity

This reference carries the TypeScript-first and Electron-first material: naming, type modeling with discriminated unions and schemas, async/error idioms, import/export hygiene, and the main/preload/renderer process boundary with a typed IPC bridge. For the universal principles (single responsibility, abstraction levels, early return) see the other references; this file is about applying them with TypeScript and Electron grain.

The examples use a neutral, illustrative desktop-app domain (documents, uploads, settings); apply the same shapes to your own domain.

---

## 1. Naming in TypeScript

The universal rules from [naming-conventions.md](naming-conventions.md) apply; TypeScript adds a few of its own.

### Functions: verbs for actions, nouns for queries

```ts
// Actions — strong verb
function createDocument(...) { }
function saveDocument(...) { }
function registerSystemHandlers(...) { }
function sanitizeSlug(raw: string): string { }

// Queries / finders — describe the result
function loadDocument(id: string): Document | undefined { }
function listDocuments(): Document[] { }
function getWindowState(id: number): WindowState | undefined { }

// Predicates — is / has / can / should, return boolean
function isLoopbackUrl(url?: string): boolean { }
function isRendererUrl(url: string): boolean { }
function canRetry(state: ConnectionState): boolean { }
```

Do not suffix async functions with `Async`. `async function createDocument()` already reads as an action; the `async` keyword and `Promise<T>` return type carry the asynchrony.

### Booleans: `is` / `has` / `can` / `should`

```ts
isArchived, isFlagged, isProcessing, isStreaming, isPackaged
hasUnread, hasWindows()
canRetry, canUpdatePath
shouldReconnect
```

Avoid bare adjectives and negatives: prefer `isEnabled` over `disabled`, `isConnectionLocked` over `connectionLocked`.

### Types and interfaces

- Name the role, not the shape: `Document`, `ServiceContext`, `Workspace` — not `DocumentData`, `Info`, `Thing`.
- **Do not reflexively `I`-prefix interfaces.** Reserve an `I`-prefix only for an *abstract contract* that has a concrete same-named implementation, and only if the repo already does this (`IDocumentStore` interface / `DocumentStore` class). Otherwise plain role names read better.
- Consistent role suffixes communicate a type's job at a glance:

| Suffix | Role | Example |
|--------|------|---------|
| `*Service` | A stateless-ish operation provider | `BillingService` |
| `*Store` | Owns a slice of state | `PreferencesStore`, `settingsStore` |
| `*Manager` | Owns a lifecycle / coordinates | `WindowManager`, `DownloadManager` |
| `*Registry` | A lookup table of things | `PluginRegistry` |
| `*Queue` | Ordered async work | `UploadQueue` |
| `*Schema` | A zod (or similar) validator | `DocumentSchema`, `AppConfigSchema` |
| `*Gate` | A one-time barrier | `StartupGate` |

Use a suffix when the repo uses it with a stable meaning; don't invent a parallel vocabulary.

---

## 2. Type Modeling: Discriminated Unions Over Boolean Piles

This is principle 4 (State Modeling) in TypeScript form. Where Swift uses an `enum` with associated values, TypeScript uses a **discriminated (tagged) union** with a literal discriminant (`type` / `kind` / `status`). The payload that exists only in one state lives only in that variant, so impossible combinations are unrepresentable.

```ts
// Avoid — booleans + optionals that must be kept in sync
interface LoadState {
  isLoading: boolean
  isError: boolean
  data?: Result
  error?: Error
}
// Is { isLoading: true, isError: true, data: X } valid? Nobody knows.

// Prefer — one value; each variant carries exactly its own payload
type LoadState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'loaded'; data: Result }
  | { status: 'failed'; error: Error }

function render(state: LoadState) {
  switch (state.status) {
    case 'idle': return renderIdle()
    case 'loading': return renderSpinner()
    case 'loaded': return renderData(state.data) // data is in scope here, typed
    case 'failed': return renderError(state.error)
  }
}
```

Real apps scale this pattern to large event/state unions — a domain event type with dozens of tagged variants, or a route state discriminated on a `route` field — each with tiny type-guard helpers instead of scattered booleans:

```ts
type RouteState =
  | { route: 'list'; query?: string }
  | { route: 'detail'; documentId: string }
  | { route: 'settings'; panel: SettingsPanel }

const isDetailRoute = (s: RouteState): s is Extract<RouteState, { route: 'detail' }> =>
  s.route === 'detail'
```

A `default: assertNever(state)` arm makes the compiler enforce exhaustiveness — add a variant, every switch that forgot it fails to compile.

### Schemas at the boundary

Validate external/persisted data with a schema (zod or similar) so the runtime shape and the compile-time type can't drift. Derive the type from the schema — never restate it.

```ts
const DocumentSchema = z.object({
  id: z.string().min(1),
  title: z.string().min(1),
})
type Document = z.infer<typeof DocumentSchema> // derived, single source of truth

// Discriminated union, validated:
const ActionSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('rename'), title: z.string() }),
  z.object({ type: z.literal('webhook'), url: z.string().url() }),
])
```

Use schemas where data enters the program — config files, IPC payloads, network responses — not for internal values the type system already guarantees.

---

## 3. Async and Error Handling

### `unknown` in catch — narrow before use

In modern TypeScript a caught error is `unknown`. Narrow it; never assume `.message` exists.

```ts
// Avoid — assumes e is an Error
catch (e) { logger.error(e.message) }

// Prefer — the consistent idiom
catch (e) {
  const message = e instanceof Error ? e.message : 'Unknown error'
  logger.error('openUrl failed:', message)
  throw new Error(`Failed to open URL: ${message}`)
}
```

### Scope try/catch to the operation that can throw

Keep validation and precondition guards *outside* the `try`. The `try` wraps only the throwing call, so the catch can't accidentally swallow a programming error from the surrounding code.

```ts
// Avoid — guards and dispatch buried inside one broad catch
async function handle(input: Input) {
  try {
    if (!input.id) throw new Error('missing id')   // validation
    const row = await db.fetch(input.id)            // the real throwing op
    dispatch(row)                                   // side effect
  } catch (e) { /* which step failed? */ }
}

// Prefer — guard first, catch only the I/O
async function handle(input: Input) {
  if (!input.id) return // early return; not exceptional
  let row: Row
  try {
    row = await db.fetch(input.id)
  } catch (e) {
    const message = e instanceof Error ? e.message : 'Unknown error'
    logger.error('fetch failed:', message)
    return
  }
  dispatch(row)
}
```

### Result types for expected failure

When failure is a routine, expected outcome (validation, parsing user input), don't `throw` — return a discriminated Result. The caller handles it with an early return, and the success payload is typed.

```ts
type Validated =
  | { valid: true; path: string }
  | { valid: false; error: string }

async function validateExecutablePath(filePath: string): Promise<Validated> {
  const trimmed = filePath.trim()
  if (!isExecutable(trimmed)) return { valid: false, error: 'Must point to an executable' }
  try {
    const info = await stat(trimmed)
    if (!info.isFile()) return { valid: false, error: 'Must point to a file' }
    return { valid: true, path: trimmed }
  } catch {
    return { valid: false, error: 'File does not exist' }
  }
}

// caller
const result = await validateExecutablePath(input)
if (!result.valid) return showError(result.error)
use(result.path) // typed, present
```

Reserve `throw` for genuinely exceptional/unexpected failures (network down, invariant broken). A dedicated `Result` library (e.g. neverthrow) is fine; a hand-rolled `{ ok } | { ok: false }` union is also fine — be consistent.

### Don't float promises

A bare `doThing()` that returns a promise discards its rejection silently. Either `await` it inside a scope that handles failure, or mark the fire-and-forget deliberately:

```ts
void refreshInBackground().catch((e) => logger.debug('background refresh failed', e))
```

### Clean up in `finally`

`finally` is the TypeScript `defer` — it runs on both the success and throw paths, so resources release even when the body throws.

```ts
let server: CallbackServer | undefined
try {
  server = await createCallbackServer(...)
  return await runAuthFlow(server)
} catch (e) {
  return { success: false, error: e instanceof Error ? e.message : 'Auth failed' }
} finally {
  server?.close()
}
```

---

## 4. Imports and Exports

- **Named exports for logic; default export only for a page/route component.** Packages, main-process, and preload code use named exports exclusively. A `renderer/pages/SettingsPage.tsx` may `export default`. This keeps refactors and find-references reliable.
- **`import type` for type-only imports.** It documents intent and lets the bundler erase the import entirely.
  ```ts
  import type { IpcServer } from '@app/core/ipc'
  import type { Document, Message } from '../../shared/types'
  ```
- **Import ordering** (top to bottom), with a blank line between groups:
  1. Node built-ins (`path`, `os`, `fs`, `child_process`)
  2. Third-party packages (`electron`, `react`, `zod`)
  3. Internal monorepo packages (`@app/shared/...`)
  4. Local relative imports (`./service-context`)
- **Explicit barrels**, not `export *` (see [file-organization.md](file-organization.md)).

---

## 5. The Electron Process Boundary

An Electron app is **three programs with three trust levels**. Clarity means each world contains only what belongs to it, and they communicate through one typed, allowlisted surface.

```
src/
  main/        ← Node, full OS access: lifecycle, BrowserWindow, native dialogs,
               ←   secure storage, auto-update, IPC handler registration
  preload/     ← the ONLY bridge: contextBridge.exposeInMainWorld(...)
  renderer/    ← sandboxed browser: React UI only, no Node, no secrets
  shared/      ← DTOs and types imported by all three (no runtime Node/DOM deps)
```

A file's directory announces its trust level. A reader who sees `fs` imported in `renderer/` knows immediately something is wrong.

### Rule 1 — the renderer never imports Node

No `fs`, `child_process`, `electron` main APIs, native modules, or secret material in renderer code. If the UI needs OS work done, it *asks the main process*:

```ts
// renderer — WRONG: reaches across the boundary
import { readFile } from 'fs'
const doc = await readFile(path)

// renderer — RIGHT: ask main over the typed bridge
const doc = await window.appAPI.readDocument(id)
```

### Rule 2 — expose one typed API, not raw stringly-typed IPC

Sprinkling `ipcRenderer.invoke('some-string', ...)` through the UI is untyped and typo-prone. Instead, define the channel registry once with `as const`, share it across processes, and expose a single typed proxy through `contextBridge`.

```ts
// shared/protocol.ts — one source of truth, literal-typed
export const IPC_CHANNELS = {
  documents: { GET: 'documents:get', CREATE: 'documents:create' },
  window: { CLOSE: 'window:close' },
} as const

// shared/types.ts — the contract the renderer sees
export interface AppAPI {
  getDocument(id: string): Promise<Document | undefined>
  createDocument(input: CreateDocumentInput): Promise<Document>
  closeWindow(): Promise<void>
}
declare global {
  interface Window { appAPI: AppAPI }
}

// preload/bootstrap.ts — the ONLY bridge; one exposed key
const api: AppAPI = buildTypedClient(IPC_CHANNELS /* ...transport... */)
contextBridge.exposeInMainWorld('appAPI', api)

// main/handlers/documents.ts — handlers keyed by the shared channel map
server.handle(IPC_CHANNELS.documents.GET, async (_ctx, id: string) => {
  return loadDocument(id) // typed both ends; 'documents:get' typo = compile error
})
```

The renderer then calls `window.appAPI.getDocument(id)` with full inference and no magic strings. A `namespace:action` channel name (`documents:get`, `window:close`) read from the shared `as const` map keeps the wire protocol self-documenting.

### Rule 3 — keep the security switches set

`contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` where feasible. The preload allowlist is the security contract: widen it one deliberate method at a time, never with a blanket pass-through that forwards arbitrary channels.

```ts
new BrowserWindow({
  webPreferences: {
    preload: bootstrapPreloadPath,
    contextIsolation: true,
    nodeIntegration: false,
  },
})
```

### Rule 4 — portable logic lives in process-neutral packages

Business rules, domain types, and pure functions belong in process-neutral packages (e.g. `packages/core`, `packages/shared`) with **no** Electron, DOM, or OS imports — so they are unit-testable and reusable across a CLI, server, and renderer. The Electron-specific handler is a thin adapter that calls the portable rule:

```ts
// packages/core — no Electron import, pure, testable
export function sanitizeSlug(raw: string): string { /* ... */ }

// main/handlers — thin Electron adapter
server.handle(IPC_CHANNELS.documents.RENAME, async (_ctx, id, raw) => {
  const slug = sanitizeSlug(raw) // logic lives below the process boundary
  return renameDocument(id, slug)
})
```

### Rule 5 — secrets live in main, behind secure storage

Tokens and credentials use the OS/Electron secure storage in the main process. Never `localStorage.setItem('token', ...)` in the renderer, and never expose a token through a renderer-visible global. The renderer asks main to *perform* the authenticated action; it never holds the credential.

### Electron diagnostic

| Symptom | Fix |
|---------|-----|
| `import 'fs'` / `child_process` in a `.tsx` renderer file | Move the work to main; call it over the typed bridge |
| `ipcRenderer.invoke('typo:channel', ...)` in the UI | Shared `as const` channel map + typed client proxy |
| `contextBridge.exposeInMainWorld('api', { invoke })` returning `any` | Expose a typed `interface AppAPI` with concrete `Promise<T>` methods |
| Business rule written inline in an IPC handler | Extract to a process-neutral package; handler just calls it |
| Token in `localStorage` or a renderer global | Secure storage in main; renderer never sees the secret |
| `nodeIntegration: true` "for convenience" | `contextIsolation: true`, `nodeIntegration: false`, preload allowlist |
| Multiple `exposeInMainWorld` keys forwarding raw channels | One key, one typed, allowlisted surface |
