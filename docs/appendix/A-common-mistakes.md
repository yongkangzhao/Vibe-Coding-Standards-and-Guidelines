# Appendix: Common Mistakes That Become Rules

> Agents repeat the same mistakes across sessions because they have no memory of previous feedback. A mistake caught in PR review on Monday will reappear on Tuesday from a fresh agent session. The only way to break this cycle is to convert repeated PR feedback into persistent, automated rules that fire before the code is committed — transforming human review effort into structural enforcement that works across every session, for every agent, permanently.

> **Standards** (must follow):
> - When the same mistake appears in two or more PRs, codify it as a rule
> - Rules must be enforced automatically (hooks, linters) where possible
>
> **Guidelines** (recommended):
> - Specific rule format and location (`.claude/rules/` or equivalent)
> - Automation approach (pre-commit, pre-push, post-edit hooks)
> - Reference the PRs where the pattern was first observed for traceability

These patterns appeared repeatedly in production codebases using AI agents. Each was first caught in PR review, then codified as a rule, then (where possible) automated as a hook.

### 1. Savepoints Must Wrap the Entire Mutation

```python
# ❌ Flush before the savepoint — the INSERT is already emitted in the
#    outer transaction, so rolling back to the savepoint can't undo it
self._db.add(obj)
self._db.flush()              # row written here, OUTSIDE the savepoint
sp = self._db.begin_nested()  # savepoint opened too late

# ✅ Open the savepoint first; the mutation happens inside it
with self._db.begin_nested():
    self._db.add(obj)
    self._db.flush()          # INSERT is inside the savepoint, fully reversible
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

The SQL-side increment removes the read-modify-write lost update. It is not the whole concurrency story: if you need the resulting value, add `RETURNING`; if the counter backs an invariant like a balance or quota, you still need the right isolation level or `SELECT ... FOR UPDATE`. Atomic increment closes one race, not all of them.

### 3. ORM Columns Must Match Migration Columns

Every column in the ORM model needs a corresponding column in the Alembic migration. Agents frequently add ORM columns and forget the migration — the code "works" in tests (where `create_all()` builds from the ORM) but crashes in production (where Alembic migrations define the schema).

### 4. Every List Query Needs `order_by`

Without `order_by`, result ordering is non-deterministic across databases and even across runs. Agents rarely add it unprompted. A pre-push hook can flag `scalars().all()` with no `order_by` on the same statement — a grep heuristic catches the common single-expression case but misses statements built across several lines or variables, so treat it as a fast first filter rather than a guarantee. AST-level analysis is what makes the check sound.

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

### 8. Router-Level Attribute Access That Seam-Mocked Tests Never Reach

```python
# auth model exposes `user_id`
class CurrentUser(BaseModel):
    user_id: str

# ❌ handler reads a field that doesn't exist → AttributeError → 500 for every caller
@router.get("/items")
def list_items(current_user = Depends(get_current_user)):
    return service.list_for(current_user.uid)   # .uid is not .user_id
```

The unit tests stay green because they call `service.list_for(...)` directly, or mock the auth dependency, or stub the network seam — none of them execute the line in the router that reads `.uid`. The bug lives in the three lines of glue *between* the seams everyone mocked, so every isolated test passes while every real request 500s. The rule: a unit test that mocks the network or calls the service directly cannot certify a route. **An end-to-end test must drive the real route** — through the real router, the real dependency, the real auth model — or the wiring between mocked seams is never exercised.

### 9. An Error Rendered as Empty — the "200 Lie" on the Client

```javascript
// ❌ empty-state keyed on message truthiness
if (!error.message) return <EmptyState text="No items yet" />;

// an HTTP/2 500 arrives with empty statusText, so error.message === ""
// → the 500 renders as "No items yet" instead of an error
```

A failed fetch whose empty-state is keyed on the *truthiness of an error string* will swallow exactly the errors that carry no message. HTTP/2 responses have an empty `statusText`, so a thrown `Error(statusText)` has `message === ""`, which is falsy, which routes a 500 straight into the "nothing here yet" branch. The user sees a calm empty page over a server on fire. The rule: branch on an **explicit error flag** (`isError`, a discriminated result, a status code) — never on whether an error happens to carry a non-empty message.

### 10. Fabricated Zero for Absent Data

```javascript
// ❌ absent ≠ zero — null renders as a real-looking number
<span>{formatMoney(balance)}</span>      // balance == null → "$0.00"
<span>{formatPercent(winRate)}</span>    // winRate == null → "0%"

// ✅ absent renders as an em-dash; only a real 0 from the backend shows "0"
<span>{balance == null ? "—" : formatMoney(balance)}</span>
```

`null` (the backend hasn't computed it, the endpoint is missing, the join returned nothing) is not `0`. Coercing it to `$0.00` / `0%` / `0` invents a fact the system never asserted — a user reads "$0.00 balance" as "I have no money," not "we don't know your balance yet." Render absent data as an em-dash (or hide the element); reserve `0` for a real zero the backend actually returned.

### 11. Optimistic Mutation That Masks a Failure

```javascript
// ❌ flip to success before the server confirms, never reconcile on error
function save(edit) {
  setRows(applyLocally(edit));   // UI shows success instantly
  api.patch(edit);               // fire-and-forget; rejection ignored
}
```

Optimistic updates are fine — *unreconciled* optimistic updates are a silent data-loss bug. If the UI flips to the success state before the server confirms and the mutation then fails, the user sees a phantom success while their edit was rejected and discarded. They navigate away believing it saved. The rule: an optimistic update must roll back to the prior state (and surface the error) when the server rejects it — the success state is provisional until the response confirms it.

### 12. Numeric Formatter That Coerces Garbage

```javascript
// ❌ Number() happily coerces nonsense to a plausible-looking value
Number("1e3")   // 1000  → rendered as 100000%
Number("0x14")  // 20
Number("")      // 0
Number(" 42 ")  // 42

// ✅ gate with a decimal regex BEFORE coercing
const isDecimal = /^-?\d+(\.\d+)?$/.test(raw.trim());
return isDecimal ? formatPercent(Number(raw)) : "—";
```

`Number()` (and the unary `+`) accept scientific notation, hex literals, and the empty string, turning `"1e3"` into `1000` and rendering it as `100000%`. A formatter that trusts its input will confidently display a garbage value as a real statistic. Validate against an explicit decimal pattern first; coerce only what passes.

### 13. Stale Deploy vs. Buggy Committed Code

When the deployed app misbehaves, the reflex is to suspect the deploy — a stale image, a bad roll-out, a CDN cache, the wrong tag promoted. That reflex sends you debugging infrastructure for an hour when the committed code simply *is* the bug: the deploy faithfully shipped exactly what's on the branch. Before inventing an infra cause, **`git diff` the working tree and read the committed source for the failing path** — confirm the code that's deployed is actually correct before you go hunting for a deploy that "didn't take." Most of the time the deploy worked perfectly; it just deployed a bug.

---

Each of these rules started as a PR comment. The first time, it was feedback. The second time, it was a pattern. The third time, it became a rule. The fourth time, it was automated. This is the flywheel that makes AI-assisted engineering sustainable.
