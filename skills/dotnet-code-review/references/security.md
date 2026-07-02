# Security Checklist

What to look for, why it matters, and the typical fix. Not exhaustive — use these as anchors, and flag anything else that smells like a real vulnerability.

## Injection

- **SQL injection**: raw string concatenation or interpolation building SQL (`$"SELECT * FROM Users WHERE Id = {id}"`, `"... WHERE name = '" + name + "'"`). Even inside `FromSqlRaw` or `ExecuteSqlRaw` calls, not just `SqlCommand`. Fix: parameterized queries (`FromSqlInterpolated`, `SqlParameter`, or EF LINQ which parameterizes automatically).
- **Command injection**: user input flowing into `Process.Start` arguments, especially via shell (`cmd.exe /c`). Fix: avoid shell invocation; if unavoidable, use `ArgumentList` (not a concatenated string) and validate/allowlist input.
- **LDAP/XPath/NoSQL injection**: same pattern, different sink. Look for string-built queries against any of these.
- **Log injection**: unsanitized user input written directly into logs can forge log entries or break log parsing. Worth a Low/Medium flag, not Critical, unless logs feed an automated system that trusts them.

## Secrets and credentials

- Hardcoded connection strings, API keys, passwords, or tokens in source — including in test files and config files committed to the repo (`appsettings.json` with real values rather than `appsettings.Development.json` placeholders).
- Secrets passed via command-line arguments (visible in process lists) rather than environment variables or a secrets manager.
- Fix: `IConfiguration` backed by environment variables, user secrets (dev), or a vault/secrets manager (prod); never literal values in source.

## Cryptography and randomness

- Weak hash algorithms for security-sensitive purposes: `MD5`, `SHA1` for password hashing (fine for non-security checksums like cache keys). Fix: `Microsoft.AspNetCore.Identity.PasswordHasher<T>` or `Argon2`/`bcrypt` via a vetted library.
- `System.Random` used for tokens, password reset codes, session IDs, or anything security-sensitive — it's predictable. Fix: `RandomNumberGenerator.GetBytes` / `System.Security.Cryptography`.
- Custom-rolled crypto (hand-written XOR "encryption", homemade key derivation). Flag as Critical regardless of apparent correctness — this is a "don't roll your own" category.
- `X509Certificate` / `ServicePointManager.ServerCertificateValidationCallback` set to always return `true`, or `HttpClientHandler.ServerCertificateCustomValidationCallback` bypassing validation. Common in "temporary" debug code that ships to prod.

## Deserialization

- `BinaryFormatter`, `NetDataContractSerializer`, or `ObjectStateFormatter` used on any data that could originate externally — these are unsafe by design and deprecated/removed in modern .NET for this reason. Fix: `System.Text.Json` or a safe serializer with type allowlisting.
- `JsonSerializerSettings.TypeNameHandling` set to anything other than `None` in Newtonsoft.Json on externally-supplied JSON — enables type-confusion deserialization attacks.

## Web/API-specific (ASP.NET Core)

- Missing `[Authorize]` on controllers/actions that should require auth — check whether `[AllowAnonymous]` is intentional or a leftover from scaffolding/testing.
- Authorization checked only in UI/client code, not enforced server-side.
- CORS configured with `AllowAnyOrigin()` combined with `AllowCredentials()` (this combination is actually rejected by browsers/the framework in recent versions, but `AllowAnyOrigin` alone plus a sensitive API is still worth flagging).
- Missing or weak input validation on model binding — relying solely on client-side validation, or `[FromBody]` models with no `[Required]`/data annotations or FluentValidation on fields used in business logic.
- Reflected user input rendered in Razor without encoding (`@Html.Raw(userInput)`) — XSS. Razor encodes by default; `Html.Raw` and `[AllowHtml]`-style bypasses are the things to catch.
- SSRF: server-side code that fetches a URL supplied by the user (webhook callback registration, "fetch this image" features) without validating against internal/private IP ranges.
- JWT validation: check `TokenValidationParameters` isn't disabling `ValidateIssuer`, `ValidateAudience`, or `ValidateLifetime`, and that signing keys aren't hardcoded test keys left in from scaffolding.

## Path and file handling

- Path traversal: user-supplied filenames/paths concatenated into a file system path without normalization (`Path.Combine(basePath, userInput)` is *not* sufficient on its own — `userInput` can still contain `..`; need to verify the resolved full path stays under `basePath`).
- Unrestricted file upload: no validation of file type/size/content, files written to a web-accessible directory, or files executed/interpreted based on extension trust.

## Severity guidance for this category

Most genuine findings here are Critical or High — that's the nature of security bugs. Reserve Medium for things like "log injection" or defense-in-depth gaps (e.g., missing rate limiting on a non-sensitive endpoint) where exploitation requires additional preconditions.