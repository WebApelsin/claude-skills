# C# / .NET Correctness & Mechanics

The concrete defect patterns to look for in C# code. Each is an `❌ / ✅` pair: the shape that's
wrong and the shape that's right. Match against the code under review; cite the file and line in
the finding.

## Contents

- [Async & await](#async--await)
- [Concurrency & thread-safety](#concurrency--thread-safety)
- [Nullable reference types](#nullable-reference-types)
- [Exceptions & error handling](#exceptions--error-handling)
- [Resource lifetime & disposal](#resource-lifetime--disposal)
- [EF Core & data access](#ef-core--data-access)
- [LINQ](#linq)
- [Correctness pitfalls](#correctness-pitfalls)
- [Allocation & performance](#allocation--performance)

---

## Async & await

### Sync-over-async (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`)

Blocking on a Task from a synchronous context risks deadlock (when a captured
`SynchronizationContext` can't re-enter) and starves the thread pool under load. This is one of
the most common 🔴 findings in ASP.NET Core code.

```csharp
// ❌ Blocks the request thread; deadlock risk, poor scalability
public ActionResult<Data> Get(int id)
{
    var data = _service.GetDataAsync(id).Result;
    return Ok(data);
}

// ✅ Async all the way down
public async Task<ActionResult<Data>> Get(int id, CancellationToken ct)
{
    var data = await _service.GetDataAsync(id, ct);
    return Ok(data);
}
```

### `async void`

`async void` methods can't be awaited and their exceptions escape to the synchronization context,
crashing the process. Only legitimate for event handlers.

```csharp
// ❌ Exceptions are unobservable; can crash the app
public async void Process() => await _service.RunAsync();

// ✅ Return Task so callers can await and observe failures
public async Task ProcessAsync() => await _service.RunAsync();
```

### Dropped `CancellationToken`

A token that isn't threaded through means long operations can't be cancelled — requests keep
running after the client disconnects, wasting resources and DB connections.

```csharp
// ❌ Accepts no token; the query can't be cancelled
public Task<List<User>> SearchAsync(string q) =>
    _db.Users.Where(u => u.Name.Contains(q)).ToListAsync();

// ✅ Accept and propagate the token end to end
public Task<List<User>> SearchAsync(string q, CancellationToken ct) =>
    _db.Users.Where(u => u.Name.Contains(q)).ToListAsync(ct);
```

### `ConfigureAwait(false)` in library code

In library / non-UI code that doesn't need the captured context, continuing on it adds overhead
and re-introduces deadlock risk for callers who block. Apply in reusable libraries; it's
unnecessary (and noise) in ASP.NET Core app code, which has no `SynchronizationContext`.

```csharp
// ✅ Library code: don't capture context
var body = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
```

### Fire-and-forget Tasks

An un-awaited Task swallows exceptions and may be killed mid-flight when the request scope ends.
If background work is intended, it needs a real home (hosted service, channel, outbox) — not a
discarded Task.

```csharp
// ❌ Exceptions vanish; work may be torn down; uses a disposed scope
_ = _service.SendEmailAsync(user);

// ✅ Hand off to infrastructure designed to outlive the request
await _backgroundQueue.EnqueueAsync(ct => _service.SendEmailAsync(user, ct));
```

### `Task.Run` to "make it async"

Wrapping synchronous or already-async work in `Task.Run` on the server just moves work to another
pool thread — it doesn't add concurrency and hurts throughput. Real async I/O (`...Async` APIs)
is what scales.

```csharp
// ❌ Offloads to the pool for no benefit on the server
await Task.Run(() => _db.SaveChanges());

// ✅ Use the genuinely async API
await _db.SaveChangesAsync(ct);
```

### `ValueTask` misuse

`ValueTask` must not be awaited more than once, nor have `.Result` read before completion.
Returning `ValueTask` is an optimization for hot paths that often complete synchronously; if the
result is awaited multiple times or stored, prefer `Task`.

```csharp
// ❌ Awaiting the same ValueTask twice is undefined behavior
var vt = GetAsync();
var a = await vt;
var b = await vt;   // bug

// ✅ Await once; materialize if you need the value twice
var a = await GetAsync();
```

### Async disposal of async resources

```csharp
// ❌ Disposing async resources synchronously can drop in-flight work
public void Dispose() => _stream.Dispose();

// ✅ IAsyncDisposable + await using
public async ValueTask DisposeAsync() => await _stream.DisposeAsync();
// caller:  await using var client = new DataClient();
```

---

## Concurrency & thread-safety

### Mutable static / shared state without synchronization

Shared mutable state touched by concurrent requests is a race waiting to happen. Look hard at
`static` fields, singletons, and captured closures.

```csharp
// ❌ Singleton/static dictionary mutated from concurrent requests
private static readonly Dictionary<int, User> _cache = new();
_cache[id] = user;            // torn writes, InvalidOperationException under load

// ✅ Concurrent collection, or a proper cache (IMemoryCache)
private static readonly ConcurrentDictionary<int, User> _cache = new();
_cache[id] = user;
```

### `lock` scope and what's locked

Locking on `this`, a `Type`, or a public object invites deadlocks from outside code. Lock on a
private dedicated object, hold it as briefly as possible, and never `await` inside a `lock`.

```csharp
// ❌ Public lock target; await inside the lock won't compile / would be wrong
lock (this) { await DoAsync(); }

// ✅ Private lock object; no await held; or SemaphoreSlim for async mutual exclusion
private readonly SemaphoreSlim _gate = new(1, 1);
await _gate.WaitAsync(ct);
try { await DoAsync(); } finally { _gate.Release(); }
```

### Check-then-act races

```csharp
// ❌ Two threads can both pass the check
if (!_cache.ContainsKey(key)) _cache.Add(key, Build(key));

// ✅ Atomic get-or-add
var value = _cache.GetOrAdd(key, Build);
```

Also watch for non-atomic counters (use `Interlocked.Increment`), and `Lazy<T>` /
`LazyThreadSafetyMode` when initialization must run once.

---

## Nullable reference types

If the project has `<Nullable>enable</Nullable>`, treat null-state warnings as design signals,
not noise.

### Null-forgiving operator (`!`) papering over real nullability

```csharp
// ❌ Silences the compiler and hides a real NRE
var name = user!.Name;

// ✅ Handle the null case, or fix the type so it can't be null
if (user is null) return NotFound();
var name = user.Name;
```

### Public API nullability that lies

A method annotated to return non-null that can actually return null pushes NREs onto callers.
Make the signature tell the truth (`T?`), or guarantee it.

### Reference-type defaults that bypass null checks

```csharp
// ❌ default(string) is null but the type says non-null
string s = default!;     // lie

// ✅ A real value, or an explicitly nullable type
string? s = null;
```

---

## Exceptions & error handling

### Rethrow that destroys the stack trace

```csharp
// ❌ Resets the stack trace to here — original location lost
catch (Exception ex) { Log(ex); throw ex; }

// ✅ Preserve the original trace
catch (Exception ex) { Log(ex); throw; }
// or wrap with context: throw new ProcessingException("…", ex);
```

### Swallowing exceptions

```csharp
// ❌ Silently eats failures — the worst kind of bug to diagnose
try { Risky(); } catch { }

// ✅ Catch what you can handle; let the rest propagate; log with context
try { Risky(); }
catch (IOException ex) { _logger.LogWarning(ex, "transient IO; retrying"); /* handle */ }
```

### Exceptions as control flow

Throwing/catching for expected outcomes (not-found, validation) is expensive and obscures intent.

```csharp
// ❌ Uses an exception for an ordinary "not found"
try { var u = await _db.Users.FirstAsync(x => x.Id == id); }
catch (InvalidOperationException) { return NotFound(); }

// ✅ Test for the condition directly
var u = await _db.Users.FirstOrDefaultAsync(x => x.Id == id, ct);
if (u is null) return NotFound();
```

### Catching `Exception` too broadly

Broad catches hide bugs (NREs, `OutOfMemory`, cancellation). Catch the specific types you can
meaningfully handle. Note: `OperationCanceledException` from a real cancellation is usually *not*
an error — don't log it as one.

---

## Resource lifetime & disposal

### Undisposed `IDisposable`

```csharp
// ❌ Leaks the handle/connection on every call
var conn = new SqlConnection(cs);
conn.Open();

// ✅ using ensures disposal even on exceptions
await using var conn = new SqlConnection(cs);
await conn.OpenAsync(ct);
```

### Disposing something you don't own

Disposing an injected dependency (e.g. an `HttpClient` from `IHttpClientFactory`, or a DI-managed
`DbContext`) breaks it for everyone else. Only dispose what you created.

### `IEnumerable` that holds resources

Returning a lazy iterator over an open reader/connection means the resource stays open until the
caller finishes enumerating — and leaks if they don't. Either materialize before returning, or
make the resource ownership explicit (`IAsyncEnumerable` with the connection scoped to it).

---

## EF Core & data access

### N+1 queries

```csharp
// ❌ One query per blog to load its posts
foreach (var blog in await _db.Blogs.ToListAsync(ct))
    foreach (var post in blog.Posts)   // lazy/again-to-DB per iteration
        Use(post);

// ✅ Project what you need in one round trip
var data = await _db.Blogs
    .Select(b => new { b.Url, Posts = b.Posts.Select(p => p.Title) })
    .ToListAsync(ct);
```

### Over-fetching (no projection)

Loading full entities when you need three columns wastes I/O, memory, and tracking cost.

```csharp
// ❌ Whole entity graph for a dropdown
var items = await _db.Products.ToListAsync(ct);

// ✅ Project to exactly what the caller needs
var items = await _db.Products
    .Select(p => new ProductOption(p.Id, p.Name))
    .ToListAsync(ct);
```

### Missing pagination — unbounded result sets

```csharp
// ❌ Could return millions of rows
var posts = await _db.Posts.Where(p => p.Active).ToListAsync(ct);

// ✅ Always bound list endpoints
var posts = await _db.Posts.Where(p => p.Active)
    .OrderBy(p => p.Id).Skip((page - 1) * size).Take(size)
    .ToListAsync(ct);
```

### Cartesian explosion from multiple `Include`

```csharp
// ❌ Multiple collection Includes multiply rows
var blogs = await _db.Blogs.Include(b => b.Posts).Include(b => b.Tags).ToListAsync(ct);

// ✅ Split the query
var blogs = await _db.Blogs.Include(b => b.Posts).Include(b => b.Tags)
    .AsSplitQuery().ToListAsync(ct);
```

### Missing `AsNoTracking` on read-only queries

Tracking entities you'll never modify costs memory and CPU. Read paths should opt out.

```csharp
var products = await _db.Products.AsNoTracking().ToListAsync(ct);
```

### Non-sargable predicates (functions on columns)

Wrapping a column in a function prevents index use → full scan.

```csharp
// ❌ Function on the column; index can't be used
.Where(p => p.Title.ToLower() == name)
.Where(p => p.Title.EndsWith(suffix))

// ✅ Case-insensitive collation or a stored normalized column; prefix match is sargable
.Where(p => p.Title.StartsWith(prefix))
```

### Sync DB calls on an async stack

```csharp
// ❌ Blocks a thread on I/O
var list = _db.Products.ToList();
_db.SaveChanges();

// ✅ Async I/O
var list = await _db.Products.ToListAsync(ct);
await _db.SaveChangesAsync(ct);
```

### Other data smells to flag

- Client-side evaluation of what should be a DB filter (see LINQ below).
- `DbContext` captured by a singleton or shared across threads (it's not thread-safe — one per
  unit of work).
- Raw SQL built by string concatenation (→ security module: injection).
- Long-lived `DbContext` accumulating tracked entities (memory growth).
- Missing transactions around multi-step writes that must be atomic.

---

## LINQ

### Materializing then filtering in memory

```csharp
// ❌ Pulls the whole table into memory, then filters client-side
var r = _db.Posts.Where(p => p.Active).ToList().Where(Expensive);

// ✅ Push everything translatable to the database first
var r = await _db.Posts.Where(p => p.Active && p.Score > 0).ToListAsync(ct);
```

### `Count() > 0` instead of `Any()`

`Any()` stops at the first match; `Count()` enumerates / counts everything.

```csharp
if (await _db.Users.AnyAsync(u => u.Active, ct)) { … }
```

### Multiple enumeration of `IEnumerable`

Enumerating a lazy sequence twice re-runs the work (or re-queries). Materialize once if you need
it more than once.

```csharp
// ❌ items enumerated twice — possibly two DB hits
if (items.Any()) foreach (var i in items) Use(i);

// ✅ Materialize, then reuse
var list = items.ToList();
if (list.Count > 0) foreach (var i in list) Use(i);
```

### Side effects inside `Select`

Deferred execution makes the timing of side effects unpredictable. Keep `Select` pure; put
effects in an explicit loop.

### `First`/`Single` without a fallback

`First()` throws on empty; `Single()` throws on empty *or* more than one. Use the `OrDefault`
variants and handle the null unless the throw is genuinely the intended contract.

---

## Correctness pitfalls

### `DateTime.Now` and `DateTime` for instants

`DateTime.Now` is server-local and ambiguous across time zones / DST; `DateTime` doesn't carry
offset. For instants in time, use `DateTimeOffset` and UTC, and inject `TimeProvider` so time is
testable.

```csharp
// ❌ Local, ambiguous, untestable
var stamp = DateTime.Now;

// ✅ UTC offset via an injectable clock
var stamp = _timeProvider.GetUtcNow();
```

### `float`/`double` for money

Binary floating point can't represent decimal fractions exactly → rounding errors in currency.

```csharp
// ❌ double total = price * qty;   // accumulates error
decimal total = price * qty;        // ✅ exact decimal arithmetic
```

### Culture-sensitive string ops by accident

Default string comparison/casing is culture-sensitive, which causes locale-dependent bugs (the
classic Turkish-I). Be explicit: `StringComparison.Ordinal[IgnoreCase]` for identifiers/keys,
culture-aware only for user-facing display/sorting.

```csharp
// ❌ Culture-dependent equality for an identifier
if (code.ToUpper() == "ADMIN") …

// ✅ Ordinal, allocation-free
if (string.Equals(code, "ADMIN", StringComparison.OrdinalIgnoreCase)) …
```

Same for `int.Parse` / `ToString` on machine-readable values: pass `CultureInfo.InvariantCulture`.

### Broken equality / `GetHashCode` contracts

If a type overrides `Equals`, it must override `GetHashCode` consistently, or it misbehaves in
dictionaries/sets. Mutable fields in a hash key are a bug (the object gets "lost" in the set).
`record` gives you correct value equality for free — prefer it over hand-rolled equality.

### `== null` vs `is null`

`is null` can't be hijacked by an overloaded `==` operator and reads clearly. Prefer `is null` /
`is not null` for reference checks.

---

## Allocation & performance

Most code doesn't need micro-optimization — flag these mainly on hot paths (per-request, loops,
high-throughput handlers), and tag them honestly with severity/confidence.

### String concatenation in loops

```csharp
// ❌ Allocates a new string each iteration — O(n²)
var s = ""; foreach (var x in items) s += x + ",";

// ✅ StringBuilder, or string.Join
var s = string.Join(",", items);
```

### Hidden boxing

Boxing value types (e.g. via non-generic collections, `object` params, or struct-to-interface in
a loop) allocates. Prefer generics and `Span<T>` on hot paths.

### LINQ allocations on hot paths

LINQ allocates iterators/closures per call. In tight loops or high-RPS handlers, a plain `for`
or `foreach` can matter. Don't pre-optimize cold paths — measure intent.

### Large synchronous reads / buffering whole payloads

Reading an entire stream/file into memory doesn't scale with payload size. Stream it
(`await foreach`, `IAsyncEnumerable`, piping) when the size is unbounded or large.
