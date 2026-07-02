# Architecture & Design Checklist

These findings are judgment calls more than other categories — explain the *consequence* of the issue (what becomes hard to change, test, or reason about) rather than citing a principle name as if it were self-evidently true.

## SOLID, applied practically

- **Single Responsibility**: a class doing unrelated things (e.g., a controller that also contains business logic, validation, *and* direct SQL) — flag when it makes the class hard to test or change for one reason without risking the others, not just because it's "doing more than one thing" in the abstract.
- **Open/Closed**: long `if`/`else` or `switch` chains on a type code that get a new branch every time a new variant is added — a sign a polymorphic/strategy pattern would let new variants be added without touching existing code. Only worth raising if there's evidence this actually grows often (multiple existing branches, or a comment/history suggesting frequent additions).
- **Liskov substitution**: a subtype that throws `NotImplementedException` for inherited members, or violates the base type's contract (e.g., a `Stack` subclass of `List` that breaks `List`'s ordering guarantees) — these cause surprising bugs at call sites that trust the base type.
- **Interface segregation**: a fat interface forcing implementers to stub out methods they don't support — usually shows up as multiple `throw new NotSupportedException()` implementations.
- **Dependency inversion**: high-level code directly depending on a concrete low-level implementation (e.g., business logic `new`-ing up a `SqlConnection` directly) instead of depending on an abstraction injected via DI — makes testing and swapping implementations hard.

## Dependency injection

- Services resolved via a static service locator (`ServiceLocator.Get<T>()`, a static `IServiceProvider`) instead of constructor injection — hides dependencies and complicates testing.
- Captive dependencies: a singleton-lifetime service holding a reference to a scoped or transient service (e.g., a singleton caching a `DbContext`) — the scoped service effectively becomes a singleton too, often incorrectly, and can cause subtle bugs or thread-safety issues since scoped services aren't designed to be shared across requests.
- Missing or incorrect service lifetime registration (`AddSingleton` vs `AddScoped` vs `AddTransient`) relative to how the service is actually used.

## State and coupling

- Mutable static fields used for state that should be per-request or per-instance — a common source of subtle bugs in web apps where static state leaks across requests/users.
- Tight coupling between layers that should be independent (e.g., a domain entity referencing EF Core types directly, or a domain model with a hard reference to a specific UI framework type).
- Circular dependencies between modules/projects/namespaces.

## Domain modeling

- Anemic domain models: entities that are just property bags with all the logic living in separate "service" classes — not always wrong (it's a legitimate architectural choice), but worth noting if the codebase otherwise reads as intending rich domain models and isn't following through.
- Primitive obsession: using raw `string`/`int`/`Guid` for concepts that have their own validation/behavior (email addresses, money, IDs that shouldn't be interchangeable with other IDs of the same primitive type) where a small wrapper type or `record struct` would prevent mixing up an `OrderId` and a `CustomerId` that are both secretly `Guid`.
- DTOs and domain/persistence models conflated — the same class serves as the EF Core entity, the API contract, and the business object, making it hard to evolve any one of those without breaking the others.

## Error handling architecture

- Catching exceptions too broadly (`catch (Exception)`) and swallowing them silently, losing the original error context, or catching and rethrowing without `throw;` (which destroys the original stack trace — use `throw;` not `throw ex;`).
- Using exceptions for expected, common control flow (e.g., throwing to signal "not found" on a lookup that fails routinely) where a result type or nullable return would be cheaper and clearer — this is a real but secondary concern, usually Low/Medium.
- Inconsistent error response shapes across an API (some endpoints return a structured error object, others return a bare string or stack trace) — a Medium concern since it affects every consumer of the API.

## Severity guidance for this category

Most architecture findings are Medium — they're about long-term maintainability rather than something breaking today. Escalate to High only when the current structure is already causing observable problems (a bug traceable to a captive dependency, a circular dependency that's currently breaking builds). Don't manufacture an architecture finding just to have one in every review.