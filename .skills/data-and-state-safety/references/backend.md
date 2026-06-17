# Backend: Data & State Safety (Python / SQLAlchemy / Alembic / pytest)

Concrete patterns for the SKILL.md standards. APIs shown are SQLAlchemy 2.x style.

## Soft-delete by default

```python
# ❌ Agent's instinct — permanent, irrecoverable
def delete_post(self, post_id: UUID) -> None:
    post = self._get(post_id)
    self._db.delete(post)
    self._db.flush()

# ✅ Soft delete preserves the business asset
def delete_post(self, post_id: UUID) -> None:
    post = self._get(post_id)
    post.deleted_at = datetime.now(timezone.utc)
    self._db.flush()
```

### Filter deleted rows in every read

```python
stmt = (
    select(PostORM)
    .where(PostORM.facility_id == facility_id)
    .where(PostORM.deleted_at.is_(None))   # never resurrect deleted content
    .order_by(PostORM.created_at.desc())   # deterministic — see below
)
```

### Block interactions on deleted rows

```python
def add_vote(self, post_id: UUID, user_id: UUID) -> Vote:
    post = self._db.scalar(
        select(PostORM)
        .where(PostORM.id == post_id)
        .where(PostORM.deleted_at.is_(None))
    )
    if post is None:
        raise NotFoundError("post not found")  # not an orphaned vote
    ...
```

## The legal hard-delete path (GDPR Art. 17 / CCPA)

Soft-delete protects business data; this satisfies the law. Keep it separate, scoped to verified requests, and audited.

```python
def erase_user_data(self, subject_id: UUID, request_id: UUID, actor: UUID) -> None:
    """Irreversible erasure for a verified right-to-be-forgotten request."""
    self._audit.record(
        action="gdpr_erasure", subject=subject_id,
        request_id=request_id, actor=actor, at=datetime.now(timezone.utc),
    )
    # Real DELETE here — this is the deliberate exception, not the default flow.
    self._db.execute(delete(PostORM).where(PostORM.author_id == subject_id))
    self._db.execute(delete(UserORM).where(UserORM.id == subject_id))
    self._db.flush()
```

## Savepoints wrap the entire mutation

```python
# ❌ Flush before the savepoint — INSERT is already in the outer transaction
self._db.add(obj)
self._db.flush()                 # written OUTSIDE the savepoint
sp = self._db.begin_nested()     # too late; rollback can't undo the INSERT

# ✅ Open the savepoint first
with self._db.begin_nested():
    self._db.add(obj)
    self._db.flush()             # INSERT lives inside the savepoint, reversible
```

## Atomic counter increments

```python
# ❌ Read-modify-write: two requests read 5, both write 6 — one update lost
post.like_count += 1

# ✅ SQL-level atomic increment
self._db.execute(
    update(PostORM)
    .where(PostORM.id == post_id)
    .values(like_count=PostORM.like_count + 1)
)

# Need the resulting value? Add RETURNING:
new_count = self._db.scalar(
    update(PostORM)
    .where(PostORM.id == post_id)
    .values(like_count=PostORM.like_count + 1)
    .returning(PostORM.like_count)
)
```

If the counter backs an invariant (a wallet balance, a seat quota), atomic increment alone is not enough — lock the row or raise the isolation level:

```python
wallet = self._db.scalar(
    select(WalletORM).where(WalletORM.id == wid).with_for_update()
)
if wallet.balance < amount:
    raise InsufficientFundsError()
wallet.balance -= amount
```

## Never hold a transaction open across an external call

```python
# ❌ DB row locked while waiting on the network
wallet.balance += amount
self._db.flush()
stripe_id = stripe.charge(...)   # slow call, lock held — stalls other writers

# ✅ External call first, then the DB write
stripe_id = stripe.charge(...)   # no DB state open
wallet.balance += amount
self._db.flush()
```

## Deterministic order_by on every list query

```python
# ❌ Non-deterministic across databases and runs — breaks pagination + tests
rows = self._db.scalars(select(PostORM)).all()

# ✅ Explicit, stable ordering (tie-break on id for fully-deterministic order)
rows = self._db.scalars(
    select(PostORM).order_by(PostORM.created_at.desc(), PostORM.id)
).all()
```

## Ownership checks on every mutating method

```python
def update_court(self, facility_id: UUID, court_id: UUID, caller: UUID, **fields) -> Court:
    facility = self._db.get(FacilityORM, facility_id)
    if facility is None or facility.created_by != caller:
        raise ForbiddenError("not your facility")   # check the PARENT, not just the court
    court = self._get_court(facility_id, court_id)
    for k, v in fields.items():
        setattr(court, k, v)
    self._db.flush()
    return Court.model_validate(court)
```

## ORM columns must match migration columns

Tests build the schema from the ORM (`Base.metadata.create_all()`); production builds it from Alembic. A column in one and not the other passes tests and crashes on deploy.

```python
# models.py
class PostORM(Base):
    __tablename__ = "posts"
    pinned: Mapped[bool] = mapped_column(default=False)   # new column
```

```python
# migrations/versions/xxxx_add_pinned.py  — MUST accompany the ORM change
def upgrade() -> None:
    op.add_column("posts", sa.Column("pinned", sa.Boolean(),
                  nullable=False, server_default=sa.false()))

def downgrade() -> None:
    op.drop_column("posts", "pinned")
```

A test that runs the migrations (not just `create_all`) against the model and diffs the schema catches the drift before deploy.
