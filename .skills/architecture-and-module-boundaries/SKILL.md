---
name: architecture-and-module-boundaries
description: Structure an AI-assisted codebase as a modular monolith with domain modules, structurally-enforced boundaries, predictable uniform layout, and day-one extractability. Use this when starting a new project or service, deciding where a new endpoint/feature/component should live, organizing backend domain modules or frontend feature slices, setting up import/lint boundary rules, picking between monolith and microservices, refactoring a "junk drawer" codebase, or whenever an agent is about to guess where code belongs — even if the user doesn't name architecture explicitly.
---

# Architecture & Module Boundaries

> Organize the codebase by domain into modules whose boundaries are enforced by tooling, give every module the same predictable shape, and design each one so it could later be lifted into its own service without a rewrite. This matters more for AI-written code than human-written code: agents start every session with no institutional memory, can't reason about future architecture, and will place code wherever seems locally reasonable — so the structure itself, not a contributing guide, has to be the navigation system and the guardrail.

## When to use this
- Starting a new project, package, or service and deciding the top-level structure.
- Deciding **where** a new endpoint, business rule, feature, or component belongs.
- **Backend:** organizing domain service modules (e.g. `task`, `user`, `billing`); choosing monolith vs. microservices; wiring cross-module dependencies.
- **Frontend:** organizing feature/domain slices; stopping cross-feature imports; deciding what is a shared primitive vs. feature-owned; keeping business logic out of components.
- Setting up lint/hook rules that block illegal imports or misplaced logic.
- Refactoring a codebase organized by technical layer (`controllers/`, `models/`, `components/`) into one organized by domain.

## The standards

**Start from a modular monolith, not microservices.** One codebase gives agents full visibility (they can find existing implementations instead of duplicating them); domain modules give enforced isolation. Microservices split visibility across repos and force agents to reason about network failures before the domain boundaries are even known to be right — and a wrong boundary is expensive to move when it's a service, cheap when it's a folder. Adopt other architectures only when scale actually demands it.

**Organize by domain, not by technical layer.** A feature should live in one module, not be smeared across `models/`, `controllers/`, and `services/` that every other feature also shares. Domain folders give a narrow blast radius per change; layered folders give a wide one. This applies identically on the frontend: feature slices, not a global `components/` + `hooks/` + `utils/` soup.

**Let constraints answer WHERE and HOW, never WHAT.** A rule like "business logic goes in `service`, tables in `orm`, contracts in `interfaces`" doesn't limit what you can build — it ends the perpetual "where does this go?" question so every agent gives the same answer every time. Constraints collapse the agent's search space from "explore everything and guess" to "go straight to the known location."

**Make every module structurally identical.** When one module's layout teaches you all of them, agents navigate by pattern instead of by exploration. Boring and predictable beats clever — clever has cognitive overhead, predictable has none. Sketch:

```
modules/<domain>/
  index            # public exports ONLY — the module's surface
  service          # all business logic
  orm | model      # persistence / data shape
  README           # role, what it does NOT own, public interface
  tests/
```

**Enforce boundaries with tooling, not convention.** A guideline ("don't import across modules") relies on memory and discipline; a guardrail (a linter/hook that rejects the import) works regardless. Agents can't violate what the tool blocks, and the rejection tells them exactly what to do instead. Allow rare, *visible* exceptions only at the composition root:

```
import OtherModuleInternals   # nocheck: arch-guard — composition root only
```

**Expose only a public surface; forbid reaching into internals.** Other modules import the barrel/`index`, never a sibling's `service`/`orm` directly. This is the seam that later becomes a network boundary — keep it clean now and extraction is tractable later.

**Design for extractability from day one.** Three decisions make a module liftable into a service without rewriting its callers — and they're the *same* decisions a microservice needs, so they're early, not premature:

- **Interface-based DI:** every service is consumed through an interface/`Protocol`, never a concrete class.
- **Single composition root:** the one startup location that constructs concretes and injects them. Swapping a local call for a remote client changes only this file.

```
# composition root — the ONLY place wiring changes
orgService: OrganizationServiceInterface = OrganizationService(db)        # today, in-process
orgService: OrganizationServiceInterface = OrganizationHTTPClient(url)    # tomorrow, remote
```

- **Opaque IDs across boundaries:** pass `org_id` strings, not ORM foreign-key objects, so there's no cross-module persistence coupling to unpick.

**Be honest that the interface only stabilizes call sites.** The `Protocol` keeps callers compiling, but crossing a real network boundary still adds work the in-process call never had: timeouts, retries, partial failures, serialization of rich types, loss of a single shared DB transaction, latency, and auth on the new seam. Extractability means you harden one existing seam instead of carving a new one through entangled code — it does not mean extraction is free.

## Checklist
- [ ] Code is organized by domain module/feature slice, not by technical layer.
- [ ] Every module has the same file layout, and one README states its role **and what it does NOT own**.
- [ ] Cross-module/cross-feature imports go through a public `index`/barrel — never into a sibling's internals.
- [ ] A linter or hook (not a doc) blocks illegal imports and misplaced logic; every exception is a visible `nocheck`/allow-list entry.
- [ ] Every service is consumed via an interface; concretes are constructed in exactly one composition root.
- [ ] Identifiers crossing module boundaries are opaque (plain IDs), not ORM/foreign-key objects.

## What breaks without this
- **Feature duplication:** the same logic (fetching a user's active subscription, a commission rate) gets re-implemented in three files, each slightly different; one gets updated, the others rot.
- **Logic leakage:** a discount/pricing rule ends up in the API handler, the frontend component, and a raw query at once — each with different rounding, none agreeing.
- **Architectural drift:** the intended structure lives in a doc or someone's head while the real structure is whatever agents did; they diverge fast because every session starts fresh.
- **Rewrite-to-scale:** without interfaces and opaque IDs, modules grow direct couplings, so the day you need a service you face a rewrite instead of hardening one seam.

## Stack-specific examples
- **Backend:** see `references/backend.md` (5-file module layout, composition root, opaque IDs).
- **Frontend:** see `references/frontend.md` (feature folders, public index barrels, lint-enforced import boundaries, logic out of components).

## Related
- Full rationale: `docs/01-modular-monolith.md`, `docs/02-what-breaks.md`, `docs/03-constraints.md`, `docs/04-structural-guidelines.md`, `docs/07-extractability.md` in this repo.
- The interface-first dev loop (write the contract, then tests, then implementation) belongs to the **interface-first-development** skill — follow it for mechanics; this skill only sets where the interfaces and modules live.
- For up-front domain decomposition before you commit to module boundaries, use **superpowers:brainstorming**.
