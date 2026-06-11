# Roles, Tool Restrictions, and Operational Patterns

The team is five roles. The restrictions are enforced at the **tool level** — an agent
without write tools cannot write, no matter what its prompt says. This is what keeps a
reviewer from fixing-then-approving its own work.

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
- Target: no test file over ~500 lines. A 2,000-line test file is as hard for an agent to
  navigate as a 2,000-line service file.

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

## Kill agents between batches; lead keeps context

Agents accumulate context across tasks; by the fifth task most of it is irrelevant. Stale
context hurts three ways: lower output quality (worse signal-to-noise), higher cost (every stale
token is re-billed on every message), and slower responses (larger contexts process slower).

```
commit batch N
  -> shut down ALL worker agents
  -> spawn fresh workers for batch N+1
     (each gets ONLY: relevant files, failing tests, issue description)
lead persists across batches: what's done, what's next, decisions made
```

Persistent lead + ephemeral workers mirrors a human team: the tech lead carries institutional
knowledge while individual contributors focus on the current task. Throwing away worker context
feels wasteful but the discarded context is mostly noise — a fresh agent with a precise prompt
beats a stale agent with a long history on output, cost, and speed.

## The cycle, end to end

```
1. [Optional] Planner — confirm spec (README) + interface already exist
2. Test Engineer — write failing tests against the interface (TDD red)
3. Engineer — implement until all tests pass (TDD green)
4. Test Engineer — review coverage (READ the code), add tests
5. Adversary — review for flaws (READ the code), write exploit tests
6. IF findings -> Engineer fixes -> back to step 4
7. REPEAT until both reviewers report "all clear"
8. Commit, then kill workers and start the next batch fresh
```

Reviewing means reading the changed code every cycle — including re-reviews after fixes. Green
tests are verification, not review: tests can be wrong, and new code can introduce issues no
existing test covers.
