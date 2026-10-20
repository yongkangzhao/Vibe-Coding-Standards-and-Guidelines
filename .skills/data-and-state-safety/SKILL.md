---
name: data-and-state-safety
description: Enforces explicit data lifecycle, concurrency invariants, deterministic persistence, and durable external-effect coordination. Use when implementing deletion or erasure, counters/balances/quotas, replace-all writes, list queries, transactions/savepoints, ORM migrations, idempotent mutations, or provider calls.
---

# Data & State Safety

> A locally obvious mutation is often globally wrong. Safe state changes require a
> declared lifecycle, concurrency invariant, transaction boundary, and recovery path.

## When to use this

- Adding, changing, retaining, deleting, anonymizing, exporting, or restoring data.
- Writing counters, balances, quotas, inventory, or bulk/replace-all mutations.
- Adding ORM fields or schema migrations.
- Calling an external provider as part of a state transition.
- Implementing retries, idempotency, optimistic concurrency, or reconciliation.
- Rendering client state that may be stale or disputed by another writer.

## The standards

**Classify the data before choosing deletion behavior.** Record purpose, sensitivity,
owner, ordinary deletion, retention, erasure, derived copies, and recovery. Soft
deletion is appropriate only when recovery, audit, moderation, or relational
continuity is a documented requirement. Data minimization and verified erasure may
require hard deletion or irreversible anonymization. “Potential future value” is not
a retention policy.

**Cover every copy.** Lifecycle behavior includes the primary row plus projections,
search indexes, caches, media, analytics, providers, exports, and backup expiry.
Never claim erasure because one table has a `deleted_at` column.

**Wrap the entire mutation inside its savepoint.** Open the savepoint before the
write; a flush emitted earlier already belongs to the outer transaction.

```text
with savepoint:
    mutate
    flush
```

**State concurrency invariants explicitly.** Use atomic updates for counters;
locking or compare-and-swap for balances, quotas, and inventory; version
preconditions for replace-all writes; and idempotency keys for retried effects.

**Never hold database locks through uncontrolled network I/O.** Choose an explicit
coordination pattern: durable outbox/intent, idempotent provider operation plus
reconciliation, or compensation. “Provider first” and “database first” are both
unsafe without a recovery design.

**Make reads deterministic.** Every paginated or user-visible list has an explicit,
stable ordering with a tie-breaker.

**Authorize every mutation at the authoritative boundary.** Verify the caller’s
permission for the parent and operation; client visibility is not authorization.

**Keep runtime and migration schemas aligned.** Schema-dependent tests run real
migrations against the production engine; creating tables directly from ORM metadata
cannot prove deployability.

## Checklist

- [ ] Data purpose, sensitivity, owner, retention, deletion, erasure, and derived copies are documented.
- [ ] Ordinary deletion and verified erasure are distinct workflows where required.
- [ ] Savepoints begin before the mutation they must undo.
- [ ] Concurrent writes use atomic updates, locking, or version preconditions appropriate to the invariant.
- [ ] Retried effects have idempotency and durable reconciliation.
- [ ] No database transaction or lock spans uncontrolled network I/O.
- [ ] Lists have deterministic ordering and a stable tie-breaker.
- [ ] Mutations enforce authoritative permission/ownership.
- [ ] ORM/schema changes include matching migrations and migration tests.

## What breaks without this

- Required data is destroyed without recovery—or personal data is retained without purpose.
- A tombstone hides one row while caches, indexes, media, and providers keep the identity.
- Concurrent writers lose updates or overwrite newer state.
- A provider succeeds while local state fails, with no idempotent reconciliation path.
- Pagination changes order between requests and skips or duplicates records.
- Runtime code expects a column the deployed migration never created.

## Stack-specific examples

- **Backend:** see `references/backend.md` for persistence, migration, concurrency, and provider-coordination patterns.
- **Frontend:** see `references/frontend.md` for server authority, idempotency, version preconditions, and honest rollback.

## Related

- `docs/10-data-retention.md` for the lifecycle model.
- `guardrails-and-rule-flywheel` for turning repeated persistence defects into checks.
- `server-authoritative-state` for client/server ownership and stale-write prevention.
