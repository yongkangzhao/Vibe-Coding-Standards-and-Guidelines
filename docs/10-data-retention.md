# Standard 16: Treat Data Lifecycle and State Safety as Architecture

> “Delete” is not one operation. Product removal, account closure, fraud retention,
> legal hold, backup expiry, and verified erasure have different requirements.

> **Standards**
> - Classify data by purpose, sensitivity, owner, retention, and erasure behavior.
> - Collect and retain only what has a documented purpose.
> - Separate ordinary product deletion from verified legal erasure.
> - Make concurrency, idempotency, and external-side-effect boundaries explicit.
>
> **Guidelines**
> - Soft deletion is useful when recovery, audit, or relational continuity is a
>   legitimate documented requirement; it is not a universal default.
> - Use automated policy manifests and tests to keep schema and lifecycle behavior aligned.

## Classify before implementing

For each table, document:

| Question | Example answers |
|---|---|
| Purpose | account operation, user content, security audit, billing record |
| Sensitivity | public, internal, personal, credential, regulated |
| Authoritative owner | identity module, content module, ledger |
| Ordinary deletion | immediate hard delete, tombstone, anonymize, archive |
| Retention | none, fixed period, contractual, statutory |
| Erasure | delete, irreversibly anonymize, retain under exception |
| Derived copies | cache, search index, analytics, media, provider |
| Recovery | supported window and authority, or explicitly none |

“Keep it because it may be valuable later” is not a retention policy. Neither is
“hard-delete everything” a safe default. Purpose, consent, contractual obligations,
law, security, and user expectations determine the lifecycle.

## Ordinary deletion and legal erasure

Ordinary deletion may preserve a tombstone or audit relationship when recovery,
moderation, reconciliation, or referential integrity requires it. Queries and
interactions must consistently respect that state.

Verified erasure is a separate, audited workflow. It must address all copies:
primary rows, projections, search indexes, caches, media, providers, exports, and
backup expiry policy. Where an obligation requires retention, minimize and isolate
the retained fields and document the authority.

Never claim compliance because a `deleted_at` column exists.

## Concurrency and transactions

State changes need explicit invariants:

- counters use atomic updates when concurrent writers exist;
- balances, quotas, and inventory use locking or compare-and-swap appropriate to
  the invariant;
- replace-all writes carry a version precondition;
- savepoints wrap the mutation they are expected to undo;
- list queries have deterministic ordering;
- ORM and migration schemas remain in parity.

The database transaction is the consistency boundary. Do not keep it open while
waiting on an uncontrolled network call.

## External effects

“Call the provider first” and “write the database first” can both be wrong. Choose a
pattern based on the effect:

- idempotency key plus retryable provider operation;
- durable intent/outbox written in the transaction, delivered after commit;
- provider call followed by idempotent reconciliation;
- compensating action when an atomic outcome is impossible.

Record provider identifiers and reconciliation state durably. Never hold row locks
through a slow external request, and never assume a process-local callback is durable.

## Enforcement

Schema changes that add personal or regulated data should fail CI unless they update
the lifecycle policy and applicable export/erasure tests. Repeated mistakes—missing
ordering, migration drift, unsafe increments—should become automated checks with a
traceable suppression mechanism.

Data safety is not a cleanup phase. It is part of the contract for every stateful
feature.
