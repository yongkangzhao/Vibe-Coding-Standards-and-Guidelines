# Standard 15: Layered, Deterministic Test Infrastructure

> Tests are useful only when they can fail for the behavior they claim to prove and
> pass repeatedly without depending on hidden shared state.

> **Standards**
> - Behavioral tests are deterministic and isolated.
> - Engine-specific persistence claims run against the production database engine.
> - Critical cross-module flows have a small set of stable golden journeys.
> - Flakes and leaked resources are defects, not a reason to normalize retries.
>
> **Guidelines**
> - Keep most feedback fast and local; reserve expensive infrastructure for claims
>   that require it.
> - Split tests by cohesive behavior before files become navigation bottlenecks.

## Match the layer to the claim

| Layer | Proves | Appropriate substitutes |
|---|---|---|
| Unit/property | local rules, transformations, invariants | clocks, randomness, declared neighboring interfaces |
| Module integration | persistence, transactions, constraints, public module API | external providers at explicit ports |
| Contract | compatibility between producer and consumer | recorded provider sandbox where appropriate |
| Golden journey | cross-module wiring and critical user outcome | as few seams as practical |
| Live smoke | artifact, routing, configuration, and basic runtime health | none for the path being observed |

Do not duplicate every assertion at every layer. Put detailed combinatorics at the
lowest layer capable of proving them, then keep higher layers focused on wiring,
authority, persistence, and externally visible outcomes.

## Use the real engine where semantics matter

An in-memory substitute cannot prove row locking, native types, partial indexes,
constraint timing, isolation, or migration behavior. Module integration, migration,
and concurrency tests that depend on those semantics must run against a disposable
instance of the production engine.

This does not mean every pure function test should start a container. Fast unit tests
remain valuable. The rule is that the infrastructure must be capable of supporting
the claim being made.

Never use a live production database for tests.

## Fixture ownership

Every fixture owns what it creates and removes only that state:

- use unique test-owned identifiers;
- do not depend on execution order;
- do not truncate shared tables from an ordinary test;
- declare when committed visibility is required;
- close sessions, tasks, files, sockets, and containers within bounded cleanup;
- inject or freeze time, randomness, locale, and provider responses when relevant.

A teardown connection leak or timing race is not “just CI.” It means the suite cannot
reliably distinguish a source failure from its own lifecycle failure.

## Golden journeys

Maintain a small, named set of production-equivalent journeys for the most valuable
or dangerous outcomes: onboarding and offboarding, permissions, a representative
purchase and recovery path, a destructive action, and other domain-critical flows.

A golden journey seeds only the minimum external identity and uses real application
actions to create later preconditions. It should not manufacture internal database
state that ordinary users could never create.

Golden journeys are intentionally few. Their purpose is confidence in integration,
not exhaustive business-rule coverage.

## Handling failures

On the first failure:

1. Preserve the log and exact revision.
2. Classify source defect, deterministic infrastructure failure, or unresolved flake.
3. Reproduce the demonstrated condition with the narrowest relevant command.
4. Fix the cause—not the symptom—with a regression test.
5. Rerun invalidated checks, then the required final lane.

Arbitrary sleeps, ever-larger timeouts, order-dependent tests, and blind automatic
reruns hide uncertainty. A retry can help diagnosis; it is not proof of reliability.

## Test organization

Mirror module ownership and name files by behavior. File-size thresholds are
diagnostic budgets, not universal laws: start with a warning based on local history,
split by domain when navigation or fixture coupling degrades, and ratchet the budget
down as large legacy files are decomposed.
