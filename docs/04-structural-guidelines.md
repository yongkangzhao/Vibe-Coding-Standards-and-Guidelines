# Standards 3–9: The Structural Guidelines

> Agents need predictable navigation. When every module has a different file layout, different naming conventions, and different patterns, agents waste tokens reasoning about where things are instead of doing useful work. Predictable structure means an agent that understands one module understands all of them — and unpredictable structure means every task starts with an exploration phase that produces inconsistent results.

> **Standards** (must follow):
> - Predictable, uniform module structure across the entire codebase
> - Interfaces defined before implementation (interface-first development)
> - Tests as ground truth — written before implementation, run automatically
> - Guardrails enforced structurally (hooks that block violations), not just documented as guidelines
>
> **Guidelines** (recommended):
> - Specific file layout (the 5-file module pattern shown below is a strong default)
> - README template and content conventions
> - Test fixture patterns and file splitting thresholds
> - Agent role separation and scope boundaries

These are drawn from production systems. Examples use a generic task management app — `task`, `user`, `notification` modules. Simple enough to be obvious, concrete enough to be useful.

### Standard 3: Predictable Structure Over Clever Structure

Every module looks exactly the same:

```
modules/task/
├── __init__.py       # public exports only
├── service.py        # all business logic
├── orm.py            # database tables
├── README.md         # design document
└── tests/
    └── test_service_task.py   # correctness tests (naming per Test Infrastructure)
```

This is deliberately boring. Clever code has cognitive overhead — unusual patterns require explanation. Boring, predictable code has zero overhead. An agent (or a new team member) that understands one module understands all of them. Business logic is always in `service.py`. Tables are always in `orm.py`. Public API is always in `__init__.py`.

When agents don't know where to look, they look everywhere and make assumptions. Predictable structure is the navigation system.

### Standard 4: Module READMEs as Executable Specs

For humans, documentation helps. For agents, documentation is the specification.

Every module README should cover:

- **Role**: one paragraph, what this module owns
- **What it does NOT own**: explicit non-ownership (just as important)
- **Public interface**: every method, what it does, what it raises
- **Database tables**: what data it stores
- **Dependencies**: what other services it requires
- **Test command**: how to run this module's tests in isolation

The "does NOT own" section is particularly important. Without it, agents optimize locally — they put things in the nearest module that seems plausible. With it, agents get a clear answer when they ask "should this go here?" — and sometimes the answer is explicitly "no, that belongs to `notification`."

A module without a README is incomplete, regardless of whether the code works.

### Standard 5: Interfaces Before Implementation

Before any implementation is written, the contract should exist:

```python
# common/interfaces.py — written first, before service.py exists

class TaskServiceInterface(Protocol):
    def create_task(self, data: TaskCreate) -> TaskResponse: ...
    def assign_task(self, task_id: UUID, user_id: str) -> TaskResponse: ...
    def complete_task(self, task_id: UUID) -> TaskResponse: ...
    def list_tasks(self, user_id: str, limit: int, offset: int) -> Tuple[List[TaskResponse], int]: ...

class NotificationServiceInterface(Protocol):
    def notify_assignment(self, entity_id: str, user_id: str) -> None: ...
    def notify_completion(self, entity_id: str) -> None: ...
```

This does several things at once:

- **Defines scope.** If a method isn't in the interface, it isn't part of this service. No ambiguity.
- **Enables parallel work.** Agent A implements the task module. Agent B implements notification. They never coordinate directly — both work against the same interfaces file. When they're done, the composition root — the single startup location where concrete implementations are constructed and injected — wires them together and they fit.
- **Makes capabilities legible.** One file. Read it. Know what the entire system can do.

The interface is the AI-to-AI communication protocol.

### Standard 6: Tests as Ground Truth

For humans, tests catch regressions. For agents, tests define correct behavior in machine-readable form.

```python
def test_assign_task_raises_if_user_not_found(task_service):
    task = task_service.create_task(TaskCreate(title="Fix the bug"))
    with pytest.raises(UserNotFoundError):
        task_service.assign_task(task.task_id, "nonexistent-user-id")

def test_complete_task_triggers_notification(task_service, mock_notification_service):
    task = task_service.create_task(TaskCreate(title="Write the tests"))
    task_service.assign_task(task.task_id, "user-123")
    task_service.complete_task(task.task_id)
    mock_notification_service.notify_completion.assert_called_once_with(str(task.task_id))
```

Without tests, an agent has no feedback loop. It produces code that compiles and returns something plausible — but whether the behavior is correct is unknowable without running the full system manually.

With tests — especially written before implementation — agents work against a concrete, unambiguous spec. Failures are immediate, specific, and self-correcting. A post-edit hook that automatically runs the module's test suite and injects failures back into the agent's context closes the loop entirely. The agent writes code, tests run, failures appear, agent corrects. No human required for basic correctness verification.

Tests are not quality assurance for agents. They are the ground truth.

### Standard 7: Guardrails Beat Guidelines

Here's the distinction that matters most in practice.

A contributing guide that says "don't put business logic in route handlers" is a **guideline**. An agent might read it, might not, might decide this case is special.

A hook that intercepts every file write and rejects it if business logic appears in a route handler is a **guardrail**. The agent cannot violate the rule — the tool call is rejected. It receives:

```
ARCHITECTURE VIOLATION: Business logic detected in routes/task.py.
Business logic belongs in modules/task/service.py.
See docs/architecture.md for module structure.
```

The agent knows what it did wrong, why, and exactly what to do instead. No re-reading the contributing guide. No judgment call about whether this case is an exception.

The distinction matters at scale. Guidelines work when everyone reads them, remembers them, and consistently chooses to follow them. Guardrails work regardless. They don't rely on memory or discipline — they make violations structurally impossible or at minimum explicit and visible.

When exceptions are genuinely needed:

```python
from modules.task.orm import TaskORM  # nocheck: arch-guard
# This is the composition root — the only valid cross-module import location
```

You can still do the thing. But you do it intentionally, visibly, and with an explanation. `grep '# nocheck'` shows every exception in the codebase.

**Encode constraints in tooling, not just documentation.**

### Standard 8: Bounded Scope Creates Reliable Agent Work

The best agent tasks have a defined start (read these files), a clear scope (touch only these files), and a verifiable end (tests pass, review checks clear). "Implement the notification module" is a great agent task. "Improve the app" is not.

Architecture designed so most feature work lives within a single module creates natural, right-sized work units. When that's true, agent tasks are bounded, predictable, and auditable. When it isn't — when adding a feature requires changes scattered across five modules — agent tasks become unpredictable, the judgment calls interact, and the result is hard to review and hard to roll back.

### Standard 9: Separate Roles, Separate Agents

The same agent should not write code, review architecture, and do code quality review. A well-defined agent system has distinct roles:

- **Implementer**: follows the TDD workflow, builds one module at a time, touches nothing else
- **Architecture reviewer**: reads code, produces a compliance report, cannot write code
- **Code quality reviewer**: finds bugs, performance issues, security problems, produces prioritized findings

These roles are enforced — the architecture reviewer literally has no write tools available. It cannot accidentally fix things while reviewing them. An agent with a narrowly scoped role does that role reliably.

Give agents one job.
