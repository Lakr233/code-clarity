# Formatting Consistency: Let the Tool Own Mechanical Style

Mechanical style — indentation, braces, `var` usage, `using` placement, line width, wrapping — is a clarity concern, but it should be decided **once, in a committed configuration enforced by a tool**, and then rarely revisited. Human attention goes to names and structure.

---

## The core idea

Inconsistent mechanical style is noise, and hand-formatting pollutes diffs: a one-line logic change arrives wrapped in forty lines of reflow, and the reviewer can no longer see what changed.

**The stance:** stop debating style, encode it in config, run it on save and in CI.

---

## The .NET toolchain

| Tool | Owns | Notes |
|---|---|---|
| **`.editorconfig`** | The single source of truth for C# style and analyzer severities | Commit at repo root; IDE, `dotnet format`, and build all read it |
| **`dotnet format`** | Applies `.editorconfig` whitespace + code-style + analyzer fixes | Built into the SDK; `dotnet format --verify-no-changes` for CI |
| **CSharpier** (optional) | Opinionated Prettier-like formatter for C# | Use *instead of* `dotnet format whitespace` if the repo adopts it; don't run both on whitespace |
| **Roslyn analyzers / StyleCop.Analyzers** | Correctness and intent | Severity in `.editorconfig`; build fails on errors |

Formatter vs analyzer: the formatter owns things with no semantic meaning (whitespace, braces, wrapping); analyzers own things a human can get wrong (unused variables, non-exhaustive switches, unforwarded `CancellationToken`). Don't use review comments for either.

---

## A baseline `.editorconfig`

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4
max_line_length = 140              # CSharpier: printWidth; a wrap target, not a hard wall
insert_final_newline = true

csharp_style_namespace_declarations = file_scoped:error
csharp_using_directive_placement = outside_namespace:warning
dotnet_sort_system_directives_first = true

csharp_prefer_braces = true:warning
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_expression_bodied_properties = true:suggestion
csharp_style_prefer_switch_expression = true:suggestion
csharp_style_prefer_pattern_matching = true:suggestion

dotnet_style_readonly_field = true:warning
dotnet_diagnostic.IDE0005.severity = warning     # unnecessary using
dotnet_diagnostic.CS8509.severity = error        # non-exhaustive switch
dotnet_diagnostic.CA2016.severity = warning      # forward CancellationToken

# Private fields: _camelCase
dotnet_naming_rule.private_fields.symbols = private_fields
dotnet_naming_rule.private_fields.style = underscore_camel
dotnet_naming_rule.private_fields.severity = warning
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private
dotnet_naming_style.underscore_camel.capitalization = camel_case
dotnet_naming_style.underscore_camel.required_prefix = _
```

Adapt to the repository; if it already has an `.editorconfig`, that file wins.

---

## Prefer comfortable line width

Line width is a wrap *target*, not a reason to fragment a readable statement by hand. Ordinary calls, conditions, and assignments stay on one line when that is the easier scan.

```csharp
// Prefer — one statement, one glance
var invoice = await _db.Invoices.SingleAsync(i => i.Id == id && i.TenantId == tenantId, ct);

// Avoid — wrapping to a column count
var invoice =
    await _db.Invoices
        .SingleAsync(
            i => i.Id == id && i.TenantId == tenantId,
            ct);
```

Break at semantic groups: one LINQ operator per line in a long query, one clause per line in a complex condition, one argument per line in a long object initializer. Never leave `=`, `return`, `await`, or a lone `(` stranded on its own line.

```csharp
var overdue = await _db.Invoices
    .Where(i => i.TenantId == tenantId)
    .Where(i => i.Status == InvoiceStatus.Issued && i.DueDate < today)
    .OrderBy(i => i.DueDate)
    .Select(i => new OverdueInvoice(i.Id, i.Number, i.DueDate, i.Total))
    .ToListAsync(ct);
```

80 columns is a historical terminal width. Use 120–140 for application code. If a configured formatter repeatedly fragments comfortable statements, fix the config rather than hand-wrapping call sites.

---

## Indentation shows structure

Indentation is how a reader sees the shape of code before reading a word. Every level means "this only runs inside that," so the left edge must match the real logical structure, and nothing else.

**Depth is a cost.** Each level is one more condition the reader has to keep in mind. Aim for no more than two or three levels inside a method body. When it gets deeper, flatten it with guard clauses ([early-return.md](early-return.md)), pattern matching, or a method that names the inner operation. Don't flatten it by reformatting.

```csharp
// Avoid — the work sits four levels deep
public async Task SendRemindersAsync(CancellationToken ct)
{
    foreach (var invoice in await _invoices.ListIssuedAsync(ct))
    {
        if (invoice.DueDate < _clock.Today)
        {
            if (invoice.Customer.Email is { } email)
            {
                await _mail.SendReminderAsync(email, invoice, ct);
            }
        }
    }
}

// Prefer — filter first, the work stays at one level
public async Task SendRemindersAsync(CancellationToken ct)
{
    var overdue = (await _invoices.ListIssuedAsync(ct)).Where(i => i.DueDate < _clock.Today);

    foreach (var invoice in overdue)
    {
        if (invoice.Customer.Email is not { } email)
            continue;

        await _mail.SendReminderAsync(email, invoice, ct);
    }
}
```

**Continuation lines indent one level, not to a column.** When a statement wraps, indent the next line one level (4 spaces) from where the statement starts. Don't align it under an opening parenthesis or `=`. Column alignment shifts every time something is renamed, makes diffs noisy, and pushes code toward the right edge.

```csharp
// Prefer
var summary = new InvoiceSummary(
    invoice.Number,
    invoice.Customer.DisplayName,
    invoice.Total);

// Avoid — aligned to the parenthesis; renaming InvoiceSummary re-indents every line
var summary = new InvoiceSummary(invoice.Number,
                                 invoice.Customer.DisplayName,
                                 invoice.Total);
```

**Blocks open and close at the same level.** The lines inside switch expressions, object and collection initializers, and lambda bodies go one level in. The closing brace or bracket goes back to the column where the statement started, so the end of the block is easy to find.

```csharp
var label = invoice.Status switch
{
    InvoiceStatus.Draft => "Concept",
    InvoiceStatus.Issued => "Verzonden",
    InvoiceStatus.Paid => "Betaald",
    _ => throw new UnreachableException(),
};
```

**Blank lines are vertical indentation.** Use one blank line to separate steps within a method (load, decide, act). Don't use blank lines everywhere or nowhere; both hide where one step ends and the next begins.

Spaces vs tabs and the indent size belong in `.editorconfig` (`indent_style = space`, `indent_size = 4`) and are never hand-edited. The decisions above are the ones the formatter can't make for you: the formatter controls how much whitespace there is, and you control how deep the structure goes.

---

## Make it non-negotiable: save + CI

1. **Editor:** format on save (Rider, Visual Studio "Code Cleanup on save", VS Code C# Dev Kit) reading the committed `.editorconfig`.
2. **Build:** `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>` so IDE rules run in `dotnet build`.
3. **CI gate:**

```bash
dotnet format --verify-no-changes
# or, if the repo uses CSharpier:
dotnet csharpier check .
```

---

## Match the config, not your preference

- If the repo's `.editorconfig` says `var` only when apparent, follow it — don't "fix" it.
- Never reformat unrelated lines in a logic PR. Do formatting-only changes in a separate commit, and add that commit to `.git-blame-ignore-revs`.
- Accept isolated formatter output; when the same output repeatedly harms readability, change the config.

---

## When to override the formatter (rarely)

Use `// csharpier-ignore` or `#pragma warning disable` with a reason only when alignment carries meaning the formatter would destroy — lookup tables, matrices, test vectors:

```csharp
// csharpier-ignore
private static readonly int[] Kernel =
[
    0, 1, 0,
    1, 4, 1,
    0, 1, 0,
];
```

"I like my arguments aligned this way" does not qualify.

---

## Diagnostic

| Symptom | Fix |
|---|---|
| Files disagree on braces/`var`/usings | Commit an `.editorconfig`; run `dotnet format` once in a separate commit |
| Logic PRs carry reformatting | Format on save; separate formatting commits; `.git-blame-ignore-revs` |
| Reviewers comment on whitespace | Move the rule into `.editorconfig` |
| Style rules only enforced in the IDE | `EnforceCodeStyleInBuild` + CI `--verify-no-changes` |
| Ordinary calls wrapped to ~80 columns | Keep on one line; raise `max_line_length` / `printWidth` |
| Method body four or more levels deep | Guard clauses, filter before looping, extract the inner operation |
| Wrapped arguments aligned under `(` | One-level continuation indent |
| Analyzer warnings ignored for months | Fix or set severity deliberately; `TreatWarningsAsErrors` |
