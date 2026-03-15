# Standard 1: Start With a Modular Monolith

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

**Modular monolith**: single codebase (full agent visibility), enforced domain boundaries (structural, not conventional), low early complexity, per-module testability, and a clear path to microservices when scale actually demands it. The modular monolith and microservices architecture share the same underlying decisions — interface contracts, dependency injection, domain ownership. The monolith is just the right first deployment target.

Start here. The rest of this article describes how to build one that holds up.
