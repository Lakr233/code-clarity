---
name: code-clarity
description: 'Write readable, intention-revealing code through precise naming, consistent abstraction levels, repository-aware conventions, and the early-return pattern. Use when naming feels off, logic is hard to follow, functions are doing too much, nested conditionals are making flow hard to trace, or a refactor should preserve the codebase''s local style. Covers function/method naming, boolean naming, guard clauses, abstraction-level consistency, repository conventions, and class/struct responsibilities. Swift-primary with Go, TypeScript, and Python equivalents throughout.'
license: MIT
metadata:
  author: qaq
  version: "1.1.0"
---

# Code Clarity Framework

A practical framework for writing code that communicates intent clearly. The central thesis: **a developer reads code far more than they write it**, so every naming and structural decision is a communication decision. Code that requires a reader to reconstruct the author's mental model has failed at its primary job.

This framework focuses on the micro-level of software design — the decisions made at the function, method, and class level — and complements macro-level architecture thinking.

## Core Principles

**Code is written once but read hundreds of times.** Every name, every function boundary, every conditional structure is a message to the next reader (usually yourself, six months later). Clarity is not a style preference — it is a correctness property: unclear code is one misunderstanding away from a bug.

**Clarity is partly local to a repository.** A refactor that ignores a codebase's existing naming, file organization, and control-flow habits can make the result less readable even when the individual function improves in isolation.

## Scoring

**Goal: 10/10.** When reviewing or writing code, rate it 0–10 on clarity. A 10/10 has names that read like prose, functions that do exactly one thing at one level of abstraction, guard clauses that eliminate nesting, structures whose responsibilities are obvious from their name alone, and changes that match the repository's established local conventions. Provide the current score and exactly what to change to reach 10/10.

## The Code Clarity Framework

Six principles for writing code that communicates clearly:

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
| **Class name** | `Manager`, `Handler`, `Helper`, `Util` | `RequestThrottler`, `TokenRefresher`, `PayloadEncoder` |
| **Parameter** | `func send(_ data: Data, _ b: Bool)` | `func send(_ payload: Data, encrypted: Bool)` |

See: [references/naming-conventions.md](references/naming-conventions.md)

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

### 4. Abstraction Levels: Hierarchy Must Be Consistent

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

See: [references/abstraction-levels.md](references/abstraction-levels.md)

---

### 5. Repository Conventions: Clarity Is Also Local

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

### 6. Class and Struct Design: Single Responsibility, Honest Names

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
| **Accumulation** | `NetworkLayer` started as request-sending, now also caches, retries, and logs | Extract `RequestCache`, `RetryPolicy`, `RequestLogger` |

See: [references/class-struct-design.md](references/class-struct-design.md)

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

---

## Quick Diagnostic

| Question | If No | Action |
|----------|-------|--------|
| Can you read the function name and know what it does without reading the body? | Name is vague or misleading | Rename to reveal intent |
| Is the main logic of every function at the leftmost indentation level? | Nested conditionals obscure flow | Invert conditions and return early |
| Does each function do exactly one thing? | Function has "and" in its description | Split at the "and" |
| Does each function operate at one level of abstraction? | Orchestration mixed with implementation | Extract the lower-level operations into named functions |
| Does every boolean variable start with `is`, `has`, `can`, or `should`? | Booleans require context to interpret | Rename with appropriate prefix |
| Can you describe every class responsibility in one sentence without "and"? | Type has multiple responsibilities | Split by responsibility |
| Are `Manager`, `Helper`, or `Util` in any type names? | Responsibilities are not well-defined | Replace with specific names |
| Does any function take more than 3 parameters? | Interface is too wide | Group related parameters into a type |
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

---

## Further Reading

- [*"Clean Code"*](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) by Robert C. Martin — chapter-level coverage of naming, functions, and classes
- [*"A Philosophy of Software Design"*](https://www.amazon.com/Philosophy-Software-Design-2nd/dp/173210221X) by John Ousterhout — deep modules and information hiding
- [common-coding-conventions](https://tum-esi.github.io/common-coding-conventions/) — language-agnostic naming and abstraction guide
- [Guard Clauses - boot.dev](https://blog.boot.dev/clean-code/guard-clauses/) — guard clause pattern with cross-language examples
