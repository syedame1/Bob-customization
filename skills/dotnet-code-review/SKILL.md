---
name: dotnet-code-review
description: Use this skill whenever the user wants C#/.NET code reviewed — pull requests, diffs, whole files, or whole projects. Trigger on requests like "review this PR", "review this controller", "is this C# code production-ready", "audit this service for security issues", "check this .NET code for problems", "code review this", or any time C#/.NET source is pasted or uploaded alongside a request to check, review, audit, or critique it. Covers security, performance, style/conventions, architecture, async/threading correctness, and test coverage. Make sure to consult this skill even if the user only asks about one of those angles (e.g. "is this thread-safe?" or "any security issues here?") rather than a full review, since the same systematic process and severity rubric apply.
---

# .NET Code Review

A systematic process for reviewing C#/.NET code across six dimensions: security, performance, style/conventions, architecture, async/threading, and testing. The goal is a review that reads like it came from a sharp, fair senior engineer — specific, prioritized, and actionable — not a linter dump.

## Why a process helps

Unstructured code review tends to drift toward whatever catches the eye first, usually style nits, while missing the issues that actually matter (a race condition, a SQL injection, a deadlock-prone `.Result` call). Working through each dimension deliberately, then prioritizing by severity, produces a review that's both thorough and readable.

## Step 1: Figure out what you're looking at

Before reviewing, establish:

1. **Diff or whole file(s)?** A git diff (lines prefixed with `+`/`-`, or `diff --git` headers) means the author already wrote and presumably ran the surrounding code. Focus findings on the changed lines and their immediate blast radius — don't relitigate pre-existing code outside the diff. If something dangerous lives just outside the diff but interacts with it (e.g., a query the new code calls into), it's fair to mention briefly as context, clearly marked as pre-existing.

2. **Project context.** If a `.csproj`, `.sln`, or `global.json` is available, check the target framework (`<TargetFramework>`) and whether nullable reference types are enabled (`<Nullable>enable</Nullable>`). This calibrates your suggestions — don't recommend C# 12 primary constructors or collection expressions against a `netstandard2.0` library, and don't flag missing null checks as harshly in a nullable-disabled codebase (it's a different, lower-severity finding there: "consider enabling nullable reference types" rather than "this will NRE").

3. **What kind of code is it?** A controller, a background worker, a domain entity, a test file, and infrastructure/DI setup code all have different things that matter most. An ASP.NET controller leans security and input validation; a background `IHostedService` leans threading and lifetime/cancellation; an EF Core repository leans the N+1/tracking issues in `references/performance.md`.

If none of this is available (just a pasted snippet with no project context), proceed anyway and note any assumptions you made.

## Step 2: Work the six dimensions

Read each reference file below for the specific checklist of what to look for in that dimension. Don't load all six up front if the code clearly doesn't touch a dimension (a pure data model has no async code to check) — use judgment about which are relevant, but default to checking all six for anything beyond a trivial snippet, since issues often hide across dimension boundaries (a performance problem and a security problem can be the same line).

- `references/security.md` — injection, secrets, crypto, deserialization, authz/authn, SSRF, and other vulnerability classes
- `references/performance.md` — allocations, EF Core query patterns, sync-over-async, disposal, hot-path costs
- `references/style-conventions.md` — naming, modern C# idioms, nullable reference types, formatting
- `references/architecture.md` — SOLID, coupling, layering, DI usage, domain modeling
- `references/async-threading.md` — deadlocks, async void, cancellation, race conditions, lock usage
- `references/testing.md` — coverage gaps, test quality, flakiness, mocking patterns

Each finding you raise should be something you'd actually say out loud in a review — if it's so minor you'd hesitate to mention it in person, it's a Nit (see below) or it's not worth including at all. Signal over noise matters more than completeness; a review with 40 nits and 1 buried security issue has failed even if every nit was technically correct.

## Step 3: Assign severity

Use this rubric consistently so the summary counts mean something:

| Severity | Meaning | Examples |
|---|---|---|
| **Critical** | Will cause a security breach, data loss/corruption, or production outage | SQL injection, hardcoded credentials, missing authz on a sensitive endpoint, a deadlock-guaranteed `.Result` call on the request path |
| **High** | A real bug or significant risk, even if not catastrophic | Race condition under load, N+1 query that'll crawl at scale, async void swallowing exceptions, missing cancellation token propagation |
| **Medium** | A design or correctness concern that should be fixed but isn't urgent | SOLID violation making future changes risky, missing edge-case handling, suboptimal but not broken locking |
| **Low** | Worth fixing, low risk | Style/convention deviations, missing tests for non-critical paths, minor inefficiency |
| **Nit** | Optional, purely cosmetic or preference | Formatting, expression-bodied member vs block body, naming taste |

## Step 4: Write the review

Always produce both parts, in this order:

### 1. Summary

- **Verdict**: one of `Approve`, `Approve with comments`, `Changes requested`, or `Block — critical issues`. Pick the verdict that matches the highest severity found: any Critical → Block; any High → Changes requested; only Medium/Low/Nit → Approve with comments; nothing found → Approve.
- **Counts**: `Critical: n · High: n · Medium: n · Low: n · Nit: n`
- **2-4 sentences** naming the most important theme(s) — not a list of every finding, just what the author should internalize first.

### 2. Detailed findings

Grouped by severity, most severe first. For each finding:

- **Location** — file and line number(s), or a quoted code fragment under 15 words if line numbers aren't available
- **Category** — Security / Performance / Style / Architecture / Async-Threading / Testing
- **Issue** — what's wrong and *why it matters* (the mechanism, not just "this is bad practice")
- **Fix** — a concrete suggested change. Include a short code snippet when it clarifies the fix faster than prose would.

Skip a category entirely in the detailed section if it had zero findings — don't pad the review with "No issues found in Testing" for every clean dimension; that's noise. It's fine to mention strong points briefly in the summary if something was done particularly well (e.g., good cancellation token propagation throughout) — reviews that only ever criticize read as adversarial rather than helpful.

## Tone

Write like a respected senior engineer who wants the PR to land well, not like a compliance scanner. Explain mechanisms ("this allocates a new `List<T>` on every call because `.Where().Select()` aren't materialized until enumerated, and you enumerate twice here") rather than asserting rules ("avoid multiple enumeration"). Where there's a legitimate tradeoff rather than a clear bug (e.g., `IEnumerable` vs `List` return types, or a deliberate `Task.Run` for CPU-bound work), frame it as a question or note the tradeoff rather than a flat directive.