# Backend: Clean Migration (Python / FastAPI / SQLAlchemy / pytest)

Concrete patterns for the SKILL.md standards. The failure mode is always the same: a surviving old path keeps the migrated bug alive.

## Making a parameter required — no `Optional` bypass

A scope argument was added so assignments are tenant-isolated. Leaving it optional means the un-scoped, leaky path is still reachable — which is the exact bug the change was meant to delete.

```python
# ❌ "for backward compat" — the un-scoped path survives forever
def assign(self, user_id: str, facility_id: UUID | None = None) -> Assignment:
    if facility_id is None:
        return self._legacy_assign(user_id)          # global, unscoped — the bug
    return self._scoped_assign(user_id, facility_id)

# ✅ Required arg; every caller migrated in the same change
def assign(self, user_id: str, facility_id: UUID) -> Assignment:
    return self._scoped_assign(user_id, facility_id)
```

Find every caller before you commit — the migration isn't done until this returns nothing:

```bash
rg -n '\.assign\(' --type py        # zero hits on the old 1-arg form
```

If a caller genuinely has no `facility_id`, that caller is a bug too — fix it here, don't give it a default to hide behind.

## Ripping out a concept — delete the name, don't re-export it

A `wallet` concept is being split into `transactions` and `memberships`. A re-export keeps the dead concept importable and agents keep wiring to it.

```python
# ❌ modules/wallet/__init__.py — shim keeps the deleted concept alive
from modules.transactions.service import TransactionService as WalletService  # noqa
# every `from modules.wallet import WalletService` still "works" → concept never dies

# ✅ The wallet module is gone. Callers import the real new home:
from modules.transactions import TransactionService
from modules.memberships import MembershipService
```

Then forbid the resurrection with a structural test:

```python
def test_wallet_concept_is_gone():
    import importlib
    with pytest.raises(ModuleNotFoundError):
        importlib.import_module("modules.wallet")   # old path must not resolve
```

## Most-correct over smallest-diff — normalized table, not a JSON hack

You need to track which users have read a message. The smallest diff is a JSON list on the row; the correct change is a join table — and it removes a lost-update race the JSON hack would keep.

```python
# ❌ Smallest diff — read-modify-write a JSON column; concurrent reads clobber
msg.read_by = list(set(msg.read_by) | {user_id})    # lost update under concurrency
self._db.flush()

# ✅ Most-correct — a real table + migration; insert is atomic and idempotent
class MessageReadORM(Base):
    __tablename__ = "message_reads"
    message_id: Mapped[UUID] = mapped_column(ForeignKey("messages.id"), primary_key=True)
    user_id:    Mapped[UUID] = mapped_column(primary_key=True)
    read_at:    Mapped[datetime] = mapped_column(default=lambda: datetime.now(timezone.utc))

self._db.execute(
    insert(MessageReadORM)
    .values(message_id=msg_id, user_id=user_id)
    .on_conflict_do_nothing()                        # no race, no duplicate
)
self._db.flush()
```

Ship the Alembic migration that creates `message_reads` in the same change, and delete the `read_by` column in that migration — don't leave both readable.

## Migrate all call sites, accept the fixture churn

When a service constructor goes from an optional injected dependency to a required one, update every construction site — including the ten test fixtures — rather than defaulting the dependency to `None`.

```python
# ❌ Optional injection keeps a no-permission-check path alive in prod
def __init__(self, db: Session, permissions: PermissionService | None = None):
    self._perm = permissions          # None → checks silently skipped

# ✅ Required; conftest and every caller updated in this change
def __init__(self, db: Session, permissions: PermissionService):
    self._perm = permissions
```

```python
# conftest.py — the churn is the point: the fixture now reflects reality
@pytest.fixture
def service(db, permission_service):
    return CourtService(db, permissions=permission_service)
```

A `None` default here would let a real route construct the service without permissions and skip every authorization check — the fixture update is cheaper than that hole.
