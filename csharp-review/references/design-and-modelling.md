# Design, Modelling & Architecture Review

This is the pass most reviews skip and the one that matters most over a codebase's life. It steps
back from "is this line correct" to "is this the right *shape*". Use it to produce
`Simplification`, `Domain Modelling`, `Architecture`, and `Maintainability` findings — and to
write the honest Design & Architecture Assessment.

The lens here is deliberately opinionated, grounded in two ideas:
1. **Deep modules** (John Ousterhout, *A Philosophy of Software Design*) — the cognitive-load side.
2. **Make illegal states unrepresentable** — the domain-modelling side.

## Contents

- [Cognitive load: deep vs shallow modules](#cognitive-load-deep-vs-shallow-modules)
- [Complexity symptoms to name](#complexity-symptoms-to-name)
- [C# cognitive smells](#c-cognitive-smells)
- [Domain modelling: make illegal states unrepresentable](#domain-modelling-make-illegal-states-unrepresentable)
- [Architecture: coupling, layering, dependencies](#architecture-coupling-layering-dependencies)
- [Maintainability & scalability](#maintainability--scalability)
- [Delivering the honest critique](#delivering-the-honest-critique)

---

## Cognitive load: deep vs shallow modules

The core measure of a good module is the ratio of the **functionality it provides** to the
**complexity of its interface**. A *deep* module hides a lot behind a simple interface; a
*shallow* one exposes nearly as much complexity as it contains, so it earns little and just adds
a layer to understand.

```csharp
// ❌ Shallow: the wrapper adds an interface but hides nothing — pure pass-through
public class UserRepository(AppDbContext db)
{
    public IQueryable<User> Query() => db.Users;          // leaks EF straight through
    public void Add(User u) => db.Users.Add(u);           // one-liner over the real API
    public Task Save() => db.SaveChangesAsync();
}
// Callers now juggle BOTH the repository and EF semantics. Two things to learn, no leverage.

// ✅ Deep: a small interface that actually hides the data-access complexity behind a real
// capability, expressed in domain terms
public interface IUserDirectory
{
    Task<User?> FindActiveByEmailAsync(Email email, CancellationToken ct);
    Task RegisterAsync(User user, CancellationToken ct);
}
```

When you see a class whose methods are each a single forwarding call, ask whether the abstraction
earns its keep. The right answer is often "inline it" — fewer layers, less to hold in your head.
Conversely, praise modules that take a genuinely hard job and present a clean front.

**Information hiding is the goal; interfaces are the cost.** Every public member, parameter, and
configuration option is a thing every caller must understand. Smaller interface + more hidden =
deeper = better. A class with twenty public methods and eight constructor parameters is shouting
that its boundary is in the wrong place.

**Complexity is incremental.** No single shallow class or stray parameter sinks a codebase; it's
the accumulation. So it's legitimate — and useful — to flag a small design erosion as 🟢/🟡 with
the note that it's one of a pattern, and to point at the pattern in the Assessment.

---

## Complexity symptoms to name

Ousterhout names three symptoms. Use them as vocabulary in findings — they make a critique
concrete instead of "this feels messy":

- **Change amplification** — a simple conceptual change requires edits in many places. (e.g. a
  status value duplicated as strings across ten files; adding a status touches all ten.) Name
  *where* the next change will fan out.
- **Cognitive load** — how much a developer must know to work here. Long parameter lists,
  non-obvious dependencies, and implicit ordering all raise it. Name *what* they're forced to
  hold in their head.
- **Unknown unknowns** — the worst: it's not obvious what you must change to make a change
  safely, or what knowledge is required. Hidden coupling and implicit invariants cause this. Name
  the trap a future maintainer will fall into.

A finding that says "this creates change amplification: adding a payment method means editing the
switch in `PaymentService`, the enum, the factory, and the validator" is actionable. "This is
hard to maintain" is not.

---

## C# cognitive smells

Concrete shapes that raise cognitive load in C#:

### Boolean / flag parameters

A `bool` at a call site is meaningless without checking the signature, and it usually means the
method does two things.

```csharp
// ❌ What does `true` mean here?
SendReport(user, true, false);

// ✅ Intent at the call site — split methods, or a named option
SendReportImmediately(user);
// or an explicit enum:  SendReport(user, ReportDelivery.Immediate, Attachments.None);
```

### Primitive obsession / stringly-typed code

Passing `string`/`int`/`Guid` everywhere loses meaning and invites mix-ups (passing a customer id
where an order id was expected — both `Guid`, compiler can't help). See the next section — this is
where modelling fixes a cognitive problem.

### Long parameter lists & "control" arguments

More than ~3–4 parameters, or parameters whose only job is to switch internal behavior, signal
the method is carrying too much. Group related parameters into a type; split behavior-switching
methods.

### Deep nesting / arrow code

Nesting beyond 2–3 levels hides the actual logic. Prefer guard clauses (early return), `switch`
expressions, and pattern matching to flatten.

```csharp
// ❌ Arrow code; the real rule is buried
if (order != null) { if (order.Items.Any()) { if (order.IsPaid) { Ship(order); } } }

// ✅ Guard clauses surface the happy path
if (order is null || order.Items.Count == 0 || !order.IsPaid) return;
Ship(order);
```

### Pass-through / wrapper layers

(See deep modules.) A layer that mostly forwards calls adds cognitive cost without hiding
anything. Flag repository/service/manager classes that are thin shells over EF or another service.

### Temporal coupling

Methods that must be called in a specific order, enforced only by convention (`Init()` then
`Process()` then `Flush()`), are a trap. Encode the ordering in the type (a builder that returns
the next valid state, or a constructor that does the init).

### Comments that compensate for unclear code

A comment explaining *what* the code does usually marks code that should be clearer. Comments that
explain *why* (rationale, invariants, gotchas) are valuable. Flag the former as a clarity issue,
keep the latter.

---

## Domain modelling: make illegal states unrepresentable

The strongest lever for both correctness and cognitive load is a type system that won't let you
express a nonsensical state. If the model can't represent the bug, you don't need a test, a
validation, or a code review to catch it.

### Anemic domain model

Entities that are bags of public setters with all behavior living in `*Service` classes scatter
each business rule across every service that touches the data, and let any caller put the object
into an invalid state.

```csharp
// ❌ Anemic: invariants live nowhere; anyone can set anything
public class Order { public OrderStatus Status { get; set; } public List<Item> Items { get; set; } }
// Five services each remember to check Status before mutating. One forgets → bug.

// ✅ The entity owns its invariants; illegal transitions are impossible from outside
public class Order
{
    private readonly List<Item> _items = [];
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public IReadOnlyList<Item> Items => _items;

    public void AddItem(Item item)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Order is locked.");
        _items.Add(item);
    }
    public void Submit()
    {
        if (_items.Count == 0) throw new InvalidOperationException("Cannot submit an empty order.");
        Status = OrderStatus.Submitted;
    }
}
```

### Primitive obsession → value objects

Wrap primitives that carry rules or meaning into small types. This makes the type system enforce
validity at construction and prevents argument mix-ups.

```csharp
// ❌ A string that may or may not be a valid email; any string fits the parameter
public void Register(string email) { … }

// ✅ A value object that can only exist if valid; the signature can't be misused
public readonly record struct Email
{
    public string Value { get; }
    public Email(string value)
    {
        if (!IsValid(value)) throw new ArgumentException("Invalid email", nameof(value));
        Value = value;
    }
}
public void Register(Email email) { … }   // "valid email" is now a compile-time guarantee
```

Strongly-typed IDs are the same idea and kill a whole class of bug:

```csharp
// ❌ Both Guid — the compiler can't stop you passing a CustomerId as an OrderId
public Task<Order> Get(Guid orderId, Guid customerId);

// ✅ Distinct types — the mix-up won't compile
public readonly record struct OrderId(Guid Value);
public readonly record struct CustomerId(Guid Value);
public Task<Order> Get(OrderId orderId, CustomerId customerId);
```

### Model states as a closed set, not flags

When an object's valid shape depends on its state, separate booleans let illegal combinations
exist (`IsPaid = true, PaidAt = null`). Model the states so the data for each is only present when
valid — a discriminated-union style with records + pattern matching, or distinct types per state.

```csharp
// ❌ Flags allow contradictory combinations
class Payment { bool IsPending; bool IsSettled; DateTime? SettledAt; string? FailureReason; }

// ✅ Each state carries exactly its own data; contradictions can't be expressed
abstract record Payment
{
    record Pending(decimal Amount) : Payment;
    record Settled(decimal Amount, DateTimeOffset At) : Payment;
    record Failed(decimal Amount, string Reason) : Payment;
}
```

### Other modelling smells

- **Ubiquitous language drift** — code names that don't match how the domain experts talk
  (`Doc1`, `tmpUser`, `data2`). Names are the cheapest documentation; flag where the model has
  stopped speaking the domain's language.
- **Leaky abstraction** — an interface that exposes its implementation (returning `IQueryable`,
  exposing EF entities to the UI layer, a "cache" whose interface reveals it's Redis). The
  abstraction should let callers ignore what's behind it.
- **God class / feature envy** — a class that knows about everything, or a method that operates
  mostly on another object's data, signals the responsibility lives in the wrong place.

---

## Architecture: coupling, layering, dependencies

- **Dependency direction.** Dependencies should point inward, toward the domain; the domain
  shouldn't reference infrastructure (EF, HTTP, the web framework). A domain entity that
  references `DbContext` or `HttpContext` is a layering violation worth a finding.
- **Coupling vs cohesion.** High coupling (many modules that must change together) plus low
  cohesion (a module doing unrelated things) is the expensive combination. Point at the specific
  edges.
- **Hidden global state.** `static` mutable state, service-locator pulls
  (`serviceProvider.GetService` deep in business logic), and ambient context create invisible
  dependencies that break testability and concurrency.
- **Abstraction at the wrong level / premature abstraction.** An interface with one
  implementation that exists "for flexibility" is speculative complexity. Equally, missing a
  seam at a true boundary (external API, clock, queue) makes the code untestable. Judge by whether
  the seam corresponds to a real axis of change.
- **Consistency.** A new piece that invents its own pattern when the codebase already has an
  established one raises cognitive load for everyone. Note the divergence and the existing
  convention it should follow.

---

## Maintainability & scalability

Tie these to a concrete future event, not a vibe:

- **Where will the next likely change land, and how far will it fan out?** (change amplification)
- **What breaks under 10× load or data?** Unbounded queries, per-request allocations on hot
  paths, sync-over-async, chatty I/O (N+1), in-process state that prevents horizontal scaling,
  locks that serialize throughput. Name the specific bottleneck and the scale at which it bites.
- **Statelessness.** In-memory caches/state in a service that's meant to scale out behave
  differently behind a load balancer — flag mutable instance/static state that assumes a single
  process.
- **Testability seams.** Code that hard-news `DateTime.Now`, `new HttpClient()`, or static
  dependencies can't be tested deterministically. The fix (inject `TimeProvider`, an interface,
  `IHttpClientFactory`) is also better design.
- **Test quality, not just coverage.** Tests that assert nothing, depend on wall-clock time
  (`Thread.Sleep`), share mutable state, or mock the thing under test give false confidence. Flag
  missing tests on branching logic and edge cases that the diff introduces.

---

## Delivering the honest critique

The user asked specifically for honest criticism when the approach is wrong. Calibrate it:

- **Lead with the failure mode, not the verdict.** "This won't scale" is dismissible; "every
  request holds a lock around the whole handler, so throughput is capped at one in-flight request
  regardless of cores" is not. Always name *which* invariant leaks, *which* change gets expensive,
  *where* it bottlenecks.
- **Separate "wrong shape" from "wrong details."** If the fundamental approach doesn't fit the
  feature, say so in the Verdict and Assessment — don't bury it as finding #23 among nits. A
  reviewer who only files line-notes on a doomed design has failed the author.
- **Propose the simpler alternative.** Criticism without a direction is just discouragement. Even
  a sketch ("model this as three states rather than five booleans") gives the author a path.
- **Respect what's working.** If the design is sound, say that clearly and briefly, and name what
  to preserve. Honesty cuts both ways — manufactured concerns are as misleading as missed ones.
- **Prefer removing complexity over adding it.** When two designs both work, the one with the
  smaller interface and fewer moving parts usually wins. Be suspicious of your own
  recommendations that add layers; make sure each one hides enough to justify itself.
