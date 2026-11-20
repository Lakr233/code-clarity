# Blazor Components

A component takes its parameters and the state it owns, and turns them into markup. Clarity in Blazor means applying the same rules as the rest of this skill to that shape: every piece of state has one owner, data flows down through parameters, changes flow up through callbacks, and the markup shows decisions without making them.

---

## State ownership

### Parameters are inputs, not storage

The parent sets a `[Parameter]` and overwrites it on every re-render. If the child writes to it, the child's change is silently lost (analyzer BL0005).

```csharp
// Avoid — the child mutates its input; the next parent render resets it
[Parameter] public bool IsExpanded { get; set; }
private void Toggle() => IsExpanded = !IsExpanded;

// Prefer — the parent owns the value; the child asks for a change
[Parameter] public bool IsExpanded { get; set; }
[Parameter] public EventCallback<bool> IsExpandedChanged { get; set; }
private Task Toggle() => IsExpandedChanged.InvokeAsync(!IsExpanded);
```

The `Value` / `ValueChanged` pair enables `@bind-IsExpanded` in the parent. When the parent doesn't care about the value, the child owns it in a private field and exposes no parameter for it. Pick one owner. Don't keep a parameter *and* a private copy that you sync in `OnParametersSet`.

### Derive, don't sync

```csharp
// Avoid — a second copy of the truth, refreshed by hand
private List<Invoice> _overdue = [];
protected override void OnParametersSet() => _overdue = Invoices.Where(i => i.IsOverdue(Today)).ToList();

// Prefer — computed when it is read
private IEnumerable<Invoice> Overdue => Invoices.Where(i => i.IsOverdue(Today));
```

Store a derived value only when computing it is measurably expensive, and then recompute it in exactly one place.

### Communicate up with `EventCallback`

Use `EventCallback<T>` for child-to-parent communication. Don't use `Action<T>`, and don't use a shared service. `EventCallback` re-renders the parent afterwards and supports async handlers. Name callbacks after what happened (`OnInvoiceSelected`), not after what the parent will do with it (`LoadInvoiceDetails`).

### Sharing state beyond parent and child

Go down this list and stop at the first option that works:

1. **Parameters**, even through two levels. Passing a value explicitly is easier to follow than receiving it implicitly.
2. **Cascading value**, for something that really is ambient, such as the current user, theme, or an `EditContext`. Don't use it as a hidden global for feature data.
3. **Scoped state service** with a change event, when components in different parts of the tree have to react to the same data. Subscribe in `OnInitialized`, unsubscribe in `Dispose`, and re-render with `InvokeAsync(StateHasChanged)`.

---

## Markup shows, code decides

Keep the conditions, LINQ and formatting in C# under a name. The markup then reads as a description of the UI.

```razor
@* Avoid — business rules inside the template *@
@if (Invoice.Status == InvoiceStatus.Issued && Invoice.DueDate < DateOnly.FromDateTime(DateTime.Today) && !Invoice.IsDisputed)
{
    <span class="badge overdue">@((DateOnly.FromDateTime(DateTime.Today).DayNumber - Invoice.DueDate.DayNumber)) days</span>
}

@* Prefer — the template refers to named concepts *@
@if (IsOverdue)
{
    <span class="badge overdue">@DaysOverdue days</span>
}
```

The rule itself (when is an invoice overdue?) belongs on the domain type, not in the component ([abstraction-levels.md](abstraction-levels.md)). The component only exposes what to show.

- Once the `@code` block is more than a screen of code, move it to a code-behind file (`InvoiceCard.razor.cs`, `partial class`). Use one convention per repository. Don't mix inline and code-behind for components of the same kind.
- Keep `@if` / `@foreach` nesting shallow, just like method bodies ([formatting-consistency.md](formatting-consistency.md#indentation-shows-structure)). When a branch grows, give it its own component.
- Styling goes in CSS isolation (`InvoiceCard.razor.css`), not in `style=""` attributes or string-concatenated class names. Use `::deep` only to style a child component you don't control.

---

## Component size and shape

Split a component when the part has its own state, its own reason to change, or is reused. Don't split just to make the file shorter. A wrapper component that only passes every parameter through is a pass-through service in markup form.

Use slots instead of flags:

```razor
@* Avoid — a parameter for each variation *@
<Card ShowHeader="true" HeaderText="Invoice" ShowFooterButtons="true" FooterButtonText="Pay" />

@* Prefer — the caller provides the content *@
<Card>
    <Header>Invoice @Invoice.Number</Header>
    <Footer><button @onclick="Pay">Pay</button></Footer>
</Card>
```

Two or more booleans that change how a component renders are a sign that it should take a `RenderFragment`, or that it is really two components.

---

## Lifecycle and rendering

| Need | Use |
|---|---|
| Load data once | `OnInitializedAsync` |
| Reload when a parameter changes | `OnParametersSetAsync`, guarded by "did the relevant parameter actually change" |
| Touch the DOM / JS | `OnAfterRenderAsync(firstRender)` |
| Update from a timer, service event, or other thread | `await InvokeAsync(StateHasChanged)` |

Blazor already re-renders after event handlers, `EventCallback`s, and lifecycle methods. An explicit `StateHasChanged()` in those places adds nothing; remove it. If it seems to be needed there, the state likely has the wrong owner.

Handle the loading state explicitly. Model it as a type ([parse-dont-validate.md](parse-dont-validate.md)), not as `bool _isLoading` + `string? _error` + `List<T>? _items`, where every combination needs its own check:

```csharp
private abstract record Load;
private sealed record Loading : Load;
private sealed record Loaded(IReadOnlyList<Invoice> Invoices) : Load;
private sealed record Failed(string Message) : Load;

private Load _state = new Loading();
```

Then the markup is a single `switch` over the cases, with no impossible combinations to handle.

Add `@key` to `@foreach` items that can be reordered, inserted, or removed. Without it, Blazor keeps component state with the wrong item.

---

## Async and disposal in components

- When a load can be outdated by a newer one (a search box, a parameter change), keep a `CancellationTokenSource`. Cancel and replace it on each new load, and dispose it in `Dispose` ([async-and-resources.md](async-and-resources.md)).
- Implement `IDisposable` / `IAsyncDisposable` for every subscription, timer, or `IJSObjectReference` the component creates. If it subscribes, it unsubscribes.
- Don't use `async void` handlers. Use `@onclick="SaveAsync"` with `Task SaveAsync()`.

---

## JS interop

Use HTML and CSS before JavaScript. When you do need JS, put it in **one** collocated module (`InvoiceChart.razor.js`). Load it once in `OnAfterRenderAsync(firstRender)`, keep it in a single `IJSObjectReference` field, and dispose it in `DisposeAsync` (catch `JSDisconnectedException` there). Calls to `IJSRuntime.InvokeVoidAsync("window.someGlobal…")` spread across the codebase are unowned global state.

---

## Forms

Use `EditForm` with one model type for the form. Validation rules belong on the model or its value objects, and are written once. Don't repeat them in markup conditions or event handlers. Convert the form model to a domain type once, on submit ([parse-dont-validate.md](parse-dont-validate.md)). Domain types shouldn't need public setters just so they can be bound.

---

## Testing

Test what the user sees and does with bUnit: render the component with parameters, interact with it, and assert on the markup or the callback that fired. Don't assert on private fields.

```csharp
var cut = Render<InvoiceStatusBadge>(parameters => parameters.Add(badge => badge.Status, InvoiceStatus.Paid));

Assert.Equal("Paid", cut.Find(".badge").TextContent);
```

Logic that is easy to test without rendering, such as an overdue rule or a filter, already belongs outside the component. If you need bUnit to test a calculation, move the calculation.

---

## Review questions

| Question | If yes |
|---|---|
| Does a component write to its own `[Parameter]`? | Add `EventCallback`, or make it private state |
| Is a parameter copied to a field in `OnParametersSet`? | Derive it, or choose one owner |
| Does the markup contain business conditions or LINQ? | Move to named members or the domain type |
| Are there two or more boolean parameters that change the layout? | `RenderFragment` slots, or split the component |
| Is there `StateHasChanged()` inside an event handler or lifecycle method? | Remove it |
| Are loading, error, and data separate nullable fields? | Use a closed load-state hierarchy |
| Is there a subscription, timer, or JS reference without `Dispose`? | Implement disposal |
| Is a cascading value carrying feature data? | Use parameters or a scoped state service |
| Does a `@foreach` over a changing list lack `@key`? | Add `@key` |
