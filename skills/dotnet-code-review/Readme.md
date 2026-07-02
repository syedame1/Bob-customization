# dotnet-code-review

A Bob skill for systematic, senior-engineer-quality code reviews of C#/.NET code. Works on pull request diffs, whole files, and whole projects. Produces a verdict, severity-bucketed finding counts, and detailed findings with locations, explanations, and concrete fixes — not a linter dump.

## What it covers

Six review dimensions, each backed by a dedicated reference checklist:

| Dimension | What it catches |
|---|---|
| **Security** | SQL injection, hardcoded secrets, weak crypto, unsafe deserialization, broken authz/authn, SSRF, path traversal, XSS |
| **Performance** | EF Core N+1 queries, sync-over-async thread starvation, `HttpClient` misuse, missing `AsNoTracking()`, allocation-heavy hot paths, resource disposal |
| **Style & conventions** | Naming (PascalCase, `_camelCase`, `Async` suffix), nullable reference types, modern C# idioms (records, pattern matching, collection expressions, primary constructors), `var` clarity |
| **Architecture** | SOLID violations, captive dependencies, service lifetime mismatches, static state, primitive obsession, DTO/domain/persistence model conflation, error handling patterns |
| **Async & threading** | Deadlock-prone `.Result`/`.Wait()`, `async void`, missing/ignored `CancellationToken`, race conditions on shared state, unsafe `lock` targets, fire-and-forget without error handling |
| **Testing** | Coverage gaps on new logic, weak assertions, test-implementation coupling, `Thread.Sleep`-based flakiness, shared mutable state between tests, undescriptive test names |

## Output format

Every review contains two sections:

**1. Summary** — verdict, severity counts, and 2–4 sentences on the most important themes.

```
Verdict: Changes requested
Critical: 0 · High: 2 · Medium: 1 · Low: 2 · Nit: 3

The async/threading issues here are the priority: two places use `.GetAwaiter().GetResult()`
on the request path which will deadlock under ASP.NET's default thread-pool behaviour and
stall requests under load. The EF Core repository also lacks `.AsNoTracking()` on read-only
queries, which adds unnecessary change-tracking overhead. Style is generally clean.
```

**2. Detailed findings** — grouped by severity (Critical → High → Medium → Low → Nit), each with location, category, mechanism, and a concrete fix (with a short code snippet where it speeds up understanding).

## Severity rubric

| Severity | Meaning |
|---|---|
| **Critical** | Security breach, data loss/corruption, or guaranteed production outage |
| **High** | Real bug or significant risk — will cause problems under load or at scale |
| **Medium** | Design/correctness concern; should be fixed but isn't immediately breaking |
| **Low** | Worth fixing; low risk |
| **Nit** | Optional, purely cosmetic |

## File layout

```
dotnet-code-review/
├── SKILL.md                        # Orchestrator: 4-step review process and output format
└── references/
    ├── security.md                 # Injection, secrets, crypto, deserialization, web/API issues
    ├── performance.md              # EF Core patterns, sync-over-async, allocations, disposal
    ├── style-conventions.md        # Naming, nullable, modern C# idioms, formatting
    ├── architecture.md             # SOLID, DI, state, domain modeling, error handling
    ├── async-threading.md          # Deadlocks, async void, cancellation, race conditions
    └── testing.md                  # Coverage, test quality, flakiness, naming
```

## How to use it

Paste or upload your code and ask for a review. The skill works with:

- **Git diffs / PR diffs** — focuses findings on changed lines; pre-existing issues outside the diff are noted but marked as such
- **Whole files** — reviews the full file across all six dimensions
- **Whole projects** — can be pointed at multiple files or a project directory

You can also ask a targeted question and still get the structured output:

> "Is this thread-safe?"
> "Are there any security issues in this controller?"
> "Is this ready for production?"

The skill applies the same systematic process and severity rubric regardless of how broadly or narrowly the question is framed.

## Target framework awareness

The skill reads `.csproj` / `global.json` when available and calibrates suggestions to the target framework — it won't suggest C# 12 primary constructors against a `netstandard2.0` project, and it adjusts null-check severity based on whether `<Nullable>enable</Nullable>` is set.

## Tone

Reviews are written in the voice of a senior engineer who wants the PR to land, not a compliance scanner. Findings explain the *mechanism* ("this creates a new `HttpClient` per request, which exhausts sockets under load because TCP sockets aren't immediately freed after `Dispose`") rather than citing rules. Tradeoffs are framed as tradeoffs, not violations.
