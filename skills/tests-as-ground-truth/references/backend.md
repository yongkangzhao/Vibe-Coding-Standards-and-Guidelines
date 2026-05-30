# Backend: Tests as Ground Truth (pytest + real Postgres)

Concrete realization of the SKILL standards for a Python / SQLAlchemy / pytest / Alembic stack.

## Layout — tests mirror the module

```
modules/task/
├── __init__.py
├── service.py
├── orm.py
├── README.md
└── tests/
    ├── __init__.py
    ├── conftest.py                 # shared fixtures, auto-discovered by pytest
    ├── test_service_task.py        # one domain per file
    ├── test_service_assignment.py
    └── test_security_task.py       # exploit / authz tests
```

`conftest.py` is discovered automatically — no imports. Keep each `test_service_<domain>.py` under ~500 lines; when it grows, move tests into a new domain file (pure refactor: same count, same results).

## Disposable real-engine fixture

Spin up the **same engine as production** (Postgres) in a throwaway container per session, not in-memory SQLite. `pytest-postgresql` or `testcontainers` both work; below uses `testcontainers`.

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from testcontainers.postgres import PostgresContainer
from alembic.config import Config
from alembic import command

@pytest.fixture(scope="session")
def pg_engine():
    # Real Postgres in a disposable container — destroyed when the session ends.
    with PostgresContainer("postgres:16") as pg:
        engine = create_engine(pg.get_connection_url(), future=True)
        cfg = Config("alembic.ini")
        cfg.set_main_option("sqlalchemy.url", pg.get_connection_url())
        command.upgrade(cfg, "head")          # run real migrations, not metadata.create_all
        yield engine
        engine.dispose()

@pytest.fixture
def db_session(pg_engine):
    # Per-test transaction, rolled back so tests stay isolated and fast.
    conn = pg_engine.connect()
    txn = conn.begin()
    Session = sessionmaker(bind=conn, future=True)
    session = Session()
    try:
        yield session
    finally:
        session.close()
        txn.rollback()
        conn.close()
```

Running real Alembic migrations (not `Base.metadata.create_all`) means the schema, constraints, and partial indexes under test are exactly the ones production runs.

## Mock across the seam, run real underneath

The storage engine under the unit you test is real. Only the *neighbor module* is mocked — at its interface, which interface-first development already defined.

```python
# tests/conftest.py (continued)
from unittest.mock import create_autospec
from common.interfaces import NotificationServiceInterface
from modules.task.service import TaskService

@pytest.fixture
def mock_notification_service():
    return create_autospec(NotificationServiceInterface, instance=True)

@pytest.fixture
def task_service(db_session, mock_notification_service):
    # Real DB session (real engine), mocked neighbor service across the module boundary.
    return TaskService(db=db_session, notifier=mock_notification_service)
```

## Tests assert observable behavior

```python
# tests/test_service_assignment.py
import pytest
from modules.task.service import TaskCreate
from common.errors import UserNotFoundError

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

## Engine-dependent behavior a substitute would hide

These pass on Postgres and silently misbehave (or error) on an in-memory substitute — which is the whole point of running the real engine:

```python
def test_claim_locks_row(task_service, db_session):
    task = task_service.create_task(TaskCreate(title="contended"))
    # SELECT ... FOR UPDATE actually locks on Postgres; SQLite ignores it.
    locked = task_service.claim_for_update(task.task_id)
    assert locked.task_id == task.task_id

def test_unique_constraint_fires(task_service):
    from common.errors import DuplicateSlugError
    task_service.create_task(TaskCreate(title="dup", slug="x"))
    with pytest.raises(DuplicateSlugError):   # partial unique index enforced by Postgres
        task_service.create_task(TaskCreate(title="dup2", slug="x"))
```

## Run the module's tests in isolation

```bash
pytest modules/task/tests -q
```

A post-edit hook can run exactly this and feed failures back into the agent's context, closing the loop with no human in it.
