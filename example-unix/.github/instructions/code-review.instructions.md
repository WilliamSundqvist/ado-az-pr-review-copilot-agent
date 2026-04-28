---
applyTo: '**'
description: Team coding standards consulted by the PR Review agent. Auto-applied workspace-wide.
---

# Code Review Standards

> The scaffolding prompt seeds this file from a quick scan of your codebase. Edit freely. Delete what doesn't apply. Add what matters. The PR Review agent reads every file in `.github/instructions/` automatically — to add more standards, drop another `.instructions.md` file in this folder.

## General

- Prefer readable code over clever code. If a junior engineer can't follow it in 30 seconds, add a comment or refactor.
- Every change should be the smallest one that solves the problem. PRs that touch unrelated areas should be split.
- Tests are part of the change. A bug fix without a regression test is incomplete unless the test would be impractical (UI flakiness, hardware dependency, etc.) — in which case say so in the PR description.

## Naming

- Functions are verbs (`fetchUser`, `validateInput`).
- Booleans read as questions (`isReady`, `hasPermission`, `shouldRetry`).
- Avoid abbreviations except for well-known ones (`id`, `url`, `db`).

## Errors

- Never swallow errors silently. Either handle them meaningfully or propagate.
- Error messages should tell the reader what went wrong AND what to do about it.
- Don't `catch` more broadly than you need to. Catching `Exception` (Python) or `catch (e)` (TS/JS) without rethrowing is a yellow flag.

## Logging

- Log at the boundary of your system (request in, request out, external call). Don't log inside tight loops.
- Never log secrets, tokens, full request bodies that may contain PII, or full SQL with parameters.
- Use structured logging where the language supports it.

## Tests

- New code should have tests. Bug fixes should have a regression test.
- Tests should test behaviour, not implementation. If a refactor breaks tests but doesn't change behaviour, the tests were too coupled.
- Avoid `sleep()` in tests. Use proper async waits or fakes.

## Security

- Never trust input from outside the trust boundary. Validate at the edge.
- SQL: parameterise. String concatenation into queries is a blocker.
- Secrets in code, config files, or commits are a blocker. Use the secret store.
- New endpoints need authn/authz checks unless they are explicitly public.

## Performance

- Watch for N+1 queries in any loop that touches the database.
- Don't pre-optimise, but flag obviously expensive operations in hot paths (sorting in inner loops, allocations, etc.).

## Dependencies

- New runtime dependencies need a reason. Prefer the standard library.
- Pin versions in lockfiles. Check the lockfile diff for unexpected transitive bumps.

## Documentation

- Public APIs need docstrings/JSDoc.
- README should reflect reality. If a PR changes setup, update the README in the same PR.

---

<!-- The scaffolding prompt may add language- or framework-specific sections below based on what it detects. -->
