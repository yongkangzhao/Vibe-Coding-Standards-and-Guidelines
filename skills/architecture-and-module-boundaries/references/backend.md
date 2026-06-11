# Backend: Modular Monolith in Python

Concrete form of the standards for a Python / FastAPI / SQLAlchemy stack. Stack details vary; the
structure (domain modules, public exports, interface-based DI, opaque IDs) does not.

## The 5-file module

Every domain module looks identical. An agent that understands `task/` understands `user/` and
`notification/`.

```
modules/task/
├── __init__.py             # public exports ONLY
├── service.py              # all business logic
├── orm.py                  # database tables (SQLAlchemy models)
├── README.md               # role, what it does NOT own, public interface, tables, test command
└── tests/
    └── test_service_task.py
```

```python
# modules/task/__init__.py — the module's entire public surface
from .service import TaskService

__all__ = ["TaskService"]
# orm.py is NOT exported. No other module may import TaskORM.
```

## Interfaces before implementation

Contracts live outside any single module so every module compiles against the same file. The
interface-first-development skill owns *how* you write these; here they pin down module ownership.

```python
# common/interfaces.py — written before service.py exists
from typing import Protocol
from decimal import Decimal

class OrganizationServiceInterface(Protocol):
    def get_commission_rate(self, org_id: str) -> Decimal: ...

class NotificationServiceInterface(Protocol):
    def notify_completion(self, entity_id: str) -> None: ...
```

A method not in the interface is not part of the service. That single line —
`get_commission_rate(org_id) -> Decimal` — ends the "which module owns the commission rate?"
debate by making the answer structural.

## Opaque IDs across boundaries

A module stores its own foreign keys, but never reaches into another module's ORM. Cross-module
references are plain strings/UUIDs, so there is no persistence coupling to unpick at extraction time.

```python
# modules/task/service.py
class TaskService:
    def __init__(self, db: Session, notifier: NotificationServiceInterface):
        self._db = db
        self._notifier = notifier            # injected interface, not a concrete import

    def complete_task(self, task_id: UUID) -> TaskResponse:
        task = self._db.get(TaskORM, task_id)
        if task is None:
            raise TaskNotFoundError(task_id)
        task.status = "done"
        self._db.commit()
        self._notifier.notify_completion(str(task.task_id))   # opaque id over the seam
        return TaskResponse.model_validate(task)
```

`notify_completion` takes `str`, not a `TaskORM` object — `notification` knows nothing about
`task`'s tables.

## The composition root

The single startup location that constructs concretes and injects them. It is the only place
allowed to import another module's internals (`# nocheck: arch-guard`), and the only place wiring
changes when a module becomes a service.

```python
# app/composition.py — the ONE wiring location
def build_services(db: Session) -> AppServices:
    notifier = NotificationService(db)

    # today: in-process call
    org: OrganizationServiceInterface = OrganizationService(db)
    # tomorrow: swap HERE only — every caller still compiles against the interface
    # org: OrganizationServiceInterface = OrganizationHTTPClient(settings.ORG_SERVICE_URL)

    return AppServices(
        task=TaskService(db, notifier=notifier),
        org=org,
    )
```

What the swap does **not** give you for free: timeouts, retries, partial failures, `Decimal`/`UUID`/
`datetime` serialization, the loss of one DB transaction spanning both modules, latency, and auth on
the new boundary. The `Protocol` keeps call sites stable; you still do the distributed-systems work.

## Enforce the boundary structurally

A guideline relies on memory; a guardrail rejects the write. A pre-commit / import linter or a
post-edit hook fails the build on an illegal import:

```python
# tools/check_imports.py (sketch) — run in CI and as a pre-edit hook
# Reject `from modules.<other>.orm import ...` and `.service import ...`
# unless the line ends with `# nocheck: arch-guard`.
```

```ini
# or with import-linter (.importlinter)
[importlinter:contract:module-isolation]
name = Domain modules are independent
type = independence
modules = modules.task | modules.user | modules.notification
```

`grep -rn "# nocheck" modules/ app/` then lists every intentional exception in the codebase.
