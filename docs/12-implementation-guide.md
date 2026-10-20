# Implementation Guide

> Adopt the system in stages. Each stage should produce a visible improvement and
> a lower-cost next step; do not pause product work for an unbounded transformation.

## Stage 0: Baseline the system

Before adding rules, measure what exists:

- module and ownership map;
- dependency cycles and shared-kernel growth;
- largest and most connected files;
- CI queue, runtime, flakes, retries, and shard imbalance;
- critical user and money journeys;
- deployment and preview lifecycle;
- data classes without retention or erasure policy.

Save the baseline. It lets later guardrails ratchet improvement instead of failing
every existing violation at once.

## Stage 1: Make ownership visible

- Name domain modules or feature slices.
- Give each one a short specification: owns, does not own, public surface, data,
  dependencies, and focused test command.
- Identify authoritative sources of permissions, prices, availability, and state.
- Add a structural dependency check in warning mode, then block new violations.

Outcome: a fresh contributor can answer where a change belongs.

## Stage 2: Make behavior executable

- Classify jobs: feature, bug, refactor, migration, or mechanical change.
- Require a meaningful failing test for behavior changes.
- Run persistence, migration, and concurrency claims against the production engine.
- Repair shared-state fixtures, nondeterministic clocks/order, and resource leaks.
- Establish a small set of critical golden journeys.

Outcome: failures distinguish product defects from test-infrastructure defects.

## Stage 3: Make feedback fast

- Measure CI before redesigning it.
- Put cheap static and contract checks first.
- Rebalance shards by observed duration.
- Cancel superseded runs.
- Add a fail-closed change classifier and one stable aggregate required check.
- Introduce narrow lanes only for explicitly proven low-risk scopes.
- Reuse exact-tree evidence only when equivalence is automated.

Outcome: fast feedback without weakening high-risk gates.

## Stage 4: Make releases observable

- Bind artifacts and configuration to immutable revisions.
- Separate draft, reviewed, selected, live, superseded, and retired states.
- Verify controller convergence, workload identity, and public behavior.
- Create previews only for named acceptance work.
- Automate and verify teardown of every metered or privileged resource.

Outcome: “deployed” means the intended artifact was observed serving, and temporary
state does not linger indefinitely.

## Stage 5: Make review compound

- Require independent review proportional to risk.
- Batch compatible findings into repair rounds.
- Convert the second occurrence of a defect into a tracked guardrail candidate.
- Add actionable lint, architecture, schema, or CI checks.
- Track suppressions with owner, reason, issue, and age.

Outcome: review effort reduces future review effort instead of repeating itself.

## Stage 6: Modernize incrementally

- Set warning budgets for size, complexity, coupling, shared-kernel growth, and
  slow/flaky tests.
- Prevent regression immediately.
- Decompose one capability or transaction boundary at a time behind characterization
  tests.
- Migrate callers and delete the old path in the same coherent change.
- Lower the baseline after each improvement.
- Consider service extraction only when measured constraints justify distribution.

Outcome: architecture improves continuously without a risky rewrite program.

## A practical first iteration

A small team can usually begin with:

1. One ownership README template.
2. One import-boundary check.
3. One real-engine test fixture.
4. Three to five golden journeys.
5. CI timing and flake telemetry.
6. Superseded-run cancellation.
7. A preview teardown check.

Then use real failures and review repetition to choose the next guardrail.

## What not to do

- Do not start by inventing dozens of agent personas.
- Do not split into services to make the diagram look mature.
- Do not run the full suite twice when exact equivalence is provable.
- Do not introduce a narrow CI lane that defaults unknown changes to “safe.”
- Do not hide flakes with retries or oversized timeouts.
- Do not require every existing file to meet a new perfect threshold immediately.
- Do not treat a successful workflow as proof that the intended runtime is serving.

## The compounding loop

```text
explicit ownership
  → smaller change surface
    → clearer tests and reviews
      → repeated failures become guardrails
        → faster trustworthy feedback
          → safer incremental architecture improvements
            → more explicit ownership
```

The objective is not more process. It is a system in which correctness and delivery
claims are cheap to prove, difficult to fake, and understandable by someone who did
not participate in the original conversation.
