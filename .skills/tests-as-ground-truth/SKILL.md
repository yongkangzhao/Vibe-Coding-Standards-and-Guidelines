---
name: tests-as-ground-truth
description: Treats tests as the machine-readable definition of correct behavior and insists they run against production-equivalent infrastructure, not substitutes. Use this when writing or reviewing tests, choosing a test database (real engine vs in-memory SQLite), setting up test fixtures or CI, deciding what to mock, rendering components under test (Testing Library vs shallow render), wiring MSW or a network layer, or organizing/splitting test files — for both backend services and frontend components, even if the user doesn't say "tests as ground truth."
---

# Tests as Ground Truth

> Tests are not regression insurance bolted on after the fact — they are the executable specification that defines what "correct" means in machine-readable form. This matters most when an agent writes the code: an agent starts every session with no institutional memory and no way to "just know" the behavior is right. Tests are its only feedback loop. Without them, an agent ships code that compiles and returns something plausible but unverified; with them, failures are immediate, specific, and self-correcting.

## When to use this
- Writing tests for a new module, service, endpoint, component, or hook (backend or frontend).
- Choosing the test database — someone reaches for in-memory SQLite while production runs Postgres.
- Setting up test fixtures, `conftest.py`, CI test runners, or a disposable database container.
- Deciding what to mock: which seams are legitimate to stub vs. the unit you must run for real.
- Frontend: choosing `render` vs shallow render, wiring MSW or a network mock, testing a component's real DOM output.
- A test file is creeping past ~500 lines, or tests are scattered without a predictable home.
- Reviewing a PR where green tests give you a vague unease that nothing real was exercised.

## The standards

**Tests define correct behavior; write them before implementation.** They are the agent's spec and feedback loop, not afterthought QA — an agent reads failing tests and writes code until they pass, no design work left. Red-green-refactor mechanics live in `superpowers:test-driven-development`; this skill is about what the tests must assert and what they run against.

**Assert behavior, not implementation.** A test should describe an observable outcome a caller can see, so it survives refactors and actually pins down the contract.
```
test "assigning to a missing user is rejected":
    task = create(title="x")
    expect( assign(task.id, "ghost-id") ) raises UserNotFound
```

**Run against your production engine, not a substitute.** A substitute database still catches ordinary logic errors — but it gives false confidence about exactly the engine-dependent behavior it cannot reproduce: row-level locking (`SELECT ... FOR UPDATE` silently ignored), native UUID/array/JSON types, partial indexes, constraint firing, concurrency. The agent sees green at machine speed and ships the bug. Use a disposable, throwaway container of the real engine per test session — destroyed after, never the live production database.

**Mock only across module/interface seams — never the unit under test.** Stubbing a *neighbor service* at its interface (the notification service from a task test) is exactly what interface-first DI is for: you test one module against its neighbors' contracts. But never substitute the storage engine underneath the module you are testing, and never mock the component or function you are actually asserting on.
```
# legitimate: neighbor across a module boundary
mock_notifier = stub(NotificationServiceInterface)
# illegitimate: the engine under the unit you're testing
db = InMemorySubstitute()   # <-- forbidden; spin a real engine container
```

**Render/exercise the real thing.** Frontend: render the actual component into a real DOM (Testing Library), drive a real-ish network via MSW, and assert what the user sees. Shallow rendering or mocking the unit under test produces the same false green as an in-memory DB.

**Predictable test structure mirrors modules.** Tests live beside the module in `tests/`, shared fixtures in `conftest.py` (or a setup file), one domain per file named so the filename tells you what's inside. An agent that understands one module's tests understands all of them.
```
modules/<name>/tests/
  conftest.py                 # shared fixtures, auto-discovered
  test_service_<domain_a>.py  # one domain per file
  test_service_<domain_b>.py
  test_security_<name>.py     # exploit / security tests
```

**No test file over ~500 lines.** When it grows, split by domain — a pure refactor: same test count, same pass/fail before and after, no behavioral change. Oversized test files are where "just add it here" accumulates fastest and where agents lose the thread.

## Checklist
- [ ] Tests exist and were written before (or alongside) the implementation, and assert observable behavior.
- [ ] Backend tests run against the **same engine** as production via a disposable container — no in-memory SQLite standing in for Postgres.
- [ ] The storage engine and the unit under test are real; only cross-module/interface seams are mocked.
- [ ] Frontend tests render the real component (Testing Library) and use MSW for network, not a stubbed unit.
- [ ] Tests live in the module's `tests/` dir, one domain per file, named `test_service_<domain>` (or framework analog).
- [ ] No single test file exceeds ~500 lines; splits preserved test count and pass/fail exactly.

## What breaks without this
- **Plausible-but-wrong code ships.** With no tests, the agent has no feedback loop; it returns something that compiles and looks right, and correctness is unknowable without running the whole system by hand.
- **False green from a substitute DB.** Tests pass against in-memory SQLite while `SELECT ... FOR UPDATE` is silently ignored, UUIDs serialize differently, and constraints never fire — the bug surfaces only in production.
- **Refactors break tests for no reason.** Tests coupled to implementation (or to a shallow-rendered tree) fail on harmless restructuring, so they get deleted or muted and stop being ground truth.
- **Test dir becomes the messiest part of the repo.** Without a mirrored structure and a size cap, a 2,000-line `test_everything` file becomes the place agents can neither navigate nor extend reliably.

## Stack-specific examples
- **Backend:** see `references/backend.md` — pytest, disposable Postgres container, real-engine fixture, `test_service_<domain>` layout.
- **Frontend:** see `references/frontend.md` — vitest + Testing Library rendering a real component, MSW network, file naming/structure.

## Related
- `superpowers:test-driven-development` for the red-green-refactor *mechanics* — this skill adds the "tests are the spec" framing and the "test against real infrastructure" rule; do not duplicate its workflow.
- `interface-first-development` — tests come third in its Spec → Interface → Tests → Implementation loop; the interfaces it defines are exactly the seams you mock here.
- Full rationale: `docs/04-structural-guidelines.md` (Standard 6) and `docs/09-test-infrastructure.md` (Standard 15) in this repo.
