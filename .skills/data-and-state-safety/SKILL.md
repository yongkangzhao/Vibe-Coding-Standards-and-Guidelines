---
name: data-and-state-safety
description: Enforces safe data-and-state handling — soft-delete user content by default with an audited legal hard-delete path, plus the recurring persistence pitfalls (savepoint wrapping, atomic counters, no DB transaction across external calls, deterministic order_by, ownership checks, ORM/migration parity). Use this when writing or reviewing any delete/erase feature, counter or balance increment, list/query endpoint, mutating repository method, ORM model or Alembic migration, payment/Stripe or other external-API flow, optimistic UI update, or idempotent mutation — on backend OR frontend — even if the user doesn't name it explicitly.
---

# Data & State Safety

> Deleting data, mutating counters, and persisting state are the places where a locally-obvious implementation is globally wrong. Tell an agent to "delete a post" and it writes `DELETE FROM` — irreversible. Tell it to "increment a count" and it writes a read-modify-write race. Agents start every session with no memory of the data loss or the lost update they caused last week, so the codebase itself must structurally steer them toward the safe pattern. These rules turn "obviously correct" instincts into actually-correct persistence.

## When to use this
- **Backend:** implementing or reviewing a delete/erase feature; writing repository methods that mutate rows; incrementing counters, balances, or quotas; building a list/query endpoint; opening a savepoint or transaction; calling an external API (Stripe, email, webhooks) inside data flow; adding an ORM column or Alembic migration; handling a GDPR/CCPA erasure request.
- **Frontend:** deciding where an authorization check lives; firing a mutation that could be retried or double-clicked; applying an optimistic update; editing shared client state that another tab/session also writes.

## The standards

**Soft-delete user-generated content by default.** Posts, comments, votes, and messages are business assets feeding analytics and audit trails; hard-delete destroys value that cannot be recovered, and an agent's default `DELETE FROM` does it silently.
```
delete(id):  row.deleted_at = now()      # not: DELETE FROM table
```

**Filter soft-deleted rows in every query, and block new interactions on them.** A query that forgets `WHERE deleted_at IS NULL` resurrects deleted content; voting on a deleted post must raise `NotFound`, not create an orphaned record.

**Keep a real, audited hard-delete path for legal erasure.** "Soft-delete by default" is not "never hard-delete" — GDPR Article 17 and CCPA can *require* irreversible removal, and a `deleted_at` tombstone that keeps the row does not satisfy the law. Build it deliberately: scoped to verified erasure requests, audited, separate from the everyday delete flow.

**Wrap the entire mutation inside the savepoint.** Open the savepoint *before* the write; a `flush` emitted before `begin_nested` is already in the outer transaction and rolling back the savepoint can't undo it.
```
with begin_nested():     # open FIRST
    add(obj); flush()    # mutation lives inside, fully reversible
```

**Increment counters atomically at the SQL layer.** `obj.count += 1` is a read-modify-write — two concurrent requests both read 5 and both write 6, losing an update. Push it into the database.
```
UPDATE t SET count = count + 1 WHERE id = ?
```
This closes one race, not all of them: if you need the new value, add `RETURNING`; if the counter backs an invariant (balance, quota), you still need `SELECT ... FOR UPDATE` or the right isolation level.

**Never hold a DB transaction open across an external API call.** A slow network call (Stripe, email) while a row is locked stalls or deadlocks other writers. Do the external call first, then write.
```
charge = stripe.charge(...)   # no DB state held open
wallet.balance += charge; flush()
```

**Give every list query a deterministic `order_by`.** Without it, ordering varies across databases and even across runs, breaking pagination and tests. Agents rarely add it unprompted.

**Check ownership on every mutating method.** If an entity has an owner (`created_by`), every method mutating it or its sub-resources must verify the caller owns the parent — agents implementing nested CRUD consistently forget the parent check.

**Keep ORM columns and migration columns in lockstep.** A new ORM column with no migration "works" in tests (schema built from the ORM) and crashes in production (schema built by migrations). Every ORM column needs a matching migration column.

## Checklist
- [ ] Delete paths set `deleted_at`; no `DELETE FROM` / `db.delete()` on user content outside the legal-erasure path
- [ ] All read queries filter `WHERE deleted_at IS NULL`; interactions on soft-deleted rows raise `NotFound`
- [ ] A separate, audited hard-delete path exists for verified GDPR/CCPA erasure requests
- [ ] Savepoints (`begin_nested`) are opened before the mutation they must be able to roll back
- [ ] Counter/balance updates run as SQL `col = col + n`, with `RETURNING` or row locks where the value or invariant matters
- [ ] No `flush`/`commit` left open across an external API call
- [ ] Every list query has an explicit `order_by`
- [ ] Every mutating method verifies caller ownership of the parent entity
- [ ] Every ORM column has a corresponding migration column

## What breaks without this
- A "delete" feature ships `DELETE FROM` and permanently destroys user content that fed analytics and audit trails — unrecoverable.
- An erasure request is answered with a soft-delete tombstone that keeps the row, leaving the company in breach of GDPR Article 17 / CCPA.
- Two concurrent likes both read `count=5` and write `count=6`; the like counter silently drifts below reality.
- A payment flow holds a row lock across a slow Stripe call, stalling unrelated writers and risking deadlock; meanwhile a forgotten `order_by` makes pagination non-deterministic and a forgotten migration crashes production on deploy.

## Stack-specific examples
- **Backend:** see `references/backend.md` (SQLAlchemy/Alembic/pytest — soft + hard delete, savepoints, atomic increment, transaction-vs-external-call, order_by, ownership, ORM/migration parity)
- **Frontend:** see `references/frontend.md` (server-side authz, idempotency keys, optimistic-update rollback)

## Related
- Full rationale: `docs/10-data-retention.md` and `docs/appendix/A-common-mistakes.md` in this repo
- `guardrails-and-rule-flywheel` for the mechanics of turning each of these — once you've seen the mistake twice — into a lint rule or pre-push/pre-commit hook so it never recurs. Don't re-implement hook plumbing here; that skill owns it.
