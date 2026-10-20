# Standard 1: Use a Modular Monolith as the Default

> AI agents need two things that most architectures don't provide simultaneously: full visibility into the codebase, and enforced boundaries that prevent them from putting code wherever seems locally reasonable. Without full visibility, agents can't find existing implementations. Without enforced boundaries, agents duplicate logic, leak business rules across layers, and make architectural decisions that contradict ones already made in other parts of the codebase.

> **Standards** (must follow):
> - Use a modular monolith as the default for a new stateful product unless a
>   demonstrated constraint requires a different deployment architecture
> - Enforce module boundaries structurally (not just by convention)
> - Maintain a single codebase so agents have full visibility
>
> **Guidelines** (recommended):
> - Specific module layout and file naming can vary by stack
> - Evaluate other architectures when measured scale, isolation, residency,
>   availability, or organizational ownership demands it

There are many ways to structure a software project. Microservices, serverless, classic monolith, MVC, event-driven — each has legitimate use cases. But for a vibe-coded project specifically, the choice matters more than usual because AI agents interact with codebases differently than humans do: no accumulated context, bounded sessions, no institutional memory. They need explicit structure.

Here's how the common options compare:

| Architecture | Agent Visibility | Domain Isolation | Early Complexity | Testable in Isolation | Path to Scale |
|---|---|---|---|---|---|
| **Classic Monolith** | Full | None — everything leaks | Low | Hard | Rewrite required |
| **Modular Monolith** ✓ | Full | Enforced — module boundaries | Low-Medium | Yes — per module | Extract modules to services |
| **Microservices** | Partial — split across services/repos | Strong — network boundary | Very High | Yes — per service | Already there |
| **Serverless** | Fragmented — logic scattered across functions | Minimal | Medium | Hard — mocking required | Vendor-dependent |
| **MVC / Layered** | Full | Weak — by technical type, not domain | Low | Hard — layers coupled | Rewrite required |

**Classic monolith**: no ceremony, fast to start, but no isolation. Agents put things wherever seems locally reasonable. Two agents, two different answers about where something belongs. Logic leaks everywhere.

**Microservices**: strong domain isolation, but far too much overhead for early stage. Agents have to reason about network failures, distributed transactions, service discovery, and multiple repos at once. Worse, the domain boundaries you define in week one are usually wrong — and moving a microservice boundary is expensive. Moving a module boundary is just moving files.

**Serverless**: fine for specific use cases (background jobs, webhooks). As a primary architecture, too granular. No module concept to anchor around. Agents lose the big picture across hundreds of functions.

**MVC/Layered**: organizes by technical type (models, controllers, services) not by domain. A user management feature touches the models layer, the controllers layer, and the services layer — all of which are shared with every other feature. No isolation, wide blast radius for any agent change.

**Modular monolith**: single codebase (full agent visibility), enforced domain boundaries (structural, not conventional), low early complexity, per-module testability, and a path to independent deployment if evidence later justifies it. Modular monoliths and services share several useful decisions—explicit ownership, contracts, dependency direction—but a service also adds network, consistency, deployment, and operational responsibilities.

This is a default, not a universal law. Existing distributed systems should not be
collapsed merely to follow a guide, and naturally isolated workloads may justify a
different shape. The decision should be written down in terms of measured constraints,
not fashion or anticipated scale.

When using the default, start here. The rest of the guide describes how to make the
boundaries real enough that a later extraction is possible without designing every
module as a pretend network service today.
