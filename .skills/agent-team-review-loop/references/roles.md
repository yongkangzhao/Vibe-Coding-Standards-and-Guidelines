# Roles, Tool Restrictions, and Operational Patterns

The table describes five useful lenses, not mandatory headcount. Combine lenses for
small, low-risk changes; separate them for sensitive work. Where supported, enforce
reviewer independence at the tool level.

## Role × tools table

| Role | Write production code? | Write tests? | Read code? | Purpose |
|------|------------------------|--------------|------------|---------|
| **Team Lead** (human or coordinator) | No | No | Yes | Routes work, batches feedback, decides when to commit, carries high-level context |
| **Planner** | No | No | Yes | Reads code, produces implementation plans; confirms spec (README) + interface exist. Consulted on demand |
| **Test Engineer** | No | Yes (`test_service_*`) | Yes | Writes failing tests BEFORE implementation (TDD red), reviews coverage after green |
| **Engineer** | Yes | Yes | Yes | Makes tests pass, fixes review findings, iterates until clean. The only writer of production code |
| **Adversary** | No | Yes (`test_security_*`, exploit only) | Yes | Thinks like an attacker; writes exploit tests that assert the *secure* outcome |

Key constraints, each enforced by absent tools rather than instructions:
- The **planner** has no write tools — it cannot modify code while analyzing it.
- The **adversary** cannot write production code — only exploit tests that prove a flaw.
- The **engineer** is the single source of production-code writes — no parallel writer can clobber it.

This maps onto the 7-step workflow in `~/.claude/CLAUDE.md` (Planner → Test-engineer → Engineer
→ Test-engineer → Adversary → Engineer fixes → README). Same roles, same tool discipline.

## Exploit tests assert the secure outcome

An exploit test is written like any correctness test — it asserts what *should* happen, so it
fails against vulnerable code and passes once the hole is closed. It never asserts the
vulnerability as permanent behavior.

```
# adversary's test_security_<name>.py
test_non_owner_cannot_delete:
    create resource owned by user_A
    expect PermissionError when user_B deletes it   # fails today, passes once authz is added

test_double_spend_rejected:
    account balance = 100
    spend(100); expect spend(100) to be rejected     # proves no negative-balance race
```

Backend exploit foci: authorization (owner vs non-owner), transaction integrity / double-spend,
zero-sum violations in financial math, bootstrap/privilege-escalation paths, race conditions.
Frontend exploit foci: business logic leaking into the presentation layer (validation, pricing,
authz decided client-side), and accessibility regressions — both are review failures, not just
nits.

## File-ownership naming convention

Two agents writing the same file means one silently overwrites the other. Ownership is encoded
in the filename so it's visible at a glance and safe to run in parallel.

```
modules/<name>/tests/
├── conftest.py                 # shared fixtures (set up once, owned by test-engineer)
├── test_service_<domain>.py    # test-engineer's correctness tests
└── test_security_<name>.py     # adversary's exploit tests (one per module)
```

- `grep -r test_security_` finds every exploit test in the codebase.
- Split `test_service_*` further by domain when a module is large —
  `test_service_auth.py`, `test_service_crud.py`, `test_service_payments.py`.
- Use a repository-specific size/coupling budget. Treat the current baseline as a
  ratchet and split by cohesive behavior when navigation or fixture coupling degrades.

## The "wait for all reviewers, then batch" pattern

When the test-engineer finishes first and finds an issue, the temptation is to send the fix to
the engineer immediately. Don't.

```
test-engineer done -> findings buffered (NOT sent yet)
adversary done      -> findings buffered
lead merges both    -> single message to engineer
engineer            -> one context load, one fix round, one test run
```

Piecemeal fixes force the engineer into multiple rounds — each re-reads context, edits, reruns
tests. Batching means one round. Exception: one reviewer is clearly going to take much longer
and the other's fix is trivial and independent. Default to batching.

## Reuse focused context; replace stale context

Agents accumulate context across tasks. Reuse is valuable while the worker remains on
the same coherent problem; unrelated history eventually lowers signal, increases cost,
and slows review.

```
complete coherent batch N
  -> keep a compact lead checkpoint: revision, decisions, evidence, remaining scope
  -> reuse workers continuing the same problem
  -> start fresh workers for unrelated scope, stale context, or independent review
```

Do not copy the full conversation into every assignment. A compact task packet contains
the outcome, relevant paths, contracts, base revision, constraints, and evidence paths.

## The cycle, end to end

```
1. [Optional] Planner — confirm spec (README) + interface already exist
2. Test Engineer — write failing tests against the interface (TDD red)
3. Engineer — implement until all tests pass (TDD green)
4. Test Engineer — review coverage (READ the code), add tests
5. Adversary — review for flaws (READ the code), write exploit tests
6. IF findings -> Engineer fixes -> back to step 4
7. REPEAT when findings changed the diff; recheck changed scope and affected invariants
8. Commit when required reviewers report "all clear"; retain or replace workers based on continuity
```

Reviewing means reading the changed code every cycle — including re-reviews after fixes. Green
tests are verification, not review: tests can be wrong, and new code can introduce issues no
existing test covers.
