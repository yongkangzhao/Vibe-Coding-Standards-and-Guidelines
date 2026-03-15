# What Breaks Without These Standards

> Every failure mode described here exists in human-written codebases too. What's different with AI agents is the speed and confidence. Agents amplify existing chaos — they produce large volumes of plausible-looking code that compounds structural problems faster than any human team could. Feature duplication, logic leakage, and architectural drift that would take a human team months to accumulate can happen in days when agents are involved, because each agent session starts fresh with no memory of decisions made in previous sessions.

Before getting to solutions, let's name the failure modes precisely, because they matter:

**Feature duplication.** The same logic gets implemented in multiple places. A function that fetches a user's active subscription exists in three files, each written at a different time, each slightly different. Billing logic changes. One of them gets updated.

**Logic leakage.** Business rules that belong in one layer appear in all of them. The discount calculation lives in the API handler, in the frontend component, and hardcoded into a database query. Each has subtly different rounding behavior. None agree.

**Inconsistent patterns.** Some endpoints paginate using `len()` (loads the full table into memory). Others use `COUNT(*)`. Some fetch related data in a loop, one query per item. Others batch. The codebase doesn't have a standard — it has a history.

**Invisible regressions.** A change silently breaks something else. There's no fast feedback loop. You find it in production.

**Architectural drift.** The intended structure lives in a document or in someone's head. The actual structure is what the code does. These diverge over time — and faster when agents are involved, because agents start fresh every session. No institutional memory.

None of this is new. Every team that has grown a codebase without discipline has hit this. What's new is the speed. Agents are productive and confident even when they're wrong, which means a codebase can get large and inconsistent very quickly.

---

## The Failure Mode in Practice

Imagine you're building a marketplace platform — businesses transact, the platform earns a commission. Different businesses have different rates. Simple enough.

Now the question: **where does the commission rate live?**

Three defensible options:

| Option | Location | Rationale |
|---|---|---|
| A | Platform module | The platform sets and owns rates; it's a revenue concern |
| B | Organization module | Rates are part of the business relationship; orgs have rates |
| C | New billing/contracts module | This is its own domain; neither platform nor org should own it |

A senior engineer could write a convincing case for any of them.

Here's what happens without a decision mechanism:

One developer starts building the commission feature and puts it in the platform module. Another, working on a different feature, needs a commission rate and looks in the organization module — makes sense, it's an org-level setting. It's not there, so they add it. A third agent, writing a billing report, finds rate data in both places with different values and doesn't know which to trust. It picks one. Now you have three implementations, two of which are wrong, and every PR that touches commission rates becomes an architectural argument.

Here's what happens with an interface contract:

```python
# common/interfaces.py — written before any implementation

class OrganizationServiceInterface(Protocol):
    def get_commission_rate(self, org_id: str) -> Decimal: ...
```

Three lines. Now:

- Every agent that needs a commission rate calls `org_service.get_commission_rate(org_id)` — because that's the only place it exists
- Every PR is checked against the interface — code that doesn't use it is wrong by definition
- The architectural debate is closed — not because everyone agreed it was the best choice, but because the interface made the choice structural

The critical insight here: **the interface's value isn't that it found the right answer. It's that it ended the question.** The cost of endless indecision — constant redesign, conflicting implementations, agents guessing — is higher than the cost of a suboptimal-but-consistent decision.

Architecture is full of questions that don't have one correct answer. The dangerous ones aren't the hard questions. They're the reasonable questions that keep getting re-asked, re-litigated, and re-implemented by each person (or agent) who encounters them fresh.
