# Style & Conventions Checklist

Style findings are almost always Low or Nit — they matter for consistency and readability, but shouldn't dominate a review. Calibrate to the codebase's existing conventions where visible (a codebase already using `var` everywhere shouldn't get flagged for using `var`); these are defaults for when there's no established convention to match.

## Naming

- Public members, types, namespaces: `PascalCase`.
- Private fields: `_camelCase` (leading underscore) is the modern .NET convention; plain `camelCase` without underscore is older-style but still common — flag inconsistency *within* a file/class more than the choice itself.
- Local variables and parameters: `camelCase`.
- Interfaces: `IPascalCase` prefix.
- Type parameters: `TPascalCase` (e.g., `TEntity`, `TKey`), or single capital letters in simple generic contexts.
- Constants: `PascalCase` in modern .NET (not `SCREAMING_CASE`, which is a C/Java carryover).
- Async methods: suffix with `Async` (`GetUserAsync`) — flag missing suffix as a Nit, but missing suffix combined with the method *not* actually being awaited properly elsewhere is worth escalating since it signals confusion about sync vs async.

## Nullable reference types

- If `<Nullable>enable</Nullable>` is set: flag suppressions (`!` null-forgiving operator) that aren't obviously safe, and missing null checks on parameters that are referenced without a prior guard.
- If nullable is *not* enabled project-wide: don't flag every potential null dereference as if it were a guaranteed bug (it's a possible bug, same as in any pre-nullable C# code) — instead, suggest enabling nullable reference types if the codebase would benefit, as a single Low/Medium suggestion rather than repeating "possible null reference" on every line.

## Modern C# idioms (calibrate to target framework — see SKILL.md Step 1)

- File-scoped namespaces (`namespace Foo.Bar;`) vs block-scoped — available C# 10+, purely stylistic.
- Primary constructors (C# 12+) for simple data-holding classes/records where the boilerplate constructor adds nothing.
- Pattern matching (`is`, `switch` expressions, property patterns) where a chain of `if`/`else` or `as` + null-check is doing the same thing less clearly.
- `record`/`record struct` for immutable data-carrying types instead of a hand-rolled class with manual `Equals`/`GetHashCode`/`ToString`.
- Collection expressions (`[1, 2, 3]`, C# 12+) vs `new List<int> { 1, 2, 3 }` — purely stylistic, only relevant if targeting C# 12+.
- Target-typed `new()` where the type is already obvious from the declaration.
- Raw string literals / interpolated string improvements where they'd clean up escaping-heavy strings (regex patterns, JSON templates, SQL).

## Expression-bodied members vs block bodies

A one-line property getter or simple method is a reasonable candidate for `=>` syntax; multi-statement logic forced into an expression body (via tuple/discard tricks) hurts readability more than it helps. This is a taste call — only flag if it's clearly fighting the language.

## `var` vs explicit type

Both are accepted styles in modern C#. The only thing worth flagging is when `var` obscures a non-obvious type (e.g., `var result = SomeMethod();` where the return type isn't guessable from context or the name) — suggest an explicit type there for readability, not as a blanket rule.

## Formatting and structure

- Inconsistent brace style, indentation, or blank-line conventions *within the same file* (cross-file/project-wide inconsistency is usually a `.editorconfig`/formatter problem to fix once, not something to call out repeatedly per-PR).
- Magic numbers/strings that would be clearer as named constants — Low severity unless the value is security/business-logic-critical (e.g., a hardcoded role name or status code), in which case it edges toward Medium because of maintainability risk, not style.
- Methods or classes that have clearly grown too large to read in one sitting — note this here as a style/readability flag; if it's actually a separation-of-concerns problem, also cross-reference `architecture.md`.

## Severity guidance for this category

Default to Nit. Use Low only when the inconsistency would plausibly confuse a future reader or reviewer (mixed naming conventions for the same kind of symbol within one file, a misleading `var`). Never go above Low for pure style.