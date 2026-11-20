# Abstraction Levels

A method, class, or module should operate at a single, coherent level of abstraction. Mixing levels forces the reader to context-switch between strategic intent and implementation mechanics.

## What Is an Abstraction Level?

```
Domain / Business Level      "Issue the invoice"
       ↓
Application Level            "Load the draft, issue it, save, publish InvoiceIssued"
       ↓
Infrastructure Level         "POST to /invoices, SaveChangesAsync, publish to Service Bus"
       ↓
Mechanism Level              "Serialize to UTF-8 JSON, compute HMAC header"
```

Each level is a valid place to write code. The mistake is writing two levels in the same method.

---

## The Mixing Violation

```csharp
// Mixed levels in one method — hard to read
public async Task<IResult> SubmitOrder(CreateOrderRequest request, CancellationToken ct)
{
    // Application level
    var order = Order.Create(request.CustomerId, request.Lines);

    // Infrastructure / mechanism level mixed in
    using var client = new HttpClient();                       // also a resource bug
    client.BaseAddress = new Uri("https://erp.example.com");
    client.DefaultRequestHeaders.Add("X-Api-Key", _config["Erp:ApiKey"]);
    var json = JsonSerializer.Serialize(new { order.Id, Lines = order.Lines.Select(l => new { l.Sku, l.Quantity }) });
    var response = await client.PostAsync("/api/orders", new StringContent(json, Encoding.UTF8, "application/json"), ct);
    if (response.StatusCode != HttpStatusCode.Created) return Results.Problem("ERP rejected order");

    // Back to application level
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct);
    return Results.Created($"/orders/{order.Id}", null);
}
```

## The Consistent Level Approach

```csharp
// Application level — endpoint handler
public async Task<IResult> SubmitOrder(CreateOrderRequest request, CancellationToken ct)
{
    var order = Order.Create(request.CustomerId, request.Lines);
    await _erp.RegisterOrderAsync(order, ct);
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct);
    return Results.Created($"/orders/{order.Id}", null);
}

// Infrastructure level — typed HttpClient
public sealed class ErpClient(HttpClient http)
{
    public async Task RegisterOrderAsync(Order order, CancellationToken ct)
    {
        using var response = await http.PostAsJsonAsync("/api/orders", ErpOrder.From(order), ct);
        response.EnsureSuccessStatusCode();
    }
}
```

Base address and API key live in `AddHttpClient<ErpClient>(...)` registration; JSON shape lives in `ErpOrder`. Each piece can be read independently.

---

## The Step-Down Rule

Order members so each is followed by the members it calls, one level down. The file reads top-down from policy to mechanism: public members first, private helpers below in call order.

```csharp
public async Task CheckoutAsync(Cart cart, CancellationToken ct)
{
    EnsureCheckoutable(cart);
    var order = await CreateOrderAsync(cart, ct);
    await CapturePaymentAsync(order, ct);
    await NotifyCustomerAsync(order, ct);
}

private static void EnsureCheckoutable(Cart cart) { ... }
private async Task<Order> CreateOrderAsync(Cart cart, CancellationToken ct) { ... }
private async Task CapturePaymentAsync(Order order, CancellationToken ct) { ... }
private async Task NotifyCustomerAsync(Order order, CancellationToken ct) { ... }
```

Follow the repository's member ordering convention (e.g. StyleCop SA1201/SA1202) if it has one; step-down then applies within the private section.

---

## Identifying Level Violations

### Named calls mixed with raw computation

```csharp
// Violation
public decimal ApplyDiscount(Order order)
{
    var eligible = SelectEligibleLines(order.Lines);
    var subtotal = eligible.Sum(l => l.UnitPrice * l.Quantity);   // inline mechanism
    return subtotal * (1 - 0.15m);                                // magic number
}

// Fix
public decimal ApplyDiscount(Order order) => DiscountedTotal(SelectEligibleLines(order.Lines));
```

### UI or endpoint contains business rules or data access

Razor pages, Blazor components, controllers, and minimal-API lambdas are presentation. Validation rules, EF queries, and HTTP calls to other systems belong in the application or infrastructure layer.

```csharp
// Violation — Blazor component with business rule and data access
@code {
    async Task Save()
    {
        if (!_model.Email.Contains('@')) return;
        await using var db = await DbFactory.CreateDbContextAsync();
        db.Customers.Update(_model);
        await db.SaveChangesAsync();
    }
}

// Prefer — component delegates
@code {
    async Task Save() => _result = await CustomerProfile.UpdateAsync(_model, _cts.Token);
}
```

### Data layer contains presentation

```csharp
// Violation — repository formats for display
public async Task<string> GetDisplayNameAsync(CustomerId id, CancellationToken ct) { ... $"{c.FirstName} {c.LastName}".Trim() ... }

// Prefer — repository returns the model; display lives on the model or in presentation
public string DisplayName => $"{FirstName} {LastName}".Trim();
```

---

## Abstraction Levels in a .NET Application

| Layer | Level | Allowed to contain |
|---|---|---|
| Razor / Blazor / Controller / Endpoint | Presentation | Binding, routing, auth attributes, DTO mapping, status codes |
| Handler / Application service | Application | Use-case orchestration, transactions, authorization decisions |
| Domain entities / value objects | Domain | Business rules, invariants |
| EF Core, HttpClient, file/bus | Infrastructure | SQL, HTTP, serialization, I/O |
| Extensions / low-level utilities | Mechanism | Algorithms, conversions |

---

## Common Violations and Fixes

| Violation | Location | Fix |
|---|---|---|
| URL or header construction in a handler | Application | Typed `HttpClient` + registration |
| `string.Format` for display in a repository | Infrastructure | Property on model / view model |
| Business rule in Razor/Blazor | Presentation | Domain method or application service |
| `JsonSerializer` calls in a controller | Presentation | Let the framework bind, or a typed client |
| Retry loops in a handler | Application | Resilience pipeline (`AddStandardResilienceHandler`, Polly) |
| `DbContext` in a domain entity | Domain | Pass data in; keep entities persistence-ignorant |

---

## Summary

| Question | Good sign | Bad sign |
|---|---|---|
| Can you name the method's level in one word? | Yes | "It does a bit of both" |
| Do all calls feel like the same height? | Yes | Some high-level, some raw operations |
| Does the file read top-down? | Yes | Members in arbitrary order |
| Does the UI contain only presentation code? | Yes | EF queries or validation rules |
| Does the data layer contain only persistence? | Yes | Display formatting |
