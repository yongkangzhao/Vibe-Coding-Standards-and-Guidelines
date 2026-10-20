# Standard 13: Design for Change Without Premature Distribution

> Good module boundaries make change local. They may also make later service
> extraction easier, but extraction is an option—not the goal of every module.

> **Standards**
> - Each domain has explicit ownership and dependency direction.
> - Cross-module calls use a narrow public contract, not another module’s internals.
> - Construction and wiring happen in known composition roots.
> - Service extraction requires measured operational or organizational need.
>
> **Guidelines**
> - Use dependency injection where substitution or isolation has real value.
> - Prefer identifiers and contracts that do not expose persistence internals.
> - Keep in-process calls in-process until distribution solves a demonstrated problem.

## Evolutionary architecture

The purpose of a modular monolith is not to simulate microservices inside one
process. It is to make ownership and change boundaries explicit while preserving
simple transactions, debugging, deployment, and local development.

A well-shaped module can evolve in several ways:

- split an oversized implementation into internal components;
- move a reusable policy into a deliberately small shared kernel;
- publish an event to decouple a non-transactional reaction;
- extract a workload that needs independent scale or isolation;
- remain an in-process module indefinitely.

All are valid outcomes.

## What makes change tractable

1. **Narrow public surface.** Callers depend on capabilities, not ORM objects or
   internal helpers.
2. **Explicit composition.** Concrete implementations are selected in a known
   wiring location rather than instantiated throughout the codebase.
3. **Owned data.** One module defines the meaning and mutation rules for its state.
4. **Characterization tests.** Existing behavior is executable before internals move.
5. **Observable dependencies.** Architecture checks can show cycles, fan-in, and
   growth in shared surfaces.

These properties improve ordinary refactoring even if no service is ever extracted.

## When extraction is justified

Consider an independent service when evidence shows at least one durable need:

- independent scaling characteristics;
- fault or security isolation;
- regulatory or data-residency boundaries;
- materially different availability requirements;
- independent release ownership by a stable team;
- a workload with a natural asynchronous boundary.

Document the decision in an ADR. Include transaction changes, consistency model,
latency budget, retries and idempotency, authentication, observability, deployment,
rollback, data migration, and operational ownership.

A typed in-process contract reduces call-site churn, but it does not make a remote
call equivalent. Networks add partial failure, serialization, latency, authentication,
and the loss of one database transaction. Those costs must be designed explicitly.

## Modernizing large modules

Do not answer a large file with a large rewrite. Use a ratchet:

1. Record current size, complexity, dependencies, and slow tests.
2. Add characterization tests around one cohesive responsibility.
3. Move that responsibility behind a reviewed internal contract.
4. Migrate all callers and delete the old path in the same change.
5. Lower the recorded budget so the old shape cannot return.

This turns architecture improvement into a sequence of reviewable outcomes rather
than a risky, open-ended cleanup project.
