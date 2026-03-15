# Appendix: Common Mistakes That Become Rules

These patterns appeared repeatedly in production codebases using AI agents. Each was first caught in PR review, then codified as a rule, then (where possible) automated as a hook.

### 1. Savepoints Must Wrap the Entire Mutation

```python
# ❌ Mutation before savepoint — rollback won't undo it
self._db.add(obj)
sp = self._db.begin_nested()
self._db.flush()

# ✅ Context manager wraps everything
with self._db.begin_nested():
    self._db.add(obj)
    self._db.flush()
```

### 2. Counter Increments Must Be Atomic

```python
# ❌ Two concurrent requests both read count=5, both write count=6
obj.count += 1

# ✅ SQL-level atomic increment
self._db.execute(
    update(TableORM).where(...).values(count=TableORM.count + 1)
)
```

### 3. ORM Columns Must Match Migration Columns

Every column in the ORM model needs a corresponding column in the Alembic migration. Agents frequently add ORM columns and forget the migration — the code "works" in tests (where `create_all()` builds from the ORM) but crashes in production (where Alembic migrations define the schema).

### 4. Every List Query Needs `order_by`

Without `order_by`, result ordering is non-deterministic across databases and even across runs. Agents rarely add it unprompted. A pre-push hook that greps for `scalars().all()` without a preceding `order_by` catches this consistently.

### 5. Ownership Checks on All Mutating Methods

If a module has an ownership model (e.g., a facility has a `created_by` field), then every method that mutates data under that entity must verify ownership. Agents implementing sub-resource CRUD (courts within a facility, pricing rules within a facility) consistently forget to check whether the caller owns the parent entity.

### 6. Never Hold a Database Transaction Open Across an External API Call

```python
# ❌ DB locks held while waiting for Stripe
wallet.balance += amount
self._db.flush()
stripe_id = stripe.charge(...)  # slow network call while DB row is locked

# ✅ External call first, then DB write
stripe_id = stripe.charge(...)  # no DB state open
wallet.balance += amount
self._db.flush()
```

### 7. Interface Signatures Must Match Implementation Exactly

Python's `isinstance` check for `Protocol` compliance only verifies method names exist — not parameter signatures. An implementation can have a completely wrong signature and still pass the isinstance check. After implementing any method, manually verify the signature matches the Protocol.

---

Each of these rules started as a PR comment. The first time, it was feedback. The second time, it was a pattern. The third time, it became a rule. The fourth time, it was automated. This is the flywheel that makes AI-assisted engineering sustainable.
