---
name: code-clarity
description: 'Write readable, intention-revealing code through precise naming, consistent abstraction levels, one-object-per-file organization, hoisted named constants, formatter-deferred mechanical consistency, repository-aware conventions, and the early-return pattern. Use when naming feels off, logic is hard to follow, functions are doing too much, nested conditionals obscure flow, files mix many exports, magic numbers are inline, formatting is inconsistent, or a refactor should preserve the codebase''s local style. Covers function/method naming, boolean naming, guard clauses, abstraction-level consistency, file organization, module-level constants, Prettier-style formatting, repository conventions, class/struct responsibilities, and Electron main/preload/renderer boundaries. Swift, TypeScript, and Electron first-class, with Go and Python equivalents throughout.'
license: MIT
metadata:
  author: qaq
  version: "1.2.0"
---

# Code Clarity Framework

A practical framework for writing code that communicates intent clearly. The central thesis: **a developer reads code far more than they write it**, so every naming and structural decision is a communication decision. Code that requires a reader to reconstruct the author's mental model has failed at its primary job.

This framework focuses on the micro-level of software design — the decisions made at the function, method, class, and file level — and complements macro-level architecture thinking.

**Languages covered.** The framework is language-agnostic in principle, with first-class guidance for **Swift**, **TypeScript**, and **Electron** (the main/preload/renderer split), plus Go and Python equivalents where they sharpen a point. Swift examples carry the value-vs-identity and `guard` material; TypeScript and Electron examples carry the one-object-per-file, module-constant, discriminated-union, typed-IPC, and formatter-consistency material drawn from real production codebases.

## Core Principles

**Code is written once but read hundreds of times.** Every name, every function boundary, every conditional structure is a message to the next reader (usually yourself, six months later). Clarity is not a style preference — it is a correctness property: unclear code is one misunderstanding away from a bug.

**Clarity is partly local to a repository.** A refactor that ignores a codebase's existing naming, file organization, and control-flow habits can make the result less readable even when the individual function improves in isolation.

## Scoring

**Goal: 10/10.** When reviewing or writing code, rate it 0–10 on clarity. A 10/10 has names that read like prose, functions that do exactly one thing at one level of abstraction, guard clauses that eliminate nesting, structures whose responsibilities are obvious from their name alone, and changes that match the repository's established local conventions. Provide the current score and exactly what to change to reach 10/10.

## The Code Clarity Framework

Twelve principles for writing code that communicates clearly. Principles 1–8 are structural and language-agnostic; principles 9–12 cover file organization, named constants, mechanical formatting consistency, and the Electron process boundary — the areas most often neglected in TypeScript and Electron codebases:

---

### 1. Naming: Names Are Your Primary API

**Core concept:** Every name — variable, function, method, class, parameter — is a contract with the reader. Names should reveal intent, not implementation. The reader should never need to read a function's body to understand what it does or what a variable contains.

**Why it works:** In most codebases, 70%+ of identifiers are custom names. If those names are precise, the code reads like a domain description. If they are vague or misleading, every reader carries extra cognitive load reconstructing what the author meant.

**Key insights:**
- Functions that *do something* use verbs: `fetchUser()`, `validateInput()`, `syncViews()`
- Functions that *return a value* describe what they return: `activeUsers()`, `formattedDate()`, `bytesReceived()`
- Booleans use `is/has/can/should`: `isVisible`, `hasChildren`, `canSubmit`, `shouldRetry`
- Avoid double negatives: `hasElements` not `!isEmpty`, `isEnabled` not `!isDisabled`
- Class methods drop redundant type context: `line.length()` not `line.getLineLength()`
- Names should be proportional to scope: loop variables can be `i`, module-level state needs full names
- Abbreviations are only valid if universally understood in the domain (`url`, `id`, `dto`)

**Code applications:**

| Context | Unclear | Clear |
|---------|---------|-------|
| **Action function** | `handle()`, `process()`, `manage()` | `submitOrder()`, `parseResponse()`, `invalidateCache()` |
| **Return-value function** | `get()`, `fetch()`, `data()` | `currentUser()`, `pendingRequests()`, `errorMessage()` |
| **Boolean** | `flag`, `check`, `status`, `valid` | `isAuthenticated`, `hasUnreadMessages`, `canRetry` |
| **Swift bool** | `!list.isEmpty` | `list.hasElements` (extension) |
| **TS bool** | `if (!user.disabled)` | `if (user.isEnabled)` |
| **Class name** | `Manager`, `Handler`, `Helper`, `Util` | `RequestThrottler`, `TokenRefresher`, `PayloadEncoder` |
| **TS type suffix** | `data`, `info`, `thing` | role suffixes: `*Service`, `*Store`, `*Schema`, `*Registry`, `*Queue` |
| **Parameter** | `func send(_ data: Data, _ b: Bool)` | `func send(_ payload: Data, encrypted: Bool)` |
| **TS interface** | prefix every interface with `I` reflexively | name the role: `Session`, `HandlerDeps`; reserve `I`-prefix for abstract contracts (`ISessionManager`) only if the repo already does |

See: [references/naming-conventions.md](references/naming-conventions.md) and [references/typescript-and-electron.md](references/typescript-and-electron.md)

---

### 2. Early Return: Flatten the Happy Path

**Core concept:** Handle preconditions, error cases, and guard conditions at the top of a function and return immediately. The main logic of a function should be at the lowest indentation level, unobstructed by nested conditionals.

**Why it works:** Nesting forces the reader to maintain a mental stack of conditions. Each level of indentation multiplies the cognitive load. Early returns collapse this stack: by the time the reader reaches the main logic, all edge cases have been disposed of and forgotten. This is idiomatic in both Go and Swift.

**Key insights:**
- Check preconditions first, return/throw immediately if they fail
- The "happy path" — the normal case — lives at the leftmost indentation level
- `guard` in Swift and early `if err != nil { return }` in Go are the same philosophy
- Each early return is a complete thought: "this condition means we're done here"
- Avoid `else` after a `return` — it is always redundant and adds visual noise
- Deeply nested `if/else` is a signal to invert and exit early
- Exception: don't force early return when the two branches are genuinely symmetric in importance

**Code applications:**

| Context | Nested (avoid) | Early Return (prefer) |
|---------|---------------|----------------------|
| **Swift guard** | `if let user = user { if user.isActive { ... } }` | `guard let user, user.isActive else { return }` |
| **Validation** | `if isValid { if hasPermission { doWork() } }` | `guard isValid else { return }; guard hasPermission else { return }; doWork()` |
| **Error handling** | `if error == nil { if result != nil { use(result!) } }` | `guard error == nil, let result else { handle(error); return }; use(result)` |
| **Go style** | `if err == nil { if data != nil { process(data) } }` | `if err != nil { return err }; if data == nil { return ErrEmpty }; process(data)` |
| **TS guard** | `if (input) { if (input.type === 'keyDown') { ... } }` | `if (!input \|\| input.type !== 'keyDown') return; ...` |
| **TS dependency guard** | wrap a whole handler body in `if (windowManager) { ... }` | `if (!windowManager) return; ...` at the top — keeps the body flat |

See: [references/early-return.md](references/early-return.md)

---

### 3. Function Design: One Thing, One Level

**Core concept:** A function should do exactly one thing, at exactly one level of abstraction. The name should make that one thing obvious. If you need "and" to describe what a function does, it is doing two things.

**Why it works:** Functions that mix abstraction levels force the reader to context-switch between strategy and implementation detail. A function that orchestrates a workflow should not also contain the bit-manipulation logic that implements one step. Keeping levels consistent lets the reader choose the depth they need.

**Key insights:**
- The "one level of abstraction per function" rule: orchestration and implementation should not coexist
- If a function's body requires a comment to explain a section, that section is probably a function
- Function length is a symptom, not a cause: a 50-line function doing one thing is fine; a 5-line function doing three things is not
- Parameters: 0–2 is good, 3 is a warning, 4+ usually means a struct/object is needed
- Avoid output parameters — return values instead
- Avoid boolean flags that change the function's behavior — split into two functions
- Side effects should be in the name: `saveUser()` not `getUser()` when it also persists

**Code applications:**

| Problem | Example | Fix |
|---------|---------|-----|
| **Mixed levels** | `submitForm()` validates, serializes, and also builds the HTTP multipart boundary | Extract `buildMultipartBody()` |
| **Boolean flag param** | `render(view, animated: Bool)` does two different things | Split into `render(view)` and `renderAnimated(view)` |
| **Too many params** | `createUser(name:email:age:role:team:avatar:)` | Accept `UserConfiguration` struct |
| **Hidden side effect** | `currentUser()` hits the network | Rename `fetchCurrentUser()` or make it async |
| **"And" function** | `validateAndSave()` | Two functions: `validate()`, `save()` |

See: [references/function-design.md](references/function-design.md)

---

### 4. State Modeling: Related Flags Are a Hidden State Machine

**Core concept:** When several booleans, optionals, or task properties describe the same lifecycle, they are usually a state machine written in the least clear form. Make the lifecycle explicit with an enum or a small value type so impossible combinations cannot be represented.

**Why it works:** Scattered flags force every caller to keep them synchronized. A reader has to ask whether `isConnected`, `isEnded`, `isThinking`, `isSpeaking`, `activeTask`, and `pendingTask` can overlap, and bugs appear when one branch updates three of them but forgets the fourth. A single state value makes the valid states visible and gives each transition one place to live.

**Key insights:**
- If states are mutually exclusive, use an enum: `.idle`, `.connecting`, `.connected`, `.ended`
- If values must update together, group them in the enum payload or a small state struct
- Avoid lifecycle pairs like `isConnected` + `isEnded`; prefer `connectionState`
- Avoid activity flag piles like `isThinking`, `isSpeaking`, `isToolCalling`; prefer one `activity`
- Computed booleans such as `connectionState.isConnected` are fine when they are aliases for a real state, not independent storage
- Store a `Task` only when the owner must cancel, replace, or coalesce it later
- Fire-and-forget async work should usually be a local `Task` or `Task.detached`, not another stored optional property
- Do not use optional as a vague state marker when an enum case can name the state directly
- In SwiftUI apps, prefer one observable app/store object passed through the view tree over many tiny dependency-injection seams
- Put UI-facing snapshots in small context/configuration value types when several settings must be applied together
- Views should read state and send intent; lifecycle transitions belong in the store/controller that owns the state

**Code applications:**

| Problem | Unclear | Clear |
|---------|---------|-------|
| **Connection lifecycle** | `isConnected`, `isEnded`, `isReconnecting` | `enum ConnectionState { case disconnected, connecting, connected, reconnecting, ended }` |
| **Mutually exclusive activity** | `isThinking`, `isSpeaking`, `isToolCalling`, `isPlayingAudio` | `enum AssistantActivity { case idle, thinking, speaking, usingTool, playingAudio }` |
| **Optional-as-state** | `currentRequest: Request?`, `isLoading: Bool`, `error: Error?` | `enum LoadState { case idle, loading(Request), failed(Error), loaded(Result) }` |
| **Saved one-shot task** | `var refreshTask: Task<Void, Never>?` that is never cancelled | Create the task where the async work starts and let it finish |
| **Replaceable task** | Multiple optional task properties cancelled in bulk | Keep only tasks that must be cancelled/replaced, or wrap them in a named lifecycle state |

**SwiftUI data-flow check:**

```swift
@Observable final class AppStore {
  private(set) var connectionState: ConnectionState = .disconnected
  private(set) var assistantActivity: AssistantActivity = .idle
  var configuration = AppConfiguration()

  var isConnected: Bool { connectionState == .connected }

  func connect() {
    guard case .disconnected = connectionState else { return }
    connectionState = .connecting
    Task { await runConnection() }
  }
}
```

The important part is not the exact names. The important part is that `connectionState`,
`assistantActivity`, and `configuration` each have one owner and each represent one coherent
piece of state. Do not spread the same lifecycle across independent `@State`, `@Binding`,
`@Published`, or optional properties.

**TypeScript: discriminated (tagged) unions are the same idea.** Where Swift uses an `enum` with
associated values, TypeScript uses a discriminated union with a literal `type`/`kind`/`status`
discriminant. The payload that only exists in one state lives only in that variant, so impossible
combinations are unrepresentable and every consumer `switch`es on the tag:

```ts
// Avoid — three booleans + an optional that must be kept in sync
interface LoadState { isLoading: boolean; isError: boolean; data?: Result; error?: Error }

// Prefer — one value, each variant carries exactly its own payload
type LoadState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'loaded'; data: Result }
  | { status: 'failed'; error: Error }
```

Real Electron/React codebases push this far: a domain event type or route state becomes one union of
many tagged variants, with small type-guard helpers (`isLoaded(state)`) instead of scattered
booleans. Validate the boundary with a schema (zod `z.discriminatedUnion('type', [...])`) so the
runtime shape and the compile-time type cannot drift. See
[references/typescript-and-electron.md](references/typescript-and-electron.md).

---

### 5. Abstraction Levels: Hierarchy Must Be Consistent

**Core concept:** Every scope — module, class, function — should operate at a single, coherent level of abstraction. High-level code manages strategy and orchestration. Low-level code handles mechanism and detail. Mixing these two in the same scope is one of the most common and most damaging readability failures.

**Why it works:** When a reader encounters a high-level function, they expect to understand the overall flow without knowing implementation details. When they encounter a low-level function, they expect to see a specific, contained operation. Mixing the two forces the reader to maintain both strategic and tactical context simultaneously.

**Key insights:**
- Abstraction level corresponds to position in the call tree: higher up = more abstract
- Red flag: a function that calls other named functions AND also does raw data manipulation
- Red flag: a class that manages business logic AND also formats strings for display
- Naming reveals level: `orchestrateCheckout()` is high-level; `appendQueryParameter(_:to:)` is low-level
- The "step-down rule": reading a file top-to-bottom, each function should be followed by the functions it calls at the next level down
- In Swift: a ViewModel should not contain URL construction logic; a NetworkLayer should not contain business rules

**Code applications:**

| Context | Mixed Levels (avoid) | Consistent Levels (prefer) |
|---------|---------------------|---------------------------|
| **Checkout flow** | `checkout()` calls `applyDiscount()` then also does `price * (1 - rate)` math inline | `checkout()` calls `applyDiscount()` which internally computes the math |
| **ViewModel** | `loadData()` builds URLRequest, parses JSON, and updates `@Published` state | `loadData()` calls `repository.fetchItems()` and maps to display models |
| **Class responsibilities** | `OrderProcessor` manages order state AND formats the confirmation email HTML | Two classes: `OrderProcessor`, `OrderConfirmationFormatter` |
| **Swift view** | `body` property contains network calls and business logic | `body` only references `viewModel` properties and calls `viewModel` methods |
| **React component** | a `.tsx` component opens a WebSocket, parses JSON, and renders | component calls a typed hook/store; transport and parsing live below the component |
| **Electron renderer** | renderer reaches into Node `fs` / spawns a child process directly | renderer calls `window.electronAPI.x()`; Node-level work lives only in the main process |

See: [references/abstraction-levels.md](references/abstraction-levels.md) and [references/typescript-and-electron.md](references/typescript-and-electron.md)

---

### 6. Repository Conventions: Clarity Is Also Local

**Core concept:** Clarity is not purely universal. It is partly local to a codebase. A change is clearer when it extends the naming, file organization, layering, and UI construction patterns that the surrounding repository already uses. A technically sound refactor that ignores established local conventions often makes the codebase harder to read, not easier.

**Why it works:** Readers build fluency from repetition. If one repository consistently uses `Type+Feature.swift`, `is/has/can` booleans, and `guard`-heavy control flow, then a new contribution that follows those patterns is faster to parse than one introducing a different but equally reasonable style. Local consistency compounds readability.

**Key insights:**
- Start by sampling representative files before prescribing style changes
- Identify local suffixes and split patterns such as `+Delegate`, `+Layout`, `+DataSource`, or module-specific equivalents
- Preserve repository vocabulary unless it is actively misleading
- Prefer extending an existing style system over introducing a parallel one
- If the codebase already uses a clear exception to generic advice, follow the repository unless the exception is causing real defects
- Review comments and logs for tone: some codebases are terse and operational, others explanatory and educational
- Clarity advice should be phrased as: "make this read like the rest of the good code in the repository"

**Code applications:**

| Problem | Generic Advice | Repository-aware Advice |
|---------|----------------|-------------------------|
| **File split** | "Put everything in one file to reduce jumping" | Match the local split strategy if the repo already uses `Type+Feature.swift` consistently |
| **Naming** | "Rename all `Manager` types" | Keep existing role suffixes if the repository uses them with stable meaning |
| **UI layout** | "Use Auto Layout everywhere" | Preserve a mixed style if the codebase uses constraints for page shells and manual frame layout for high-frequency subviews |
| **Control flow** | "Use one preferred pattern globally" | Match local `guard`/early-return habits if they already make the module read consistently |
| **Comments** | "Add more comments" | Match the house style: brief rationale comments in terse codebases, richer guidance in instructional ones |

See: [references/repository-conventions.md](references/repository-conventions.md)

---

### 7. Error Boundaries: Handle Failures at the Right Level

**Core concept:** Error handling should communicate intent just as clearly as names do. User-facing
operations should surface failures. Best-effort background refreshes may fail quietly, but should
usually leave a trace. Internal invariants should fail loudly in debug instead of being silently
papered over.

**Why it works:** Broad `do/catch` blocks and casual `try?` make different failures look identical.
The reader cannot tell whether an error is expected, ignorable, user-visible, or a bug. Clear error
boundaries keep the happy path flat while preserving the information needed to debug production
behavior.

**Key insights:**
- Validate preconditions with early returns before entering `do/catch`
- Keep `do/catch` scoped to the operation that can actually throw
- Avoid `try?` when the failure changes user-visible state or lifecycle state
- Use `try?` only for genuinely disposable cleanup, probing, or cache-style best-effort work
- Background failures should usually trace or log at debug level rather than vanish
- Use assertions for impossible states in debug, then return safely in release
- Do not wrap optional binding inside a large `do/catch`; unwrap first, then run the throwing work

**Code applications:**

| Problem | Unclear | Clear |
|---------|---------|-------|
| **Validation inside catch** | `do { guard let data else { throw ... }; decode(data) } catch { ... }` | `guard let data else { return error }; do { decode(data) } catch { ... }` |
| **Silent lifecycle failure** | `try? startCapture()` | `do { try startCapture() } catch { state = .stopped; emit(error) }` |
| **Best-effort refresh** | `if let event = try? await refresh() { append(event) }` | `do { guard let event = try await refresh() else { return }; append(event) } catch { trace.debug(...) }` |
| **Impossible optional** | `if let session { try await session.send(...) }` in connected state | `guard let session else { assertionFailure(...); return }` |

**TypeScript error idioms:**

| Problem | Unclear | Clear |
|---------|---------|-------|
| **`unknown` catch** | `catch (e) { log(e.message) }` (`e` is `unknown`; may not be an `Error`) | `catch (e) { const msg = e instanceof Error ? e.message : 'Unknown error'; log(msg) }` |
| **Swallowed promise** | `doThing()` (floating promise, rejection vanishes) | `await doThing()` inside a scoped `try/catch`, or `void doThing().catch(reportError)` deliberately |
| **Expected-failure as throw** | `try { return parse(x) } catch { return null }` for routine validation | return a Result: `{ ok: true; value } \| { ok: false; error }` and `if (!r.ok) return` |
| **Missing cleanup on throw** | resource left open when the `try` body throws | release in `finally` (`server?.close()`), mirror of Swift `defer` |

---

### 8. Class and Struct Design: Single Responsibility, Honest Names

**Core concept:** A class or struct should have one reason to change. Its name should be specific enough that adding a second responsibility feels obviously wrong. Vague names like `Manager`, `Helper`, and `Util` are a signal that responsibilities have not been thought through.

**Why it works:** When a type has a single, clearly-named responsibility, it is easy to test, easy to modify, and easy to understand. When a type accumulates responsibilities over time, every change requires understanding the whole type, and changes to one responsibility risk breaking another.

**Key insights:**
- The name test: can you describe the class's responsibility in one sentence without using "and"?
- `Manager`, `Handler`, `Helper`, `Controller`, `Util` are almost always symptoms of unclear design — what specifically does it manage?
- Prefer small, named types over large bags of loosely related functionality
- In Swift: `struct` for value semantics and data, `class` for identity and lifecycle — this is a design decision, not just a performance one
- Low coupling: a type should communicate with as few other types as possible, passing minimal information
- High cohesion: everything inside a type should be strongly related to its single responsibility
- Watch for: types that are hard to name, types where methods don't use `self`, types that grow by accumulation

**Code applications:**

| Problem | Symptom | Fix |
|---------|---------|-----|
| **Vague name** | `UserManager` handles auth, profile edits, avatar upload, and session management | `AuthenticationService`, `ProfileEditor`, `AvatarUploader`, `SessionStore` |
| **Two reasons to change** | `OrderService` fetches orders AND sends notification emails | Split: `OrderRepository`, `OrderNotifier` |
| **Method without self** | A method on a class never references `self` | It belongs as a free function or on a different type |
| **Swift struct vs class** | Using `class` for a pure data container because "that's how we always do it" | `struct` for `Address`, `Coordinate`, `PriceBreakdown` — they are values |
| **TS type vs class** | a `class` with only a constructor and public fields and no methods that use `this` | a plain `type`/`interface` (or zod schema) — it is a value, not an object with behavior |
| **Accumulation** | `NetworkLayer` started as request-sending, now also caches, retries, and logs | Extract `RequestCache`, `RetryPolicy`, `RequestLogger` |

See: [references/class-struct-design.md](references/class-struct-design.md)

---

### 9. File Organization: One Object Per File

**Core concept:** A file should have one primary export — one class, one component, one schema, or one tightly-cohesive set of free functions on a single subject. The filename names that export. A reader who wants `WindowManager` opens `window-manager.ts`; they never scroll past three unrelated classes to find it.

**Why it works:** When the unit of the file matches the unit of thought, the file tree becomes a table of contents. Imports read as a dependency list, diffs are scoped to one concept, and merge conflicts shrink. Files that bundle many exports force the reader to load the whole file to understand any part of it, and they grow without resistance because "it's already this file's general area."

**Key insights:**
- One primary export per file; co-locate only what is meaningless on its own (a private helper, a props type for one component).
- Name the file after its export. Two common conventions, applied consistently: **kebab-case** for logic/utility modules (`document-store.ts`, `slug-sanitizer.ts`), **PascalCase** for component files that export a component (`DocumentCard.tsx`). Pick the repo's convention and keep it.
- Swift analog: one public type per file, `Type+Feature.swift` for focused extensions.
- Group by **domain folder**, then keep files flat inside it: `handlers/`, `domain/`, `services/`, `components/app-shell/`.
- Barrels (`index.ts`) are re-export tables of contents — explicit named re-exports, never implementation code. Prefer `export { X } from './x'` lines over a blanket `export *` so the package's public surface is visible and stable.
- A file that needs a `// MARK:`/`// ===== section =====` banner to separate two unrelated exports is two files.
- Interface/contract files can carry a clear suffix (`document-store-interface.ts`) when the repo separates abstract contracts from implementations.

**Code applications:**

| Problem | Unclear | Clear |
|---------|---------|-------|
| **Grab-bag module** | `utils.ts` exporting 20 unrelated helpers | Split by subject: `date-formatting.ts`, `url-building.ts`, `string-encoding.ts` |
| **Multiple classes per file** | `managers.ts` with `WindowManager`, `DocumentStore`, `DownloadManager` | One file each: `window-manager.ts`, `document-store.ts`, `download-manager.ts` |
| **Filename ≠ export** | `helpers.ts` exporting `sanitizeSlug` | `slug-sanitizer.ts` exporting `sanitizeSlug` |
| **Blanket barrel** | `export * from './everything'` hiding the surface | explicit `export { sanitizeSlug } from './slug-sanitizer'` |
| **Component + unrelated logic** | `DocumentCard.tsx` also defines the billing client | `DocumentCard.tsx` (component) + `billing-client.ts` (sibling) |

See: [references/file-organization.md](references/file-organization.md)

---

### 10. Named Constants: Hoist Fixed Values, Name Them, Scope Them

**Core concept:** A literal that carries meaning — a timeout, a retry ceiling, a path, a magic string, a channel name — should be a named constant hoisted to the top of its file, not an anonymous number buried at a call site. This is the cross-language form of Swift's `private`/`fileprivate let`: a value with a name and the narrowest scope that still serves every user in the file.

**Why it works:** `0.3` tells the reader nothing; `animationSettleDuration` tells them what it is and where to change it. Hoisting to the top centralizes the value so a change happens in one place, and naming it converts a mystery into documentation. Scoping it (`fileprivate`/`private`/module-local `const`) keeps it from leaking into the package's public surface where it would become an accidental API.

**Key insights:**
- TypeScript: `const SCREAMING_SNAKE_CASE` at module top, above the first function: `const MAX_LOG_AGE_MS = 24 * 60 * 60 * 1000`. Keep it un-exported unless other files genuinely need it — an un-exported `const` is file-private by default.
- Compose constants from constants so relationships stay visible: `const META_DEBOUNCE_MS = isWindows ? 300 : DEBOUNCE_MS`.
- Lock fixed maps/tuples with `as const` so TypeScript narrows to literal types and typos become compile errors: `const IPC_CHANNELS = { ... } as const`. Derive types from the data with `typeof FIELDS[number]` instead of restating them.
- Swift: `private let maxRetryCount = 3` / `private enum Layout { static let spacing: CGFloat = 12 }`. Use `fileprivate`/`private` deliberately; a constant only one type needs should not be internal.
- A literal that appears twice, or that a maintainer would need a comment to understand, has earned a name.
- Keep the constant at the lowest scope that covers its readers: function-local if one function uses it, file-private if the file uses it, exported only if the contract is shared.

**Code applications:**

| Problem | Unclear | Clear |
|---------|---------|-------|
| **Magic number** | `setTimeout(fn, 100)` | `const DEBOUNCE_MS = 100; setTimeout(fn, DEBOUNCE_MS)` |
| **Repeated magic string** | `'documents:get'` typed in three files | `IPC_CHANNELS.documents.GET` from one `as const` registry |
| **Leaked constant** | `export const RETRY_LIMIT = 3` used only in this file | drop `export` — keep it module-private |
| **Stringly-typed set** | a `string[]` of field names restated as a type | `const FIELDS = [...] as const; type Field = typeof FIELDS[number]` |
| **Swift inline literal** | `DispatchQueue.main.asyncAfter(deadline: .now() + 0.3)` | `private let settleDelay: TimeInterval = 0.3` |

See: [references/file-organization.md](references/file-organization.md) (constants section) and [references/typescript-and-electron.md](references/typescript-and-electron.md).

---

### 11. Mechanical Consistency: Defer Formatting to the Formatter

**Core concept:** Indentation, quote style, semicolons, trailing commas, line width, and import wrapping are clarity decisions too — but they should be made *once*, by a formatter, not re-litigated per file. Adopt Prettier (or the repo's configured formatter) and let it own all mechanical style so humans spend their attention on names and structure, not whitespace.

**Why it works:** Inconsistent mechanical style adds noise to every diff and every read — the eye trips on a stray quote-style change the same way it trips on a bad name. A formatter makes mechanical style *invisible*: every file looks the same, so genuine logic changes are the only thing a diff shows. The Prettier philosophy is the point — **stop debating style, encode it, run it on save and in CI.**

**Key insights:**
- Run Prettier on TypeScript/JS/JSON/CSS/Markdown; pair it with a linter (ESLint) for correctness rules, not style — the formatter owns style, the linter owns bugs.
- Prettier's opinionated defaults are a fine baseline: 80-col print width, 2-space indent, semicolons, double quotes, trailing commas (`es5`/`all`), `arrowParens: always`. Override only with a committed `.prettierrc`, and then never by hand.
- Make it non-negotiable: format-on-save in the editor, plus a CI check (`prettier --check` / `eslint`) so unformatted code cannot merge. A `build` script that runs `lint` first is the enforcement point.
- Match what already exists. If a file is semicolon-less and single-quoted because that's the repo's Prettier config, follow the config — do not hand-introduce the "default" style. The formatter's config is the source of truth, not your preference.
- Do not hand-align columns, hand-wrap arguments, or fight the formatter with `// prettier-ignore` except for genuinely tabular data where alignment carries meaning.
- Consistency beats personal preference: a slightly-less-preferred style applied uniformly reads better than two "better" styles mixed.

**Code applications:**

| Problem | Unclear | Clear |
|---------|---------|-------|
| **Mixed quote/semicolon style** | some files `'x'` no-semi, others `"x";` | one `.prettierrc`, formatter run in CI — the repo has exactly one style |
| **Hand-wrapped arguments** | manually aligned multi-line call that breaks on the next edit | let Prettier wrap at print width; re-runs stay stable |
| **Style churn in diffs** | a logic PR also reformats 40 lines by hand | format separately (or never by hand); logic diffs stay readable |
| **Style debate in review** | reviewer comments on quote style | not a review topic — the formatter already decided |
| **Editor drift** | each contributor's editor formats differently | committed config + format-on-save + `--check` gate |

See: [references/formatting-consistency.md](references/formatting-consistency.md)

---

### 12. Electron Boundaries: Main, Preload, and Renderer Are Different Worlds

**Core concept:** An Electron app is three programs with three trust levels — the **main** process (Node, full OS access), the **renderer** (a sandboxed browser running your UI), and the **preload** (the only bridge between them). Clarity in Electron means each world contains only what belongs to it, and they talk through one typed, allowlisted surface — never by reaching across the boundary.

**Why it works:** When the renderer can touch Node, filesystem, native capture, or tokens directly, the boundary that exists for security and testability dissolves, and every file becomes a potential cross-process tangle. A single typed bridge makes the contract between UI and system legible: a reader sees exactly what the renderer is allowed to ask for, and the type system enforces it. (This is the standard Electron security posture: the renderer must not access Node or secrets; a minimal typed preload allowlist is the only bridge.)

**Key insights:**
- **Separate the three worlds by directory and never blur them:** `src/main/`, `src/preload/`, `src/renderer/`, with cross-world DTOs/types in a `src/shared/`. A file's directory tells the reader its trust level.
- **The renderer never imports Node.** No `fs`, `child_process`, `electron` main APIs, native modules, or secret material in renderer code. If the UI needs that work, it asks the main process to do it.
- **Expose one typed API over `contextBridge`, not raw IPC.** A single `contextBridge.exposeInMainWorld('appAPI', api)` with a fully-typed interface (`window.appAPI.createSession(): Promise<Session>`) beats sprinkling `ipcRenderer.invoke('some-string')` through the UI. Type the channel registry once (`as const`) and let both sides share it, so a typo is a compile error.
- **Keep `contextIsolation` on and `nodeIntegration` off.** The preload's allowlist is the security contract; widen it deliberately, one method at a time, never with a blanket pass-through.
- **Put portable logic in process-neutral packages** (`packages/core`, `packages/shared`) with no Electron, DOM, or OS imports, so it is testable and reusable. Electron-specific handlers (BrowserWindow, autoUpdater, native dialogs) stay in `main/`.
- **Name handlers and channels by namespace:** `sessions:get`, `window:close` — a `namespace:action` string read straight from a shared `as const` map, so the wire protocol is self-documenting.
- **Secrets and tokens live in the main process** behind secure storage, never in renderer `localStorage` or a renderer-visible global.

**Code applications:**

| Problem | Unclear / unsafe | Clear |
|---------|------------------|-------|
| **Renderer reaches into Node** | `import { readFile } from 'fs'` inside a `.tsx` | `await window.appAPI.readDocument(id)` → main does the I/O |
| **Stringly-typed IPC** | `ipcRenderer.invoke('docments:get', id)` (typo ships) | `IPC_CHANNELS.documents.GET` from a shared `as const`; typed client proxy |
| **Untyped bridge** | `contextBridge.exposeInMainWorld('api', { invoke })` returning `any` | expose a typed `interface AppAPI` with concrete `Promise<T>` methods |
| **Logic stuck in main** | business rules written inline in an IPC handler | rule lives in `packages/core`; the handler just calls it |
| **Token in renderer** | `localStorage.setItem('token', t)` | token in main-process secure storage; renderer never sees it |
| **nodeIntegration on** | renderer has full Node for "convenience" | `contextIsolation: true`, `nodeIntegration: false`, preload allowlist |

See: [references/typescript-and-electron.md](references/typescript-and-electron.md) (Electron section)

---

## Common Mistakes

| Mistake | Why It Fails | Fix |
|---------|-------------|-----|
| **Verb-less noun names** | `UserData`, `RequestInfo` — what does it *do*? | Name what it represents precisely: `AuthenticatedUser`, `PendingRequest` |
| **Negated booleans** | `!isNotEnabled` requires two mental inversions | Always name the positive state: `isEnabled` |
| **Else after return** | `if error { return }; else { doWork() }` — the `else` is noise | Remove it: the return already guarantees the else case |
| **Flag parameters** | `save(user, force: true)` — the caller knows two behaviors exist | Split: `save(user)`, `forceSave(user)` |
| **Mixed abstraction** | `processOrder()` calls `applyDiscount()` but also does `price * rate` inline | Keep one level: either all calls or all computation |
| **Accumulating types** | Adding to `UserManager` because "it's user-related" | Ask: would this responsibility cause the class to change for a different reason? If yes, it belongs elsewhere |
| **Repository-blind refactors** | Improving one function while making it read unlike the surrounding module | Preserve the local naming, file split, and control-flow patterns used by the rest of the repository |
| **Comment-instead-of-rename** | `// this is the active users list` above `var data` | The name should be `activeUsers`; the comment is evidence of a naming failure |
| **Long parameter lists** | `createRequest(url:method:headers:body:timeout:retry:)` | Group into `RequestConfiguration` |
| **Boolean state piles** | Several `is*` flags must be updated together or cannot validly overlap | Replace them with one enum or state value that names each valid state |
| **Stored one-shot tasks** | Keeping `Task?` properties for work that is never cancelled or replaced | Keep task storage only for real lifecycle ownership; otherwise launch locally and let it finish |
| **Casual `try?`** | Important lifecycle or user-facing failures disappear | Use scoped `do/catch`; trace best-effort failures and surface user-facing failures |
| **Broad error wrapper** | Validation, decoding, dispatch, and side effects all share one catch block | Early-return preconditions first, then catch only the throwing operation |
| **Grab-bag file** | `utils.ts` / `managers.ts` bundles many unrelated exports | One primary export per file, named after the file |
| **Filename hides export** | `helpers.ts` exports `sanitizeSlug` | Rename the file to its export: `slug-sanitizer.ts` |
| **Inline magic literal** | `setTimeout(fn, 100)`, `'documents:get'` typed in three places | Hoist to a named module-top `const`; lock maps with `as const` |
| **Leaked constant** | `export const` for a value only this file uses | Drop `export` — keep it file-private |
| **Hand-formatting** | Manually aligned/wrapped code, mixed quotes and semicolons | Adopt Prettier; format on save and in CI; never style by hand |
| **`unknown` catch misuse** | `catch (e) { e.message }` assuming `e` is an `Error` | `e instanceof Error ? e.message : 'Unknown error'` |
| **Renderer touches Node** | `import 'fs'` or a token in a `.tsx` renderer file | Go through the typed preload bridge; do Node work in main |
| **Stringly-typed IPC** | `ipcRenderer.invoke('typo:channel')` | Shared `as const` channel map + typed client; typos fail to compile |

---

## Quick Diagnostic

| Question | If No | Action |
|----------|-------|--------|
| Can you read the function name and know what it does without reading the body? | Name is vague or misleading | Rename to reveal intent |
| Is the main logic of every function at the leftmost indentation level? | Nested conditionals obscure flow | Invert conditions and return early |
| Does each function do exactly one thing? | Function has "and" in its description | Split at the "and" |
| Does each function operate at one level of abstraction? | Orchestration mixed with implementation | Extract the lower-level operations into named functions |
| Does every boolean variable start with `is`, `has`, `can`, or `should`? | Booleans require context to interpret | Rename with appropriate prefix |
| Are several booleans or optionals describing one lifecycle? | Hidden state machine with possible impossible combinations | Replace with an enum or grouped state value |
| Is a `Task` stored because the owner must cancel or replace it later? | Task storage is just bookkeeping noise | Make it local/fire-and-forget, or name the lifecycle that owns it |
| Is `try?` hiding a failure that changes lifecycle or UI state? | The code is silent in exactly the place users need feedback | Use scoped `do/catch`, surface or trace according to the boundary |
| Are precondition guards inside a broad `do/catch`? | Expected validation is mixed with exceptional failure | Move guards before `do/catch` and return early |
| Can you describe every class responsibility in one sentence without "and"? | Type has multiple responsibilities | Split by responsibility |
| Are `Manager`, `Helper`, or `Util` in any type names? | Responsibilities are not well-defined | Replace with specific names |
| Does any function take more than 3 parameters? | Interface is too wide | Group related parameters into a type |
| Does each file have one primary export named after the file? | File is a grab-bag, or filename hides its export | Split by subject; rename file to its export |
| Is every meaningful literal a named, top-of-file constant at the narrowest scope? | Magic numbers/strings inline, or constants leaked via `export` | Hoist + name; `as const` for fixed maps; drop needless `export` |
| Is mechanical style owned by a formatter run in CI? | Quotes/semicolons/wrapping vary; style churns diffs | Adopt Prettier; format on save and gate in CI |
| In Electron, does the renderer stay free of Node, secrets, and raw IPC? | Renderer imports `fs`/tokens or calls stringly-typed IPC | Route through one typed `contextBridge` allowlist; Node work in main |
| Does the proposed refactor still read like the surrounding repository? | Change is clearer in isolation but less native in context | Re-align naming, file organization, and control-flow with the local house style |

---

## Review Workflow

Use this sequence when the skill is applied:

1. Sample 3–5 representative files from the same module or layer before giving advice.
2. Infer local conventions: naming suffixes, file split patterns, comment density, control-flow style, and UI construction habits.
3. Score current clarity from 0–10.
4. Identify the smallest set of changes that would move the code toward 10/10.
5. Prefer repository-aligned fixes over generic rewrites when both are equally clear.
6. If a repository convention is genuinely harmful, say so explicitly and justify breaking it.

### Suggested Output Format

When reviewing code with this skill, prefer this structure:

1. `Clarity score: X/10`
2. `What is already clear`
3. `What is confusing`
4. `What to change next`
5. `Repository conventions to preserve`

This keeps the advice concrete and makes the tradeoff between universal clarity and local consistency explicit.

---

## Reference Files

- [naming-conventions.md](references/naming-conventions.md): Full naming rules by category — functions, booleans, classes, parameters, with Swift/Go/TypeScript examples
- [early-return.md](references/early-return.md): Guard clause pattern, Swift `guard`, Go idioms, when not to use early return
- [function-design.md](references/function-design.md): Single responsibility, abstraction level consistency, parameter design, side effects
- [repository-conventions.md](references/repository-conventions.md): How to infer a repository's local house style and keep clarity improvements aligned with it
- [abstraction-levels.md](references/abstraction-levels.md): Step-down rule, mixing levels as a code smell, organizing call hierarchies
- [class-struct-design.md](references/class-struct-design.md): Single responsibility, Swift struct vs class decision, coupling and cohesion, naming types
- [file-organization.md](references/file-organization.md): One object per file, file naming, folder/barrel structure, and module-level named constants
- [formatting-consistency.md](references/formatting-consistency.md): The Prettier pattern — deferring mechanical style to a formatter, config, and CI enforcement
- [typescript-and-electron.md](references/typescript-and-electron.md): TypeScript naming/types/discriminated unions/error handling and the Electron main/preload/renderer boundary with typed IPC

---

## Further Reading

- [*"Clean Code"*](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) by Robert C. Martin — chapter-level coverage of naming, functions, and classes
- [*"A Philosophy of Software Design"*](https://www.amazon.com/Philosophy-Software-Design-2nd/dp/173210221X) by John Ousterhout — deep modules and information hiding
- [common-coding-conventions](https://tum-esi.github.io/common-coding-conventions/) — language-agnostic naming and abstraction guide
- [Guard Clauses - boot.dev](https://blog.boot.dev/clean-code/guard-clauses/) — guard clause pattern with cross-language examples
