---
name: interface-first-development
description: Defines module specs (a README that says what the module owns AND does NOT own) and typed contracts before any implementation, then drives the Spec -> Interface -> Tests -> Implementation loop. Use this when starting a new module, service, feature, component, or API client; when deciding "where does this logic live?"; when two pieces of code need a commission-rate / shared-value boundary; when writing or reviewing a module/feature README; or when defining backend service interfaces (Protocols, interfaces.py) or frontend contracts (component props/events, typed API clients) — even if the user never says "interface-first" or "contract."
---

# Interface-First Development

> Write the contract before the code: a spec that states what a module owns and explicitly does NOT own, a typed interface that locks scope, then tests, then implementation — in that order. This matters most when AI agents write the code, because an agent starts every session fresh with no institutional memory. Without a spec and interface already in place, it designs on the fly from local context and re-litigates decisions the team already made — fast, confidently, and three times over.

## When to use this
- Starting a new backend module or service — before writing `service.py`, write its README and its `interfaces.py` entry.
- Starting a new frontend feature or component — before writing the component, define its prop/event contract, its typed API-client interface, and a feature README.
- Someone asks "where should this logic live?" or the same value (a commission rate, a discount, a status) is about to be computed in two places.
- Reviewing a module/feature README, or a PR that adds public methods, props, or endpoints.
- Coordinating parallel work — two agents or two people need to build against each other without direct coordination.

## The standards

**Write the README as an executable spec, owns AND does-NOT-own.** For agents, documentation IS the specification — they read it instead of carrying months of context. The does-NOT-own section is what stops an agent from optimizing locally and dropping logic into the nearest plausible module.
```
# <module> README
Role:            one paragraph — what this module owns
Does NOT own:    explicit non-ownership (e.g. "pricing lives in billing, not here")
Public interface: every method/prop/endpoint — what it does, what it raises/returns
Data:            tables or state it stores
Dependencies:    what other services/contracts it requires
Test command:    how to run this unit's tests in isolation
```

**Define the interface before any implementation exists.** The contract is the AI-to-AI communication protocol: if a method isn't in the interface, it isn't part of this unit — no ambiguity, nothing to invent mid-build.
```
interface OrganizationService:
    get_commission_rate(org_id) -> Decimal      # the ONLY place this value lives
    # backend: a Protocol/interface in a shared contracts file
    # frontend: a typed component prop/event contract + a typed API-client interface
```

**Let the contract end the question, not find the perfect answer.** The commission-rate value comes from exactly one method, so every caller uses it — closing an architectural debate that would otherwise be re-asked and re-implemented by each fresh agent.
```
# without a contract: rate lives in 3 modules, 2 wrong, every PR an argument
# with a contract:    callers do org_service.get_commission_rate(org_id) — done
```

**Follow the loop in order: Spec -> Interface -> Tests -> Implementation.** By the time implementation begins, three layers of constraints already define "correct," so the agent's only job is to satisfy them — there is no design work left to improvise.
```
1. Spec          README: owns / does NOT own
2. Interface     signatures in the shared contracts file — before service code
3. Tests         written against the interface; MUST fail first (stubs raise NotImplemented)
4. Implementation make the failing tests pass, within the constraints already set
```

**Build against interfaces, not implementations, to unblock parallel work.** Agent A builds the task module and Agent B builds notification, each against the same contracts file; they never coordinate directly and still compose at the wiring root.

## Checklist
- [ ] README exists and has both a "Role / owns" and an explicit "does NOT own" section.
- [ ] Every public method/prop/endpoint appears in the contract before its implementation.
- [ ] Each shared value (rate, price, status) is owned by exactly ONE interface method.
- [ ] Tests were written against the interface and observed FAILING before implementation began.
- [ ] No design decisions were made during implementation — only build work to pass tests.
- [ ] Callers depend on the interface/contract, not on a concrete class or component internals.

## What breaks without this
- **Feature duplication:** the same logic (fetch active subscription, compute a rate) lands in three files written at different times; billing changes, one gets updated.
- **The re-litigated decision:** where the commission rate lives gets re-asked by every fresh agent; you end up with three implementations, two wrong, and every PR becomes an architectural argument.
- **Design-on-the-fly drift:** with no spec or interface in place, an agent invents architecture mid-implementation that reflects local context, not the global decisions already made.
- **Logic leakage:** without a does-NOT-own boundary, business rules bleed into handlers, components, and queries — each with subtly different behavior, none authoritative.

## Stack-specific examples
- **Backend:** see `references/backend.md` — `interfaces.py` Protocols, a module README template, a failing test stub raising `NotImplementedError`.
- **Frontend:** see `references/frontend.md` — TypeScript interfaces/types, a typed API client, component prop contracts, a feature README template.

## Related
- Full rationale: `docs/04-structural-guidelines.md` (Standards 4 and 5), `docs/05-development-loop.md` (Standard 10), and `docs/02-what-breaks.md` (the commission-rate example) in this repo.
- `tests-as-ground-truth` and `superpowers:test-driven-development` for the red-green test mechanics of step 3 — write the tests there; this skill only requires that they exist and fail first.
