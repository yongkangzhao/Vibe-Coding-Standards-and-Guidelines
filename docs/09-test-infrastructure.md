# Standard 14: Test Infrastructure as Architecture

> Tests are the only feedback loop an agent has. When tests run against SQLite in-memory while production runs PostgreSQL, agents receive false confirmation that their code works — `SELECT FOR UPDATE` silently ignored, UUID handling differs, array types missing. The agent sees green and moves on. The bug ships to production. Test infrastructure that doesn't match production is worse than no tests at all, because it produces false confidence at machine speed.

> **Standards** (must follow):
> - Test against your production database (not a substitute like SQLite for PostgreSQL)
> - Predictable test file structure that mirrors module structure
> - No test file over 500 lines — split by domain when it grows
>
> **Guidelines** (recommended):
> - Testcontainers or equivalent for disposable database instances per test session
> - SQLite as an optional `--quick` fast path for local iteration (not CI default)
> - Specific hook implementations (pre-write, pre-commit, pre-push, post-edit)
> - Suppression with traceability (`# nocheck` requires issue reference)

Tests are ground truth (Standard 4). But test infrastructure itself needs architectural discipline. Without it, test files become the most disorganized part of the codebase — the place where "just add it here" accumulates fastest.

### Test Your Production Database, Not a Substitute

A common pattern: tests run against SQLite in-memory for speed, while production runs PostgreSQL (or MySQL, or another database). This creates a parallel universe where your tests pass but your production code is wrong.

Specific things SQLite hides:
- **Row-level locking** (`SELECT ... FOR UPDATE`) is silently ignored
- **UUID handling** differs (strings vs native types)
- **Partial indexes** behave differently
- **Array types** don't exist
- **Concurrent access** patterns are completely different

The fix: use disposable containers. Libraries like `testcontainers` spin up a real PostgreSQL instance in Docker for each test session. The container is destroyed after tests complete. Setup adds ~2 seconds to the test run — negligible compared to the bugs it catches.

```python
@pytest.fixture(scope="session")
def db_engine():
    from testcontainers.postgres import PostgresContainer
    with PostgresContainer("postgres:16") as pg:
        engine = create_engine(pg.get_connection_url())
        Base.metadata.create_all(engine)
        yield engine
```

Every test runs against the same database your production code uses. Every `SELECT FOR UPDATE` actually locks. Every UUID is a native UUID. Every constraint fires exactly as it would in production.

Keep SQLite as an optional fast path (`--quick` flag) for rapid local iteration. But CI and the default test run should hit real PostgreSQL.

### Predictable Test Structure

Standard 1 says every module looks the same. The same applies to test directories:

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
- **Shared fixtures in `conftest.py`.** pytest discovers them automatically — no explicit imports needed.
- **One domain per file.** All payment tests in `test_service_payments.py`. All auth tests in `test_service_auth.py`. The filename tells you exactly what's inside.
- **No behavioral changes during splits.** Splitting is a pure refactor — move code between files. Same test count before and after. Same pass/fail results.

### The Hook Taxonomy

Standard 5 says "guardrails beat guidelines." Here's the concrete hook architecture that makes this real:

**Pre-write hooks** (block before the file is modified):
- Cross-module import guard: rejects any import from a sibling module
- ORM-in-common guard: rejects database models in the shared kernel

**Pre-commit hooks** (block before the commit is created):
- ORM without migration: staged `orm.py` without a corresponding Alembic migration
- Pattern violations: `len()` for counting DB rows, `echo=True` in engine config, subquery misuse

**Pre-push hooks** (block before code reaches remote):
- Architecture compliance: exceptions defined in interface files, missing interface for new service
- Code quality: list queries without `order_by`, write operations without foreign key validation

**Post-edit hooks** (run after a file is modified):
- Auto-run the affected module's test suite when `service.py` is edited
- Inject test failures back into the agent's context for self-correction

Each hook outputs a structured message: what rule was violated, which file, which line, what the fix is. The agent receives this as feedback and self-corrects. No human intervention needed for mechanical violations.

**Suppression with traceability:**

```python
from sibling_module import Something  # nocheck: arch-guard  # remove when #42 is fixed
```

Every suppression requires an issue reference. `grep '# nocheck'` shows all exceptions. Suppressions without issue numbers are rejected in review.
