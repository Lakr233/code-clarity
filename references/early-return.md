# Early Return and Guard Clauses

The early return pattern — also known as guard clauses or the bouncer pattern — is the practice of handling preconditions, edge cases, and error conditions at the top of a function with an immediate return, so the function's main logic runs at the base indentation level, unobstructed.

This is idiomatic in both Go and Swift, and represents the same underlying philosophy in both languages.

## The Core Problem: Arrow Code

Deeply nested conditionals create "arrow code" — the indented shape of the code forms an arrow pointing right. Each level of nesting forces the reader to maintain a mental stack of active conditions.

```swift
// Arrow code — avoid
func processOrder(_ order: Order?) {
    if let order = order {
        if order.isValid {
            if currentUser.canPlaceOrders {
                if inventory.has(order.items) {
                    // main logic buried 4 levels deep
                    charge(order)
                } else {
                    showOutOfStock()
                }
            } else {
                showPermissionError()
            }
        } else {
            showValidationError()
        }
    }
}
```

The main logic (`charge(order)`) is 4 levels deep. The reader must hold all four conditions in mind simultaneously to understand the code.

## The Solution: Exit Early, Stay Flat

```swift
// Early return — prefer
func processOrder(_ order: Order?) {
    guard let order else { return }
    guard order.isValid else { showValidationError(); return }
    guard currentUser.canPlaceOrders else { showPermissionError(); return }
    guard inventory.has(order.items) else { showOutOfStock(); return }

    // main logic at base level
    charge(order)
}
```

The reader can scan the guards linearly, and the main logic is immediately visible at the base level. Each guard is a complete thought that can be read and dismissed.

---

## Swift: `guard` Statements

Swift's `guard` statement is a first-class language construct for early return. It was designed specifically for this pattern.

### Basic guard

```swift
guard condition else { return }
guard condition else { throw SomeError.reason }
```

### guard let / guard var

Unwrap optionals and bind them for the rest of the scope:

```swift
func displayProfile(for userId: String?) {
    guard let userId else { return }          // userId is now String, not String?
    guard let user = users[userId] else {
        showUserNotFound()
        return
    }

    // userId and user are both available and non-optional here
    render(user)
}
```

### guard with multiple conditions

Chain conditions in a single guard to reduce visual noise:

```swift
guard
    let user = currentUser,
    user.isActive,
    user.hasCompletedOnboarding
else {
    redirectToOnboarding()
    return
}

// user is available, active, and onboarded here
loadDashboard(for: user)
```

### guard in async context

```swift
func fetchAndDisplay(id: String) async {
    guard !id.isEmpty else { return }

    do {
        let user = try await userRepository.fetch(id: id)
        guard user.isActive else {
            await MainActor.run { showDeactivatedNotice() }
            return
        }
        await MainActor.run { render(user) }
    } catch {
        await MainActor.run { showError(error) }
    }
}
```

---

## Go: Idiomatic Early Return

Go's convention of returning `(value, error)` pairs combined with `if err != nil { return }` achieves the same pattern:

```go
// Go idiomatic early return
func processOrder(orderID string) error {
    if orderID == "" {
        return errors.New("orderID cannot be empty")
    }

    order, err := fetchOrder(orderID)
    if err != nil {
        return fmt.Errorf("fetch order: %w", err)
    }

    if !order.IsValid() {
        return ErrInvalidOrder
    }

    if err := chargeCard(order); err != nil {
        return fmt.Errorf("charge card: %w", err)
    }

    return nil
}
```

Each error case is handled immediately and the function exits. The reader never needs to track "what happens if fetch fails?" — it returned. The success path is at the base indentation level.

---

## TypeScript: Early Return Pattern

```typescript
// TypeScript — same philosophy
async function processOrder(orderId: string | null): Promise<void> {
    if (!orderId) return;

    const order = await fetchOrder(orderId);
    if (!order) {
        showOrderNotFound();
        return;
    }

    if (!order.isValid) {
        showValidationError();
        return;
    }

    await chargeCard(order);
}
```

---

## The `else` After `return` Anti-Pattern

Once a block ends with `return`, `throw`, or `continue`, the `else` branch is redundant. It adds visual indentation without adding logic.

```swift
// Redundant else — avoid
func label(for status: Status) -> String {
    if status == .active {
        return "Active"
    } else {
        return "Inactive"
    }
}

// Remove the else — prefer
func label(for status: Status) -> String {
    if status == .active {
        return "Active"
    }
    return "Inactive"
}
```

This applies recursively: if every branch of an if-else returns, rewrite as sequential guards.

---

## When NOT to Use Early Return

Early return is not always the right choice. Use judgment:

### Don't force asymmetry

If two branches are genuinely equal in importance and neither is a "failure case", an if-else communicates parity clearly:

```swift
// This is fine — both branches are valid outcomes of equal weight
if user.isPremium {
    showPremiumDashboard()
} else {
    showFreeDashboard()
}
```

Forcing one branch into an early return would make it look like an error case.

### Don't chain many tiny guards for simple logic

If all you need is a two-condition check before a single line of logic, an `if` with a compound condition reads fine:

```swift
// Acceptable for simple cases
if isReady && hasData {
    displayContent()
}

// Over-engineered early return
guard isReady else { return }
guard hasData else { return }
displayContent()
```

### Nested resources requiring cleanup

If early return would skip important cleanup (resource release, deferred work), use `defer` in Swift/Go, or restructure:

```swift
func readFile(at path: String) throws -> Data {
    let file = try openFile(at: path)
    defer { file.close() }  // always runs, even on early return

    guard file.size < maxFileSize else { throw FileError.tooLarge }
    return try file.readAll()
}
```

---

## Decision Guide

| Condition | Pattern |
|-----------|---------|
| Nil check / optional unwrap | `guard let x else { return }` |
| Precondition (function can't proceed) | `guard condition else { return }` |
| Error from async operation | `guard let result, error == nil else { handle(error); return }` |
| Two branches of equal importance | `if / else` |
| Complex multi-path logic | Consider breaking into smaller functions first |
| Resource cleanup required | `defer` + early return |

---

## Swift vs Go Comparison

| Scenario | Swift | Go |
|----------|-------|-----|
| Nil / missing value | `guard let x else { return }` | `if x == nil { return ErrMissing }` |
| Precondition check | `guard isValid else { return }` | `if !isValid { return ErrInvalid }` |
| Error from call | `let x = try call()` + `catch` | `x, err := call(); if err != nil { return err }` |
| Bind for scope | `guard let x else { return }` (x available below) | `x, err := ...; if err != nil { return }` (x available below) |
| Multiple conditions | `guard a, b, c else { return }` | Sequential `if` blocks |
