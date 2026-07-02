# Async & Threading Checklist

This is the category where "looks fine" and "is fine" diverge the most in C# — bugs here are often invisible until load, timing, or scale changes. Read carefully rather than pattern-matching on keywords.

## Deadlocks and blocking

- `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` on a `Task` from a context that has a `SynchronizationContext` (classic ASP.NET, WPF, WinForms) — see `performance.md` for the mechanism. In ASP.NET Core there's no `SynchronizationContext` by default, which makes this less universally deadly than in classic ASP.NET, but it's still a thread-pool-starvation risk under load and a portability hazard if the code is ever reused in a context that does have one. Treat as High regardless of which ASP.NET flavor, since it's rarely intentional.
- Mixing blocking and async code in the same call chain inconsistently — a sign the author wasn't deliberate about it.

## `async void`

- `async void` methods other than top-level event handlers (the one legitimate use case, since event handler delegates can't return `Task`). Exceptions thrown inside an `async void` method can't be caught by the caller (there's no `Task` to observe/await) and will instead crash the process via the `SynchronizationContext` or thread pool's unhandled exception path. Fix: `async Task` everywhere except actual event handlers.

## Cancellation

- Async methods that accept work that could be long-running (I/O, loops over external calls) but don't take a `CancellationToken` parameter, or take one but don't actually pass it through to the awaited calls inside (an unused/ignored token is almost as bad as missing one — it gives a false sense of cancellability).
- `CancellationToken.None` passed explicitly where the ambient token (from the HTTP request, the hosted service's stopping token, etc.) was available and should have been threaded through instead.

## Race conditions and shared state

- Multiple threads/requests reading and writing shared mutable state (static fields, singleton-scoped fields, captured closures shared across `Task.Run` calls) without synchronization. Look especially at singleton-lifetime services with mutable instance fields — every request shares the same instance.
- Check-then-act patterns without atomicity (`if (!_dict.ContainsKey(k)) _dict[k] = v;` from multiple threads) — use `ConcurrentDictionary.GetOrAdd` or equivalent atomic operations instead.
- Non-thread-safe collections (`List<T>`, `Dictionary<K,V>`) accessed concurrently without a lock — fix with `lock`, a `Concurrent*` collection, or by confirming (and commenting) that access is actually single-threaded.
- `lock` taken on a non-private, non-readonly, or boxed object (locking on a string literal, a publicly accessible field, or a boxed value type) — these can be inadvertently shared or re-created, defeating the lock. Fix: a `private readonly object` dedicated to the lock, or a `Lock` object (.NET 9+).
- Double-checked locking implemented without the right combination of `volatile`/memory barriers — easy to get subtly wrong; prefer `Lazy<T>` or `LazyInitializer` over a hand-rolled version.

## Task composition

- `Task.Run` used to wrap already-async I/O-bound work (no benefit — it just adds a thread-pool hop) vs. genuinely CPU-bound work (where it's the right tool to avoid blocking the calling thread). Check which case applies before flagging.
- `await Task.WhenAll(...)` vs sequential `await`s in a loop where the operations are independent — sequential awaiting of independent work is a missed concurrency opportunity, not a correctness bug, so this is usually Medium not High.
- Fire-and-forget tasks (`_ = SomeAsync();` or an unawaited call) where exceptions will be lost and the task's lifetime isn't tracked — acceptable only if deliberate and exceptions are handled inside the fired task itself (e.g., wrapped in its own try/catch with logging).
- `ConfigureAwait(false)` usage: matters most in library code that shouldn't assume a `SynchronizationContext`; largely a non-issue in modern ASP.NET Core app code (no context to avoid). Don't flag its absence in application-level ASP.NET Core code as a bug — at most a Nit/Low style preference for library projects.

## Async streams and enumerables

- A method returning `Task<List<T>>` that buffers an entire large/unbounded result set in memory before returning, where `IAsyncEnumerable<T>` would let the consumer process items as they arrive.
- `IAsyncEnumerable<T>` consumed without `WithCancellation` when a token is available and cancellation matters.

## Severity guidance for this category

Race conditions and deadlock-prone blocking calls on a request path are High by default — they're often intermittent and hard to reproduce, which makes them costly later even if they haven't caused a visible incident yet. `Task.WhenAll` opportunities and `ConfigureAwait` nitpicks are Medium/Low — real, but not urgent.as