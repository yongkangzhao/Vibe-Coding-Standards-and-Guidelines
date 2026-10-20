---
name: architecture-and-module-boundaries
description: Structure an AI-assisted codebase with explicit domain ownership, enforced dependency direction, predictable module layout, and incremental evolution. Use when starting a project, deciding where behavior belongs, organizing backend modules or frontend feature slices, setting boundary checks, choosing monolith versus services, or decomposing a coupled codebase.
---

# Architecture & Module Boundaries

> Organize the codebase by domain, enforce dependency direction with tooling, and
> evolve boundaries from evidence. A modular monolith is a strong default, not a
> mandate to simulate future microservices inside one process.

## When to use this
- Starting a new project, package, or service and deciding the top-level structure.
- Deciding **where** a new endpoint, business rule, feature, or component belongs.
- **Backend:** organizing domain service modules (e.g. `task`, `user`, `billing`); choosing monolith vs. microservices; wiring cross-module dependencies.
- **Frontend:** organizing feature/domain slices; stopping cross-feature imports; deciding what is a shared primitive vs. feature-owned; keeping business logic out of components.
- Setting up lint/hook rules that block illegal imports or misplaced logic.
- Refactoring a codebase organized by technical layer (`controllers/`, `models/`, `components/`) into one organized by domain.

## The standards

**Use a modular monolith as the default for a new stateful product.** One codebase gives agents full visibility; domain modules give enforced isolation without adding network and operational failure modes. Choose another architecture when measured scale, isolation, residency, availability, or team ownership requires it—not because distribution looks mature. Preserve an existing sound architecture rather than rewriting it to match this default.

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

**Design for local change; keep extraction optional.** Narrow contracts and explicit composition reduce ordinary refactor cost and can make later extraction tractable. They do not require every service to have a ceremonial interface or pretend to be remote.

- **Contract-based dependency:** cross-module consumers use a narrow public capability; introduce substitutable interfaces where multiple implementations, testing seams, or boundary stability justify them.
- **Known composition roots:** concrete construction and wiring are visible. A later
  extraction starts there, while callers and contracts still need review for remote
  latency, failure, and consistency semantics.

```
# composition root — the ONLY place wiring changes
orgService: OrganizationServiceInterface = OrganizationService(db)        # today, in-process
orgService: OrganizationServiceInterface = OrganizationHTTPClient(url)    # tomorrow, remote
```

- **Opaque IDs across boundaries:** pass `org_id` strings, not ORM foreign-key objects, so there's no cross-module persistence coupling to unpick.

**Require evidence before extraction.** Independent scaling, fault/security isolation, residency, availability, or stable team ownership may justify a service. Record the transaction, consistency, latency, auth, observability, deployment, rollback, and operational consequences in an ADR. A typed call site does not make a network boundary free.

**Modernize with ratchets, not rewrites.** Baseline file size, complexity, dependency cycles, fan-in/fan-out, and shared-kernel growth. Prevent new regression immediately, then move one cohesive capability behind characterization tests and lower the baseline after each change.

## Checklist
- [ ] Code is organized by domain module/feature slice, not by technical layer.
- [ ] Every module has the same file layout, and one README states its role **and what it does NOT own**.
- [ ] Cross-module/cross-feature imports go through a public `index`/barrel — never into a sibling's internals.
- [ ] A linter or hook (not a doc) blocks illegal imports and misplaced logic; every exception is a visible `nocheck`/allow-list entry.
- [ ] Cross-module dependencies use narrow public contracts; substitutable implementations are wired in known composition roots.
- [ ] Identifiers crossing module boundaries are opaque (plain IDs), not ORM/foreign-key objects.
- [ ] Architecture budgets prevent new cycles, shared-kernel growth, and complexity regression while legacy debt is reduced incrementally.

## What breaks without this
- **Feature duplication:** the same logic (fetching a user's active subscription, a default locale) gets re-implemented in three files, each slightly different; one gets updated, the others rot.
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
