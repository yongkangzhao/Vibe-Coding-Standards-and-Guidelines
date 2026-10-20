# Standard 10: Spec → Contract → Tests → Implementation → Proof

> Code generation should begin only after the change has a home, a contract, and
> an observable definition of correctness.

> **Standards**
> - Classify the job and its risk before choosing the workflow.
> - Define ownership and externally visible behavior before implementation.
> - For a behavior change, demonstrate a meaningful RED state before GREEN.
> - Finish with evidence through the real integration path appropriate to the risk.
>
> **Guidelines**
> - A module README is a strong specification format, but not the only one.
> - Contracts may be typed interfaces, API schemas, events, or state machines.
> - Low-risk documentation or mechanical refactors need proportionate evidence,
>   not artificial tests.

## 1. Classify the job

Different jobs start at different places:

| Job | First evidence |
|---|---|
| New feature or capability | ownership map, journey, contract, failing behavior test |
| Bug fix | failing reproduction of the reported behavior |
| Refactor with no behavior change | characterization tests of current behavior |
| Migration | old/new schema and rollback or forward-recovery contract |
| Documentation or mechanical change | rendered/structural validation and affected guardrails |

Do not force every edit through identical ceremony. Do force every behavioral claim
to have evidence capable of disproving it.

## 2. Specify ownership and the journey

Before implementation, write down:

- the outcome and actors;
- the ordered user or system journey;
- what the affected module owns and does not own;
- the authoritative source of state;
- the failure and recovery behavior;
- privacy, security, accessibility, and operational constraints;
- the evidence required before merge and after deployment.

This step prevents “locally reasonable” work from inventing a second scheduler,
ledger, permission model, or source of truth.

## 3. Define the contract

Express the boundary without exposing implementation details. Depending on the
system, this may be a protocol, API schema, event, command, or state transition:

```python
class NotificationPort(Protocol):
    def notify_assignment(self, entity_id: str, recipient_id: str) -> None: ...
```

Prefer module-owned contracts. A central contract package is appropriate for a
small shared kernel, but it should not become a global junk drawer.

Contracts make parallel work possible and give tests a stable surface. They do not
eliminate design changes; when evidence shows the contract is wrong, revise the
specification and all affected callers deliberately.

## 4. Establish RED

Write the smallest test that demonstrates the missing or broken behavior and run it.
The failure must be for the intended reason—not a syntax error, missing fixture, or
unrelated environment failure.

For a refactor, RED may be a deliberately introduced mutation proving the
characterization test can fail. For a pure documentation change, RED is usually not
meaningful; use a link, render, schema, or generation check instead.

## 5. Implement and refactor

Make the focused test pass with the smallest coherent change. While editing:

- run cheap, relevant checks first;
- preserve module boundaries and transaction ownership;
- delete replaced paths instead of hiding a migration behind dual behavior;
- update specification and contract when implementation evidence changes them;
- keep the diff reviewable as one user outcome.

GREEN is not the finish line. It means the local claim now holds.

## 6. Review according to risk

Every meaningful change needs an independent view of the diff. Sensitive surfaces—
money, identity, authorization, privacy, migrations, destructive actions, and
shared infrastructure—need additional lenses and executable adversarial evidence.

Batch reviewer findings into a repair round where practical. Re-review the changed
scope and affected invariants; do not rerun an identical review merely for ceremony.

## 7. Prove the real path

Unit tests certify local behavior, not wiring. Finish with the narrowest real-path
evidence that can prove the outcome:

- an in-process request through the real router and production-shaped auth;
- a real database engine for persistence and concurrency semantics;
- a golden journey across module boundaries;
- a genuine UI route with honest loading, empty, error, and success states;
- an immutable artifact and a post-deploy probe bound to its revision.

For UI work, capture visual evidence only after page-specific content is present.
For deployment work, “the workflow ran” is not proof; observe the intended artifact
serving and the environment converged.

## The loop

```text
classify
  → specify ownership and journey
    → define contract
      → prove RED
        → implement GREEN
          → independent review
            → prove the real path
              → merge and observe rollout
```

The human role is to own intent, risk, and acceptance. Agents can contribute to
design and implementation, but important decisions must be represented in durable
artifacts rather than left in one conversation.
