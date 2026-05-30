# Standard 15: Test Infrastructure as Architecture

> Tests are the only feedback loop an agent has. When tests run against a different database than production, agents receive false confirmation that their code works — locking silently ignored, type handling differs, constraints missing. The agent sees green and moves on. The bug ships to production. A substitute database still catches ordinary logic errors, but it produces false confidence about exactly the engine-dependent behavior — locking, native types, constraints — that it cannot reproduce, and it does so at machine speed.

> **Standards** (must follow):
> - Test against the same database engine as production (not an in-memory substitute)
> - Predictable test file structure that mirrors module structure
> - No test file over 500 lines — split by domain when it grows
>
> **Guidelines** (recommended):
> - Disposable containers for database instances per test session
> - Suppression with traceability (`# nocheck` requires issue reference)

Tests are ground truth (Standard 6). But test infrastructure itself needs architectural discipline. Without it, test files become the most disorganized part of the codebase — the place where "just add it here" accumulates fastest.

### Test Against Your Production Engine, Not a Substitute

A common pattern: tests run against an in-memory database for speed, while production runs a different engine entirely. This creates a parallel universe where your tests pass but your production code is wrong.

Things a substitute database hides:
- **Row-level locking** (`SELECT ... FOR UPDATE`) silently ignored
- **UUID handling** differs (strings vs native types)
- **Partial indexes** behave differently or don't exist
- **Array/JSON types** have different semantics
- **Concurrent access** patterns are completely different

The fix: use disposable containers. Spin up a real instance of your production database engine for each test session — a throwaway container, never the live production database itself. The container is destroyed after tests complete. Setup adds seconds to the test run — negligible compared to the bugs it catches.

Every test runs against the same database engine your production code uses. Every lock actually locks. Every type behaves as it would in production. Every constraint fires exactly as expected.

**This rule is about the database engine, not collaborators.** Mocking a *service* at its interface seam — like the notification-service mock in the "Tests as Ground Truth" example — is legitimate, and it's exactly what interface-first dependency injection is for: you test one module in isolation against the contracts of its neighbors. What you never substitute is the database engine underneath the module you're actually testing. Mock across module boundaries; run real against your own storage.

### Predictable Test Structure

Standard 3 says every module looks the same. The same applies to test directories:

```
modules/<name>/tests/
├── __init__.py
├── conftest.py                     # shared fixtures for this module
├── test_service_<domain_a>.py      # one domain per file
├── test_service_<domain_b>.py
└── test_security_<name>.py         # exploit/security tests
```

Rules:
- **No test file over 500 lines.** If it grows beyond that, split by domain.
- **Shared fixtures in `conftest.py`.** Your test framework discovers them automatically — no explicit imports needed.
- **One domain per file.** All payment tests in `test_service_payments`. All auth tests in `test_service_auth`. The filename tells you exactly what's inside.
- **No behavioral changes during splits.** Splitting is a pure refactor — move code between files. Same test count before and after. Same pass/fail results.
