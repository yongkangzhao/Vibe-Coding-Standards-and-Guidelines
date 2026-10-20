# Standard 14: Coordinate Agent Work Deliberately

> Parallelism helps only when assignments are independent, ownership is explicit,
> and integration cost is lower than the time saved.

> **Standards**
> - One production-code owner at a time for a bounded change surface.
> - Independent review for meaningful changes; stronger separation for high risk.
> - Parallel agents must have non-overlapping writes or a defined integration owner.
> - Every assignment has a start point, scope, completion evidence, and base revision.
>
> **Guidelines**
> - Use planner, test, implementation, and adversarial lenses when the scope warrants.
> - Reuse a focused worker within one coherent task; start fresh when context becomes
>   unrelated or noisy.
> - Batch compatible findings into one repair round.

## Roles are lenses, not mandatory headcount

A useful review system separates four responsibilities:

| Lens | Primary responsibility | Typical write access |
|---|---|---|
| Planning | ownership, dependencies, journey, risk, acceptance | plan/spec only |
| Test design | RED behavior, coverage dimensions, fixture quality | tests |
| Implementation | production change and focused GREEN checks | scoped code |
| Adversarial review | security, races, misuse, data loss, accessibility | read-only or exploit tests |

These may be different people or agents for a high-risk feature. For a small,
low-risk change, one implementer plus one independent reviewer can cover the same
lenses. The invariant is independence where it matters, not a fixed number of agents.

## Dispatch only meaningful independent work

A good task packet contains:

- outcome and non-goals;
- files or modules to inspect;
- allowed write scope;
- governing contracts and decisions;
- base revision;
- named tests and review evidence;
- where complete logs or artifacts should be stored.

“Improve the app” is not an assignment. “Add the missing rejection behavior to the
documented API contract; touch this module and its tests; done when the regression
test and affected journey pass” is.

Do not delegate a helper method merely to keep an agent busy. Startup, context
transfer, review, and merge conflicts are real costs.

## Parallelism requires ownership

Safe parallel tasks either read shared files or write disjoint surfaces. Examples:

```text
planner:       read-only architecture and journey map
test designer: tests/test_behavior_*.py
implementer:   module production files
adversary:     tests/test_security_*.py
```

If two tasks must edit the same contract or fixture, serialize them or designate one
integration owner. Shared-directory access is not coordination.

## Review iteratively—but do not repeat unchanged work

The useful loop is:

```text
implementer reaches focused GREEN
  → reviewers inspect the changed code and evidence
    → findings are combined into a repair round
      → implementer repairs
        → reviewers recheck changed scope and affected invariants
```

Continue until objective gates are clear. Do not require an identical second review
of an unchanged diff merely to satisfy a round count. Conversely, a changed authority,
money, migration, or shared contract invalidates the relevant prior clearance.

A reviewer who only runs the test suite has not reviewed the code. A reviewer who
changes production code and then approves it has not provided independent review.

## Context is a resource

Long context is not automatically better. Keep a concise checkpoint containing:

- current decision and revision;
- completed evidence;
- remaining acceptance criteria;
- blockers and exact resume conditions;
- links to full logs rather than copied histories.

Reuse a worker when it remains focused on the same coherent problem—the accumulated
local understanding can be valuable. Start a fresh worker for an unrelated batch,
when stale history dominates, or when independence is required. “Kill every worker
after every commit” wastes useful context; “reuse every worker forever” accumulates
noise. Choose based on task continuity and measured effectiveness.

## Completion

Agent activity is not progress by itself. A coordinated batch ends with a coherent
user outcome, exact revision, review decision, verification evidence, and clean
handoff or merge. Count verified outcomes and rework—not messages, agents, or commits.
