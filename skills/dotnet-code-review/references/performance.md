# Performance Checklist

## Sync-over-async (the most common .NET deadlock source)

- `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` called on a `Task` from synchronous code that's itself reachable from an async context (e.g., an ASP.NET request, or anything that might run under a `SynchronizationContext`). This is the classic deadlock pattern: the calling thread blocks waiting for the task, but the task's continuation needs that same thread/context to resume. Fix: make the caller `async` and propagate `await` up the call chain ("async all the way").
- `Task.Run(() => asyncMethod()).Result` as a workaround for the above — it dodges the immediate deadlock but burns a thread pool thread and still isn't a real fix; the right answer is almost always to make the caller async.

## EF Core / database query patterns

- **N+1 queries**: a loop that accesses a navigation property without it being eagerly loaded (`.Include()`) — each iteration triggers a separate round-trip. Fix: `.Include()`/`.ThenInclude()` for the access pattern needed, or `.AsSplitQuery()` for large fan-outs.
- **Tracking overhead on read-only queries**: EF Core tracks entities by default for change detection; read-only queries (display, API responses with no subsequent `SaveChanges`) pay this cost for nothing. Fix: `.AsNoTracking()`.
- **Pulling more than needed**: `.ToList()` before filtering/paging client-side instead of pushing the filter into the query (`.Where()` before materializing), or selecting whole entities when only a few columns are used (`.Select(x => new Dto { ... })` projects only what's needed and can avoid loading large columns entirely).
- **Missing pagination** on endpoints that return collections that can grow unbounded.
- Multiple `SaveChanges()` calls in a loop instead of batching changes and calling once.

## Allocations and hot paths

- String concatenation in a loop (`result += item` or `result = result + item`) — each iteration allocates a new string. Fix: `StringBuilder` for anything beyond a handful of iterations.
- Repeated `.Where().Select()` (or similar) LINQ chains enumerated more than once without materializing — each enumeration re-runs the whole pipeline, and if the source is `IEnumerable` rather than already a `List`/array, this can mean repeated expensive work (e.g., repeated DB hits if it's `IQueryable`, repeated computation if it's a generator). Fix: materialize once with `.ToList()`/`.ToArray()` if it'll be enumerated more than once.
- Boxing of value types: a `struct` passed where `object` or a non-generic interface is expected (e.g., putting `int`s into an `ArrayList`, or calling a method taking `object` with a value type argument) — easy to miss for older-style APIs.
- `Regex` constructed fresh inside a hot path/loop instead of cached or compiled. Fix: static readonly compiled `Regex`, or the `[GeneratedRegex]` source generator on .NET 7+.
- Large object allocations in tight loops where a pooled or `Span<T>`/`Memory<T>`-based alternative would avoid the allocation entirely — flag as a suggestion (Low/Medium) rather than a hard requirement unless profiling evidence is mentioned.

## Resource disposal

- `IDisposable` objects (streams, `HttpClient` short-lived instances, `DbContext` outside DI lifetime management, `SemaphoreSlim`, etc.) created without a `using` statement/declaration or try/finally disposal.
- `HttpClient` instantiated per-request with `new HttpClient()` rather than via `IHttpClientFactory` or a shared/static instance — can exhaust sockets under load (a known .NET footgun, distinct from the more familiar "always dispose" advice since `HttpClient` is designed to be reused).

## Caching and repeated work

- Expensive computation or external calls repeated on every request when the result is cacheable (config lookups, reference data, computed values that don't change per-request). Fix: `IMemoryCache`/`IDistributedCache` with a sensible TTL, or precomputation.
- Cache entries with no eviction policy/expiration in a long-running process — slow memory growth.

## Severity guidance for this category

N+1 queries, sync-over-async deadlock risks, and unbounded HttpClient creation are High (they degrade or break under real load). String concatenation in a loop or a missed `.AsNoTracking()` on a small read is usually Low/Medium unless it's clearly in a hot path. Don't invent precise numbers ("this is 40% slower") unless you have actual evidence — describe the mechanism and let the magnitude speak for itself.