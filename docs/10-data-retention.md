# Standard 15: Data Retention as an Architectural Decision

> Agents default to hard-delete. Tell an agent to "delete a post" and it writes `DELETE FROM posts WHERE id = ?` — permanent, irrecoverable removal. This is the locally obvious implementation, and agents will produce it every time unless the codebase structurally steers them toward soft-delete. User-generated content is a business asset that feeds analytics, recommendations, and audit trails. Once hard-deleted, it cannot be recovered.

> **Standards** (must follow):
> - Soft-delete user-generated content (set `deleted_at`, never `DELETE FROM`)
> - Filter deleted content in all queries (`WHERE deleted_at IS NULL`)
> - Block new interactions on deleted content (raise errors, don't create orphaned records)
>
> **Guidelines** (recommended):
> - Specific soft-delete implementation pattern (`deleted_at` timestamp column)
> - Archive-to-cold-storage strategy for cleanup
> - How to encode soft-delete as a rule and enforce it in hooks

When an agent implements a "delete" feature, it will default to `DELETE FROM` — permanent removal. This is usually wrong.

User-generated content — posts, comments, messages, votes, interactions — is a business asset. It feeds analytics, recommendation systems, and potentially ML models. Hard-deleting it destroys value that cannot be recovered.

**Default to soft-delete everywhere:**

```python
# ❌ Agent's instinct — permanent deletion
def delete_post(self, post_id: UUID) -> None:
    self._db.delete(post)
    self._db.flush()

# ✅ Correct — soft delete preserves data
def delete_post(self, post_id: UUID) -> None:
    post.deleted_at = datetime.now(timezone.utc)
    self._db.flush()
```

The follow-up requirements:
1. **Filter deleted content in queries**: `WHERE deleted_at IS NULL`
2. **Block new interactions on deleted content**: voting on a deleted post should raise `NotFoundError`, not create an orphaned vote
3. **Never hard-delete user-generated content rows**: if cleanup is needed, archive to cold storage

This is an architectural decision that affects every module. Encode it in a rule, enforce it in hooks, and make it explicit in module READMEs. When an agent is told to "delete" something, the rule should guide it to soft-delete automatically.
