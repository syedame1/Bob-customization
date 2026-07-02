# Python PR Review Skill

## When to use this skill
Load this skill whenever reviewing Python files in a pull request.

## Review checklist

### Style and conventions
- PEP 8 compliance — line length (88 chars, Black standard), spacing, imports ordered (stdlib → third-party → local)
- Naming — snake_case for functions and variables, PascalCase for classes, UPPER_SNAKE for constants
- No unused imports, no wildcard imports (`from x import *`)
- Docstrings on all public functions and classes

### Security red flags (flag these immediately)
- Hardcoded secrets, API keys, passwords, or tokens in any form
- Raw SQL string concatenation — always flag, always suggest parameterized queries
- `eval()`, `exec()`, `pickle.loads()` on untrusted input
- `subprocess` calls with `shell=True`
- Missing input validation on any data entering from outside the system
- `DEBUG = True` in any config that could reach production

### Code quality
- Functions longer than 40 lines — flag for decomposition
- Cyclomatic complexity — more than 3 nested levels is a smell
- Exception handling — bare `except:` or `except Exception:` without logging is never acceptable
- Magic numbers — unexplained numeric literals should be named constants

### Testing expectations
- New functions need corresponding tests
- Tests should cover the happy path and at least one failure case
- No `time.sleep()` in tests — mock it

## Output format
Structure your review as:

**Summary** — two sentences: what this PR does, overall quality signal

**Issues** — numbered list, each with:
- Severity: 🔴 Critical / 🟡 Warning / 🔵 Suggestion
- File and line number
- What the problem is
- What the fix looks like (show the corrected code where possible)

**Positives** — one or two things done well (skip if nothing genuine)

Never use generic phrases like "consider refactoring" without showing what the refactor looks like.